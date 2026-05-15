---
title: Optimizing an MP3 Codec with OCaml/OxCaml
description: "After reading Anil\u2019s post about his zero-allocation HTTP parser
  httpz, I decided to apply some OxCaml optimisation techniques to my pure OCaml MP3
  encoder/decoder."
url: https://www.tunbury.org/2026/02/11/ocaml-mp3/
date: 2026-02-11T18:30:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>After reading Anil’s post about his zero-allocation HTTP parser <a href="https://anil.recoil.org/notes/oxcaml-httpz">httpz</a>, I decided to apply some OxCaml optimisation techniques to my pure OCaml MP3 encoder/decoder.</p>

<p>The <a href="https://github.com/mtelvers/ocaml-mp3">OCaml-based MP3 encoder/decoder</a> has been the most ambitious project I’ve tried in Opus 4.5. It was a struggle to get it over the line, and I even needed to read large chunks of the ISO standard and get to grips with some of the maths and help the AI troubleshoot.</p>

<h1>Profiling an OCaml MP3 Decoder with Landmarks</h1>

<p>Before dividing into OxCaml, I wanted to get a feel for the current performance and also to make obvious non-OxCaml performance improvements; otherwise, I would be comparing an optimised OxCaml version with an underperforming OCaml version.</p>

<p>It was 40 times slower than <code class="language-plaintext highlighter-rouge">ffmpeg</code>: 29.5 seconds to decode a 3-minute file versus 0.74 seconds. I used the <a href="https://github.com/LexiFi/landmarks">landmarks</a> profiling library to identify and fix the bottlenecks, bringing decode time down to 3.5 seconds (a 8x speedup).</p>

<h2>Setting Up Landmarks</h2>

<p>Landmarks is an OCaml profiling library that instruments functions and reports cycle counts. It was easy to add to the project (*) with a simple edit of the <code class="language-plaintext highlighter-rouge">dune</code> file:</p>

<pre><code class="language-sexp">(libraries ... landmarks)
(preprocess (pps landmarks-ppx --auto))
</code></pre>

<p>The <code class="language-plaintext highlighter-rouge">--auto</code> flag automatically instruments every top-level function — no manual annotation needed. Running the decoder with <code class="language-plaintext highlighter-rouge">OCAML_LANDMARKS=on</code> prints a call tree with cycle counts and percentages.</p>

<blockquote>
  <p>(*) It needed OCaml 5.3.0 for <code class="language-plaintext highlighter-rouge">landmarks-ppx</code> compatibility; it wouldn’t install on OCaml 5.4.0 due to a ppxlib version constraint.</p>
</blockquote>

<h2>Issues</h2>

<p>78% of the time was spent in the Huffman decoding, specifically <code class="language-plaintext highlighter-rouge">decode_pair</code>. The implementation read one bit at a time, then scanned the table for a matching Huffman code. I initially tried a Hashtbl, which was much better than the scan before deciding to use array lookup instead.</p>

<p>The bitstream operations still accounted for much of the time, but these could be optimised with appropriate <code class="language-plaintext highlighter-rouge">Bytes.get_...</code> calls, as the most frequent path is reading 32 bits in big endian layout.</p>

<p>The profile now showed <code class="language-plaintext highlighter-rouge">find_sfb_long</code> consuming 3.4 billion cycles inside requantization. This function does a linear search through scalefactor band boundaries for every one of the 576 frequency lines, every granule, every frame. Switching to precomputed 576-entry arrays mapping each frequency line directly to its scalefactor band index.</p>

<p>There were some additional tweaks, such as adding more precomputed lookup tables stored in <code class="language-plaintext highlighter-rouge">floatarray</code>, using <code class="language-plaintext highlighter-rouge">[@inline]</code> and <code class="language-plaintext highlighter-rouge">unsafe_get</code>, <code class="language-plaintext highlighter-rouge">land</code> instead of <code class="language-plaintext highlighter-rouge">mod</code>.</p>

<p>After this, no single function dominated the profile, and I could move on to OxCaml.</p>

<h1>OxCaml</h1>

<p>OxCaml has <code class="language-plaintext highlighter-rouge">float#</code>, an unboxed float type that lives in registers, and <code class="language-plaintext highlighter-rouge">let mutable</code> for stack-allocated mutable variables. Together, they let you write inner loops where the accumulator never touches the heap:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">module</span> <span class="nc">F</span> <span class="o">=</span> <span class="nn">Stdlib_upstream_compatible</span><span class="p">.</span><span class="nc">Float_u</span>

