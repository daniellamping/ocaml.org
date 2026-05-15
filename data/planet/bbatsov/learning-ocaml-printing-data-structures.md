---
title: 'Learning OCaml: Printing Data Structures'
description: "If there\u2019s one thing that frustrated me early on in my OCaml journey,
  it was printing stuff. In Ruby I can p anything and get a useful representation.
  In Clojure, prn just works on every data structure. In OCaml? There\u2019s no generic
  print that works on any type \u2013 the type information is erased at runtime, so
  the language simply doesn\u2019t know how to stringify an arbitrary value."
url: https://batsov.com/articles/2026/03/01/printing-data-in-ocaml/
date: 2026-03-01T16:00:00-00:00
preview_image: https://batsov.com/assets/img/og-image.png
authors:
- Bozhidar Batsov
source:
ignore:
---

<p>If there’s one thing that frustrated me early on in my OCaml journey, it was
printing stuff. In Ruby I can <code class="language-plaintext highlighter-rouge">p</code> anything and get a useful representation.
In Clojure, <code class="language-plaintext highlighter-rouge">prn</code> just works on every data structure. In OCaml? There’s no
generic <code class="language-plaintext highlighter-rouge">print</code> that works on any type – the type information is erased at
runtime, so the language simply doesn’t know how to stringify an arbitrary
value.</p>

<p>This means you need to be explicit about how to print every type you define.
That sounds tedious (and it can be), but the community has developed tools
that make it mostly painless. Let me walk you through the common approaches.</p>



<h2>Printing Built-in Types</h2>

<p>For basic types, OCaml provides dedicated print functions in <code class="language-plaintext highlighter-rouge">Stdlib</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
</pre></td><td class="rouge-code"><pre><span class="n">print_int</span> <span class="mi">42</span><span class="p">;;</span>
<span class="n">print_float</span> <span class="mi">3</span><span class="o">.</span><span class="mi">14</span><span class="p">;;</span>
<span class="n">print_string</span> <span class="s2">"hello"</span><span class="p">;;</span>
<span class="n">print_endline</span> <span class="s2">"hello with newline"</span><span class="p">;;</span>
<span class="n">print_char</span> <span class="k">'</span><span class="n">x'</span><span class="p">;;</span>
<span class="n">print_newline</span> <span class="bp">()</span><span class="p">;;</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>There are also simple conversion functions like <code class="language-plaintext highlighter-rouge">string_of_int</code> and
<code class="language-plaintext highlighter-rouge">string_of_float</code> when you just need a string:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">s</span> <span class="o">=</span> <span class="s2">"Age: "</span> <span class="o">^</span> <span class="n">string_of_int</span> <span class="mi">42</span><span class="p">;;</span>
<span class="c">(* "Age: 42" *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>And of course there’s <code class="language-plaintext highlighter-rouge">Printf.printf</code> for formatted output:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
</pre></td><td class="rouge-code"><pre><span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"Name: %s, Age: %d, Score: %.2f</span><span class="se">\n</span><span class="s2">"</span> <span class="s2">"Alice"</span> <span class="mi">30</span> <span class="mi">95</span><span class="o">.</span><span class="mi">5</span><span class="p">;;</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<blockquote class="prompt-tip">
  <p>A quick cheat sheet for the format specifiers you’ll use most often: <code class="language-plaintext highlighter-rouge">%s</code>
for strings, <code class="language-plaintext highlighter-rouge">%d</code> for ints, <code class="language-plaintext highlighter-rouge">%f</code> for floats (<code class="language-plaintext highlighter-rouge">%.2f</code> for 2 decimal places),
<code class="language-plaintext highlighter-rouge">%b</code> for bools, and <code class="language-plaintext highlighter-rouge">%S</code> for strings with quotes around them (handy for
debugging). You can use <code class="language-plaintext highlighter-rouge">Printf.sprintf</code> instead of <code class="language-plaintext highlighter-rouge">printf</code> to get a
string back rather than printing to stdout.</p>
</blockquote>

<p>This is all straightforward. The trouble starts when you want to print your
own types.</p>

<h2>The Running Example</h2>

<p>Let’s use something more fun than the usual <code class="language-plaintext highlighter-rouge">person</code> record. We’ll model
superheroes:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
10
11
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">power</span> <span class="o">=</span> <span class="nc">Flight</span> <span class="o">|</span> <span class="nc">SuperStrength</span> <span class="o">|</span> <span class="nc">Telepathy</span> <span class="o">|</span> <span class="nc">Speed</span> <span class="o">|</span> <span class="nc">Gadgets</span>

<span class="k">type</span> <span class="n">strength</span> <span class="o">=</span> <span class="nc">Human</span> <span class="o">|</span> <span class="nc">Enhanced</span> <span class="o">|</span> <span class="nc">Superhuman</span> <span class="o">|</span> <span class="nc">Cosmic</span>

<span class="k">type</span> <span class="n">superhero</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">alias</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">powers</span> <span class="o">:</span> <span class="n">power</span> <span class="kt">list</span><span class="p">;</span>
  <span class="n">strength</span> <span class="o">:</span> <span class="n">strength</span><span class="p">;</span>
  <span class="n">first_appearance</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>  <span class="c">(* year *)</span>
<span class="p">}</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>And a few heroes to work with:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">batman</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">=</span> <span class="s2">"Bruce Wayne"</span><span class="p">;</span>
  <span class="n">alias</span> <span class="o">=</span> <span class="s2">"Batman"</span><span class="p">;</span>
  <span class="n">powers</span> <span class="o">=</span> <span class="p">[</span><span class="nc">Gadgets</span><span class="p">];</span>
  <span class="n">strength</span> <span class="o">=</span> <span class="nc">Human</span><span class="p">;</span>
  <span class="n">first_appearance</span> <span class="o">=</span> <span class="mi">1939</span><span class="p">;</span>