<span class="k">let</span><span class="p">[</span><span class="o">@</span><span class="n">inline</span><span class="p">]</span> <span class="n">imdct_long</span> <span class="n">input</span> <span class="o">=</span>
  <span class="k">for</span> <span class="n">i</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">to</span> <span class="mi">35</span> <span class="k">do</span>
    <span class="k">let</span> <span class="k">mutable</span> <span class="n">sum</span> <span class="o">:</span> <span class="kt">float</span><span class="o">#</span> <span class="o">=</span> <span class="nn">F</span><span class="p">.</span><span class="n">of_float</span> <span class="mi">0</span><span class="o">.</span><span class="mi">0</span> <span class="k">in</span>
    <span class="k">for</span> <span class="n">k</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">to</span> <span class="mi">17</span> <span class="k">do</span>
      <span class="k">let</span> <span class="n">cos_val</span> <span class="o">=</span> <span class="nn">F</span><span class="p">.</span><span class="n">of_float</span> <span class="p">(</span><span class="nn">Float</span><span class="p">.</span><span class="nn">Array</span><span class="p">.</span><span class="n">unsafe_get</span> <span class="n">cos_table</span> <span class="p">(</span><span class="n">i</span> <span class="o">*</span> <span class="mi">18</span> <span class="o">+</span> <span class="n">k</span><span class="p">))</span> <span class="k">in</span>
      <span class="k">let</span> <span class="n">inp_val</span> <span class="o">=</span> <span class="nn">F</span><span class="p">.</span><span class="n">of_float</span> <span class="p">(</span><span class="nn">Array</span><span class="p">.</span><span class="n">unsafe_get</span> <span class="n">input</span> <span class="n">k</span><span class="p">)</span> <span class="k">in</span>
      <span class="n">sum</span> <span class="o">&lt;-</span> <span class="nn">F</span><span class="p">.</span><span class="n">add</span> <span class="n">sum</span> <span class="p">(</span><span class="nn">F</span><span class="p">.</span><span class="n">mul</span> <span class="n">inp_val</span> <span class="n">cos_val</span><span class="p">)</span>
    <span class="k">done</span><span class="p">;</span>
    <span class="nn">Array</span><span class="p">.</span><span class="n">unsafe_set</span> <span class="n">output</span> <span class="n">i</span> <span class="p">(</span><span class="nn">F</span><span class="p">.</span><span class="n">to_float</span> <span class="n">sum</span><span class="p">)</span>
  <span class="k">done</span>
</code></pre></div></div>

<p>These kinds of optimisations got me from 2.35s down to 2.01s.</p>

<p>What I felt was missing was an accessor function which returned an unboxed float from a floatarray, so I wouldn’t need to unbox with <code class="language-plaintext highlighter-rouge">F.of_float</code>. However, I couldn’t find it.</p>

<p>The httpz parser really benefited from OxCaml’s unboxed types because its hot path operates on small unboxed records that stay entirely in registers:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">#</span><span class="p">{</span> <span class="n">off</span><span class="o">:</span> <span class="n">int16</span><span class="o">#;</span> <span class="n">len</span><span class="o">:</span> <span class="n">int16</span><span class="o">#</span> <span class="p">}</span>
</code></pre></div></div>

<h1>Results</h1>

<p>The optimisations brought a 29.5s MP3 decoder down to 2.01s. Mostly through standard OCaml optimisations, but OxCaml’s <code class="language-plaintext highlighter-rouge">float#</code> saved another ~14%.</p>

<table>
  <thead>
    <tr>
      <th>Decoder</th>
      <th>Time</th>
      <th>vs ffmpeg</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ffmpeg</td>
      <td>0.74s</td>
      <td>1x</td>
    </tr>
    <tr>
      <td>LAME</td>
      <td>0.81s</td>
      <td>1.1x</td>
    </tr>
    <tr>
      <td>ocaml-mp3 (original)</td>
      <td>29.5s</td>
      <td>40x</td>
    </tr>
    <tr>
      <td>ocaml-mp3 (Hashtbl)</td>
      <td>6.4s</td>
      <td>8.6x</td>
    </tr>
    <tr>
      <td>ocaml-mp3 (flat + fast bitstream)</td>
      <td>3.5s</td>
      <td>4.7x</td>
    </tr>
    <tr>
      <td>ocaml-mp3 (best)</td>
      <td>2.4s</td>
      <td>3.2x</td>
    </tr>
    <tr>
      <td>ocaml-mp3 (OxCaml)</td>
      <td>2.0s</td>
      <td>2.7x</td>
    </tr>
  </tbody>
</table>