<span class="p">}</span>

<span class="k">let</span> <span class="n">superman</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">=</span> <span class="s2">"Clark Kent"</span><span class="p">;</span>
  <span class="n">alias</span> <span class="o">=</span> <span class="s2">"Superman"</span><span class="p">;</span>
  <span class="n">powers</span> <span class="o">=</span> <span class="p">[</span><span class="nc">Flight</span><span class="p">;</span> <span class="nc">SuperStrength</span><span class="p">];</span>
  <span class="n">strength</span> <span class="o">=</span> <span class="nc">Cosmic</span><span class="p">;</span>
  <span class="n">first_appearance</span> <span class="o">=</span> <span class="mi">1938</span><span class="p">;</span>
<span class="p">}</span>

<span class="k">let</span> <span class="n">wonder_woman</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">=</span> <span class="s2">"Diana Prince"</span><span class="p">;</span>
  <span class="n">alias</span> <span class="o">=</span> <span class="s2">"Wonder Woman"</span><span class="p">;</span>
  <span class="n">powers</span> <span class="o">=</span> <span class="p">[</span><span class="nc">Flight</span><span class="p">;</span> <span class="nc">SuperStrength</span><span class="p">;</span> <span class="nc">Telepathy</span><span class="p">];</span>
  <span class="n">strength</span> <span class="o">=</span> <span class="nc">Superhuman</span><span class="p">;</span>
  <span class="n">first_appearance</span> <span class="o">=</span> <span class="mi">1941</span><span class="p">;</span>
<span class="p">}</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>Now try <code class="language-plaintext highlighter-rouge">print_endline batman</code> and… you get a type error. OCaml has no
idea how to turn a <code class="language-plaintext highlighter-rouge">superhero</code> into a string. Let’s fix that.</p>

<h2>Writing Manual Printers</h2>

<p>The most basic approach is to write dedicated print functions. For our
superhero types, we’d need to handle each layer:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">string_of_power</span> <span class="o">=</span> <span class="k">function</span>
  <span class="o">|</span> <span class="nc">Flight</span> <span class="o">-&gt;</span> <span class="s2">"Flight"</span>
  <span class="o">|</span> <span class="nc">SuperStrength</span> <span class="o">-&gt;</span> <span class="s2">"Super Strength"</span>
  <span class="o">|</span> <span class="nc">Telepathy</span> <span class="o">-&gt;</span> <span class="s2">"Telepathy"</span>
  <span class="o">|</span> <span class="nc">Speed</span> <span class="o">-&gt;</span> <span class="s2">"Speed"</span>
  <span class="o">|</span> <span class="nc">Gadgets</span> <span class="o">-&gt;</span> <span class="s2">"Gadgets"</span>

<span class="k">let</span> <span class="n">string_of_strength</span> <span class="o">=</span> <span class="k">function</span>
  <span class="o">|</span> <span class="nc">Human</span> <span class="o">-&gt;</span> <span class="s2">"Human"</span>
  <span class="o">|</span> <span class="nc">Enhanced</span> <span class="o">-&gt;</span> <span class="s2">"Enhanced"</span>
  <span class="o">|</span> <span class="nc">Superhuman</span> <span class="o">-&gt;</span> <span class="s2">"Superhuman"</span>
  <span class="o">|</span> <span class="nc">Cosmic</span> <span class="o">-&gt;</span> <span class="s2">"Cosmic"</span>

<span class="k">let</span> <span class="n">show_superhero</span> <span class="n">h</span> <span class="o">=</span>
  <span class="nn">Printf</span><span class="p">.</span><span class="n">sprintf</span> <span class="s2">"%s (%s) - %s, first appeared in %d, powers: %s"</span>
    <span class="n">h</span><span class="o">.</span><span class="n">alias</span> <span class="n">h</span><span class="o">.</span><span class="n">name</span>
    <span class="p">(</span><span class="n">string_of_strength</span> <span class="n">h</span><span class="o">.</span><span class="n">strength</span><span class="p">)</span>
    <span class="n">h</span><span class="o">.</span><span class="n">first_appearance</span>
    <span class="p">(</span><span class="n">h</span><span class="o">.</span><span class="n">powers</span> <span class="o">|&gt;</span> <span class="nn">List</span><span class="p">.</span><span class="n">map</span> <span class="n">string_of_power</span> <span class="o">|&gt;</span> <span class="nn">String</span><span class="p">.</span><span class="n">concat</span> <span class="s2">", "</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
</pre></td><td class="rouge-code"><pre><span class="n">print_endline</span> <span class="p">(</span><span class="n">show_superhero</span> <span class="n">batman</span><span class="p">);;</span>
<span class="c">(* Batman (Bruce Wayne) - Human, first appeared in 1939, powers: Gadgets *)</span>

<span class="n">print_endline</span> <span class="p">(</span><span class="n">show_superhero</span> <span class="n">wonder_woman</span><span class="p">);;</span>
<span class="c">(* Wonder Woman (Diana Prince) - Superhuman, first appeared in 1941, powers: Flight, Super Strength, Telepathy *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>This works, but it’s tedious. We had to write a conversion function for
every variant type and compose them manually. Every time you add a new
power or a new field to the record, you have to remember to update all
the printers. And for nested types the boilerplate multiplies fast.
Coming from languages where printing just works out of the box, this
feels like a lot of ceremony for something that should be trivial.</p>

<p>That said, manual printers give you full control over formatting, which is
exactly what you want for user-facing output (error messages, CLI output,
log lines).</p>

<h2>Automatic Printing with ppx_deriving</h2>

<p>This is where <a href="https://github.com/ocaml-ppx/ppx_deriving">ppx_deriving</a>
saves the day. Its <code class="language-plaintext highlighter-rouge">show</code> plugin generates printer functions automatically
from your type definitions. Just add <code class="language-plaintext highlighter-rouge">[@@deriving show]</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
10
11
12
13
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">power</span> <span class="o">=</span> <span class="nc">Flight</span> <span class="o">|</span> <span class="nc">SuperStrength</span> <span class="o">|</span> <span class="nc">Telepathy</span> <span class="o">|</span> <span class="nc">Speed</span> <span class="o">|</span> <span class="nc">Gadgets</span>
<span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">show</span><span class="p">]</span>

<span class="k">type</span> <span class="n">strength</span> <span class="o">=</span> <span class="nc">Human</span> <span class="o">|</span> <span class="nc">Enhanced</span> <span class="o">|</span> <span class="nc">Superhuman</span> <span class="o">|</span> <span class="nc">Cosmic</span>
<span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">show</span><span class="p">]</span>

<span class="k">type</span> <span class="n">superhero</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">alias</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">powers</span> <span class="o">:</span> <span class="n">power</span> <span class="kt">list</span><span class="p">;</span>
  <span class="n">strength</span> <span class="o">:</span> <span class="n">strength</span><span class="p">;</span>
  <span class="n">first_appearance</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
<span class="p">}</span> <span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">show</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>That annotation generates two functions per type:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">pp_superhero : Format.formatter -&gt; superhero -&gt; unit</code> – a pretty-printer
for use with OCaml’s <code class="language-plaintext highlighter-rouge">Format</code> module</li>
  <li><code class="language-plaintext highlighter-rouge">show_superhero : superhero -&gt; string</code> – returns the string representation
directly</li>
</ul>

<p>Now printing is trivial:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
</pre></td><td class="rouge-code"><pre><span class="n">print_endline</span> <span class="p">(</span><span class="n">show_superhero</span> <span class="n">batman</span><span class="p">);;</span>
<span class="c">(* { name = "Bruce Wayne"; alias = "Batman";
     powers = [Gadgets]; strength = Human;
     first_appearance = 1939 } *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The generated printers are compositional – they know how to handle lists,
options, and other standard types automatically, as long as the element type
also derives <code class="language-plaintext highlighter-rouge">show</code>. So printing a list of heroes just works:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">justice_league</span> <span class="o">=</span> <span class="p">[</span><span class="n">batman</span><span class="p">;</span> <span class="n">superman</span><span class="p">;</span> <span class="n">wonder_woman</span><span class="p">]</span>

<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span> <span class="n">print_endline</span> <span class="p">([</span><span class="o">%</span><span class="n">show</span><span class="o">:</span> <span class="n">superhero</span> <span class="kt">list</span><span class="p">]</span> <span class="n">justice_league</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">[%show: superhero list]</code> syntax is really handy – it lets you derive a
printer for any type expression inline, without needing a separate type
declaration.</p>

<p>To use <code class="language-plaintext highlighter-rouge">ppx_deriving</code> in your project, add it to your <code class="language-plaintext highlighter-rouge">dune</code> file:</p>

<pre><code class="language-dune">(library
 (name mylib)
 (preprocess (pps ppx_deriving.show)))
</code></pre>

<p>I slap <code class="language-plaintext highlighter-rouge">[@@deriving show]</code> on pretty much every type I define these days.
The small compile-time cost is well worth the debugging convenience. If you
want to learn more about PPX in general, I wrote a <a href="https://batsov.com/articles/2026/03/03/ppx-for-mere-mortals/">longer
article</a> about it.</p>

<h2>Debugging Tips</h2>

<p>A few practical things I’ve picked up:</p>

<h3>Debug Printing Mid-pipeline</h3>

<p>You can sneak a print into a <code class="language-plaintext highlighter-rouge">|&gt;</code> chain without breaking the flow:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">debug</span> <span class="n">label</span> <span class="n">x</span> <span class="o">=</span>
  <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"[DEBUG] %s: %s</span><span class="se">\n</span><span class="s2">"</span> <span class="n">label</span> <span class="p">(</span><span class="n">show_superhero</span> <span class="n">x</span><span class="p">);</span>
  <span class="n">x</span>

<span class="k">let</span> <span class="n">result</span> <span class="o">=</span>
  <span class="n">batman</span>
  <span class="o">|&gt;</span> <span class="n">debug</span> <span class="s2">"before upgrade"</span>
  <span class="o">|&gt;</span> <span class="n">upgrade_powers</span>
  <span class="o">|&gt;</span> <span class="n">debug</span> <span class="s2">"after upgrade"</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<h3>Inline <code class="language-plaintext highlighter-rouge">[%show: ...]</code> for Complex Types</h3>

<p>When you need to print something and don’t want to define a type just for it:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"Heroes and powers: %s</span><span class="se">\n</span><span class="s2">"</span>
  <span class="p">([</span><span class="o">%</span><span class="n">show</span><span class="o">:</span> <span class="p">(</span><span class="kt">string</span> <span class="o">*</span> <span class="n">power</span> <span class="kt">list</span><span class="p">)</span> <span class="kt">list</span><span class="p">]</span>
    <span class="p">(</span><span class="nn">List</span><span class="p">.</span><span class="n">map</span> <span class="p">(</span><span class="k">fun</span> <span class="n">h</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="n">h</span><span class="o">.</span><span class="n">alias</span><span class="o">,</span> <span class="n">h</span><span class="o">.</span><span class="n">powers</span><span class="p">))</span> <span class="n">justice_league</span><span class="p">))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<h3>Watch Out for Missing <code class="language-plaintext highlighter-rouge">show</code> on Nested Types</h3>

<p>If your record contains a type that doesn’t derive <code class="language-plaintext highlighter-rouge">show</code>, you’ll get a
compile error about a missing <code class="language-plaintext highlighter-rouge">pp_</code> function. The error message can be
cryptic – just look for the type that’s missing the derivation and add
<code class="language-plaintext highlighter-rouge">[@@deriving show]</code> to it.</p>

<h3><code class="language-plaintext highlighter-rouge">{ with_path = false }</code> for Cleaner Output</h3>

<p>By default, derived printers qualify variant constructors with their module
path (e.g. <code class="language-plaintext highlighter-rouge">Superhero.Flight</code>). If you find this noisy:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">power</span> <span class="o">=</span> <span class="nc">Flight</span> <span class="o">|</span> <span class="nc">SuperStrength</span> <span class="o">|</span> <span class="nc">Telepathy</span> <span class="o">|</span> <span class="nc">Speed</span> <span class="o">|</span> <span class="nc">Gadgets</span>
<span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">show</span> <span class="p">{</span> <span class="n">with_path</span> <span class="o">=</span> <span class="bp">false</span> <span class="p">}]</span>

<span class="c">(* Now prints "Flight" instead of "Superhero.Flight" *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<h2>Printing in the Toplevel</h2>

<p>If you’re exploring code in <code class="language-plaintext highlighter-rouge">utop</code> (or the plain <code class="language-plaintext highlighter-rouge">ocaml</code> toplevel), you’ll
notice that it already displays values of built-in types:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
</pre></td><td class="rouge-code"><pre><span class="o">#</span> <span class="mi">42</span><span class="p">;;</span>
<span class="o">-</span> <span class="o">:</span> <span class="kt">int</span> <span class="o">=</span> <span class="mi">42</span>
<span class="o">#</span> <span class="p">[</span><span class="mi">1</span><span class="p">;</span> <span class="mi">2</span><span class="p">;</span> <span class="mi">3</span><span class="p">];;</span>
<span class="o">-</span> <span class="o">:</span> <span class="kt">int</span> <span class="kt">list</span> <span class="o">=</span> <span class="p">[</span><span class="mi">1</span><span class="p">;</span> <span class="mi">2</span><span class="p">;</span> <span class="mi">3</span><span class="p">]</span>
<span class="o">#</span> <span class="s2">"hello"</span><span class="p">;;</span>
<span class="o">-</span> <span class="o">:</span> <span class="kt">string</span> <span class="o">=</span> <span class="s2">"hello"</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>But for your own types, you just see the structure without much help:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="o">#</span> <span class="n">batman</span><span class="p">;;</span>
<span class="o">-</span> <span class="o">:</span> <span class="n">superhero</span> <span class="o">=</span> <span class="p">{</span><span class="n">name</span> <span class="o">=</span> <span class="s2">"Bruce Wayne"</span><span class="p">;</span> <span class="n">alias</span> <span class="o">=</span> <span class="s2">"Batman"</span><span class="p">;</span> <span class="o">...</span><span class="p">}</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>Actually, the toplevel does a decent job with simple records and variants –
it knows the structure from the type definition. Where it falls short is
with abstract types or types from external modules where the representation
is hidden.</p>

<p>If you’ve derived <code class="language-plaintext highlighter-rouge">show</code> for your types, you can register the generated
pretty-printer with the toplevel using <code class="language-plaintext highlighter-rouge">#install_printer</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="o">#</span> <span class="o">#</span><span class="n">install_printer</span> <span class="n">pp_superhero</span><span class="p">;;</span>
<span class="o">#</span> <span class="n">batman</span><span class="p">;;</span>
<span class="o">-</span> <span class="o">:</span> <span class="n">superhero</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span> <span class="o">=</span> <span class="s2">"Bruce Wayne"</span><span class="p">;</span> <span class="n">alias</span> <span class="o">=</span> <span class="s2">"Batman"</span><span class="p">;</span> <span class="o">...</span> <span class="p">}</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>This tells the toplevel to use your <code class="language-plaintext highlighter-rouge">pp_superhero</code> function whenever it
needs to display a <code class="language-plaintext highlighter-rouge">superhero</code> value. This is especially useful for abstract
types or when you want a custom representation. The function you install
must have the signature <code class="language-plaintext highlighter-rouge">Format.formatter -&gt; 'a -&gt; unit</code> – which is exactly
what <code class="language-plaintext highlighter-rouge">[@@deriving show]</code> generates as <code class="language-plaintext highlighter-rouge">pp_*</code>.</p>

<h2>When to Use What</h2>

<p>For debugging and development, <code class="language-plaintext highlighter-rouge">[@@deriving show]</code> is the obvious winner.
It’s become such a standard part of my OCaml workflow that I barely think
about it anymore.</p>

<p>For user-facing output (error messages, CLI formatting, log lines), you’ll
still want manual formatting with <code class="language-plaintext highlighter-rouge">Printf.sprintf</code> or <code class="language-plaintext highlighter-rouge">Format.asprintf</code>.
The derived printers produce a generic representation that’s great for
debugging but not something you’d want to show end users.</p>

<p>And in the toplevel, make friends with <code class="language-plaintext highlighter-rouge">#install_printer</code> – it makes
interactive exploration much more pleasant.</p>

<p>That’s all I have for you today. Keep hacking!</p>
