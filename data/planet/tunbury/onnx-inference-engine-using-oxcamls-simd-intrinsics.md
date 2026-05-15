---
title: "ONNX inference engine using OxCaml\u2019s SIMD intrinsics"
description: Following my previous CPU vs GPU post I started thinking about what the
  ONNX inference engine actually did and if it could be replicated in OxCaml with
  SIMD.
url: https://www.tunbury.org/2026/03/13/oxcaml-inference/
date: 2026-03-13T18:30:00-00:00
preview_image: https://www.tunbury.org/images/tessera.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Following my previous <a href="https://www.tunbury.org/2026/03/11/gpu-vs-cpu/">CPU vs GPU</a> post I started thinking about what the ONNX inference engine actually did and if it could be replicated in <a href="https://oxcaml.org">OxCaml</a> with SIMD.</p>

<p>Protocol Buffers are Google’s language-neutral, platform-neutral serialisation format. ONNX uses them to define its model file format. The schema is defined at <a href="https://github.com/onnx/onnx/blob/main/onnx/onnx.proto">onnx/onnx.proto</a>.</p>

<p><a href="https://github.com/mransan/ocaml-protoc">mransan/ocaml-protoc</a> is a protobuf compiler for OCaml that can read the ONNX schema and generate OCaml types and interface files.</p>

<blockquote>
  <p>Tessera is a dual-backbone transformer that processes Sentinel-1 (SAR) and Sentinel-2 (multispectral) satellite imagery time-series into 128-dimensional embeddings.</p>
</blockquote>

<p>Analysing the Tessera model using a Python script showed the 25 ONNX operator types used by the model.</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kn">import</span> <span class="nn">onnx</span>
<span class="kn">from</span> <span class="nn">collections</span> <span class="kn">import</span> <span class="n">Counter</span>


<span class="n">model</span> <span class="o">=</span> <span class="n">onnx</span><span class="p">.</span><span class="n">load</span><span class="p">(</span><span class="s">"tessera_model.onnx"</span><span class="p">)</span>
<span class="n">ops</span> <span class="o">=</span> <span class="n">Counter</span><span class="p">(</span><span class="n">node</span><span class="p">.</span><span class="n">op_type</span> <span class="k">for</span> <span class="n">node</span> <span class="ow">in</span> <span class="n">model</span><span class="p">.</span><span class="n">graph</span><span class="p">.</span><span class="n">node</span><span class="p">)</span>
<span class="k">for</span> <span class="n">op</span><span class="p">,</span> <span class="n">count</span> <span class="ow">in</span> <span class="n">ops</span><span class="p">.</span><span class="n">most_common</span><span class="p">():</span>
    <span class="k">print</span><span class="p">(</span><span class="sa">f</span><span class="s">"</span><span class="si">{</span><span class="n">op</span><span class="si">:</span><span class="mi">20</span><span class="n">s</span><span class="si">}</span><span class="s"> </span><span class="si">{</span><span class="n">count</span><span class="si">:</span><span class="mi">4</span><span class="n">d</span><span class="si">}</span><span class="s">"</span><span class="p">)</span>
<span class="k">print</span><span class="p">(</span><span class="sa">f</span><span class="s">"</span><span class="se">\n</span><span class="si">{</span><span class="nb">len</span><span class="p">(</span><span class="n">ops</span><span class="p">)</span><span class="si">}</span><span class="s"> operator types, </span><span class="si">{</span><span class="nb">sum</span><span class="p">(</span><span class="n">ops</span><span class="p">.</span><span class="n">values</span><span class="p">())</span><span class="si">}</span><span class="s"> total nodes"</span><span class="p">)</span>
</code></pre></div></div>

<p>Operations used:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Add                   608
Gemm                  489
Mul                   262
Sigmoid               160
Gather                109
Unsqueeze              93
Reshape                90
Tanh                   80
Sub                    80
Transpose              75
MatMul                 46
Slice                  29
Concat                 26
LayerNormalization     18
Shape                  12
Relu                   10
Softmax                10
Squeeze                 9
ScatterND               4
Identity                4
Range                   3
Expand                  2
Sin                     2
Cos                     2
ReduceSum               2

25 operator types, 2225 total nodes
</code></pre></div></div>

<p>ocaml-protoc gives us the <code class="language-plaintext highlighter-rouge">.onnx</code> file parser and graph description. <code class="language-plaintext highlighter-rouge">ops.ml</code> implements what each operation does to tensors, and <code class="language-plaintext highlighter-rouge">graph.ml</code> walks the graph in topological order, feeding outputs of one operation as inputs into the next.</p>

<h1>Heap allocations</h1>

<p>The initial emphasis was on getting a working version; then it was time to optimise the code. Profiling shows that matrix multiplication (MatMul) was the dominant operation. For example, using <code class="language-plaintext highlighter-rouge">Float32.Bigstring.unsafe_get</code> rather than <code class="language-plaintext highlighter-rouge">Bigarray.Array1.get</code> was a huge saving. As were functions like <code class="language-plaintext highlighter-rouge">Base_bigstring.unsafe_blit</code> for bulk copies.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">get_f32_raw</span> <span class="p">(</span><span class="n">data</span> <span class="o">:</span> <span class="n">bigstring</span><span class="p">)</span> <span class="n">byte_off</span> <span class="o">=</span>
  <span class="nn">Stdlib_stable</span><span class="p">.</span><span class="nn">Float32</span><span class="p">.</span><span class="n">to_float</span>
    <span class="p">(</span><span class="nn">Stdlib_stable</span><span class="p">.</span><span class="nn">Float32</span><span class="p">.</span><span class="nn">Bigstring</span><span class="p">.</span><span class="n">unsafe_get</span> <span class="n">data</span> <span class="o">~</span><span class="n">pos</span><span class="o">:</span><span class="n">byte_off</span><span class="p">)</span>
</code></pre></div></div>

<p>The General Matrix Multiply (GEMM) inner loop broadcasts a scalar across 8 SIMD lanes. In code, <code class="language-plaintext highlighter-rouge">F32x8.set1</code> broadcasts one element of matrix A across all 8 lanes so it can be multiplied against 8 consecutive elements of matrix B in a single instruction.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>A[i, k] = 2.5 -&gt;  broadcast  -&gt;  [2.5, 2.5, 2.5, 2.5,  2.5, 2.5,  2.5, 2.5]
B[k, j..j+7]  =                  [1.0, 2.0, 3.0, 4.0,  5.0, 6.0,  7.0, 8.0]
multiply      -&gt;                 [2.5, 5.0, 7.5, 10., 12.5, 15., 17.5, 20.]
</code></pre></div></div>

<p>The CPU instruction is <code class="language-plaintext highlighter-rouge">vbroadcastss</code> aka “broadcast scalar single-precision” into a 256-bit YMM register. One cycle to fill all 8 lanes.</p>

<p>In the 4-row-unrolled version, the core looked like this:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">for</span> <span class="n">kk</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">to</span> <span class="n">k</span> <span class="o">-</span> <span class="mi">1</span> <span class="k">do</span>
  <span class="k">let</span> <span class="n">a_bc0</span> <span class="o">=</span> <span class="nn">F32x8</span><span class="p">.</span><span class="n">set1</span>
    <span class="p">(</span><span class="nn">Float32_u</span><span class="p">.</span><span class="n">of_float32</span>
      <span class="p">(</span><span class="nn">Float32</span><span class="p">.</span><span class="n">of_float</span>
        <span class="p">(</span><span class="n">get_f32_raw</span> <span class="n">a_data</span> <span class="p">(</span><span class="n">a_row0</span> <span class="o">+</span> <span class="n">kk</span> <span class="o">*</span> <span class="mi">4</span><span class="p">))))</span> <span class="k">in</span>
  <span class="o">...</span>
  <span class="c">(* 4 rows x SIMD FMA inner loop *)</span>
<span class="k">done</span>
</code></pre></div></div>

<p>This looks reasonable. <code class="language-plaintext highlighter-rouge">get_f32_raw</code> reads the value. <code class="language-plaintext highlighter-rouge">Float32.of_float</code> converts to float32. <code class="language-plaintext highlighter-rouge">Float32_u.of_float32</code> unboxes it. <code class="language-plaintext highlighter-rouge">F32x8.set1</code> broadcasts to all 8 lanes.</p>

<p>The problem is <code class="language-plaintext highlighter-rouge">Float32.of_float</code>, which returns a <code class="language-plaintext highlighter-rouge">float32</code>. A <strong>boxed</strong> 32-bit float. Boxed means heap-allocated, so every call allocates 16 bytes on the heap.</p>

<p>With 4 rows and K=512, that’s 2,048 heap allocations per GEMM call just for the broadcast. For the 46 MatMuls in the model, roughly 20,000 allocations per inference.</p>

<p>OxCaml’s <code class="language-plaintext highlighter-rouge">[@zero_alloc]</code> annotation asks the compiler to verify that a function performs no heap allocation. The function annotation looks like this:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span><span class="p">[</span><span class="o">@</span><span class="n">zero_alloc</span><span class="p">]</span> <span class="n">gemm_broadcast</span> <span class="o">...</span> <span class="o">=</span>
  <span class="k">for</span> <span class="n">kk</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">to</span> <span class="n">k</span> <span class="o">-</span> <span class="mi">1</span> <span class="k">do</span>
    <span class="k">let</span> <span class="n">a_bc0</span> <span class="o">=</span> <span class="nn">F32x8</span><span class="p">.</span><span class="n">set1</span>
      <span class="p">(</span><span class="nn">Float32_u</span><span class="p">.</span><span class="n">of_float32</span>
        <span class="p">(</span><span class="nn">Float32</span><span class="p">.</span><span class="n">of_float</span>
          <span class="p">(</span><span class="n">get_f32_raw</span> <span class="n">a_data</span> <span class="p">(</span><span class="n">a_row0</span> <span class="o">+</span> <span class="n">kk</span> <span class="o">*</span> <span class="mi">4</span><span class="p">))))</span> <span class="k">in</span>
   <span class="o">...</span>
  <span class="k">done</span>
</code></pre></div></div>

<p>The compiler rejected it:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Error: Annotation check for zero_alloc failed.

  (Float32.of_float
  ^^^^^^^^^^^^^^^^^
Error: called function may allocate
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">Float32.of_float</code> returns a boxed <code class="language-plaintext highlighter-rouge">float32</code>, meaning that there would be heap allocation. The compiler caught it instantly. OxCaml has a complete unboxed float32 pipeline. The key types:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">float32#</code> unboxed 32-bit float (kind <code class="language-plaintext highlighter-rouge">float32</code>, not <code class="language-plaintext highlighter-rouge">value</code>)</li>
  <li><code class="language-plaintext highlighter-rouge">float32</code> boxed 32-bit float (heap-allocated, kind <code class="language-plaintext highlighter-rouge">value</code>)</li>
  <li><code class="language-plaintext highlighter-rouge">float</code> standard 64-bit float (OCaml’s usual <code class="language-plaintext highlighter-rouge">float</code>)</li>
</ul>

<p>The allocating path went:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>bigstring -&gt; Float32.Bigstring.unsafe_get -&gt; float32 (boxed)
          -&gt; Float32.to_float             -&gt; float   (boxed)
          -&gt; Float32.of_float             -&gt; float32 (boxed)
          -&gt; Float32_u.of_float32         -&gt; float32# (unboxed)
          -&gt; F32x8.set1
</code></pre></div></div>

<p>Three boxed intermediates. The zero-alloc path:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>bigstring -&gt; Float32_u.Bigstring.unsafe_get -&gt; float32# (unboxed)
          -&gt; F32x8.set1
</code></pre></div></div>

<p>One step with zero allocations. The primitive <code class="language-plaintext highlighter-rouge">%caml_bigstring_getf32u#</code> reads a float32 directly into an unboxed register.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span><span class="p">[</span><span class="o">@</span><span class="n">inline</span> <span class="n">always</span><span class="p">]</span> <span class="n">get_f32u</span> <span class="p">(</span><span class="n">data</span> <span class="o">:</span> <span class="n">bigstring</span><span class="p">)</span> <span class="n">byte_off</span> <span class="o">:</span> <span class="n">float32</span><span class="o">#</span> <span class="o">=</span>
  <span class="nn">F32u</span><span class="p">.</span><span class="nn">Bigstring</span><span class="p">.</span><span class="n">unsafe_get</span> <span class="n">data</span> <span class="o">~</span><span class="n">pos</span><span class="o">:</span><span class="n">byte_off</span>
</code></pre></div></div>

<h1>Cross-module inlining detector</h1>

<p>With the boxing addresses, annotating any hot functions like the dot product function seemed logical to highlight any allocations.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span><span class="p">[</span><span class="o">@</span><span class="n">zero_alloc</span><span class="p">]</span> <span class="n">simd_dot_f32u</span> <span class="p">(</span><span class="n">a_data</span> <span class="o">:</span> <span class="n">bigstring</span><span class="p">)</span> <span class="n">a_byte</span>
    <span class="p">(</span><span class="n">b_data</span> <span class="o">:</span> <span class="n">bigstring</span><span class="p">)</span> <span class="n">b_byte</span> <span class="n">len</span> <span class="o">:</span> <span class="n">float32</span><span class="o">#</span> <span class="o">=</span>
  <span class="o">...</span>
  <span class="k">while</span> <span class="n">kk</span> <span class="o">&lt;</span> <span class="n">len</span> <span class="k">do</span>
    <span class="n">sum</span> <span class="o">&lt;-</span> <span class="nn">F32u</span><span class="p">.</span><span class="n">fma</span> <span class="p">(</span><span class="n">get_f32u</span> <span class="n">a_data</span> <span class="p">(</span><span class="n">a_byte</span> <span class="o">+</span> <span class="n">kk4</span><span class="p">))</span>
                    <span class="p">(</span><span class="n">get_f32u</span> <span class="n">b_data</span> <span class="p">(</span><span class="n">b_byte</span> <span class="o">+</span> <span class="n">kk4</span><span class="p">))</span> <span class="n">sum</span><span class="p">;</span>
    <span class="o">...</span>
  <span class="k">done</span><span class="p">;</span>
  <span class="n">sum</span>
</code></pre></div></div>

<p>The compiler rejected this but for a different reason:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Error: Annotation check for zero_alloc failed on function simd_dot_f32u.

  sum &lt;- F32u.fma (get_f32u a_data (a_byte + kk4))
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Error: called function may allocate (direct call caml_apply2_RS)
</code></pre></div></div>

<p>This time the code was correctly returning an unboxed type: <code class="language-plaintext highlighter-rouge">float32#</code>, but the function was defined in another module, and the compiler couldn’t inline it across the module boundary. It fell back to a generic <code class="language-plaintext highlighter-rouge">caml_apply2</code> calling convention, which <em>might</em> allocate.</p>

<p>The fix was to move the <code class="language-plaintext highlighter-rouge">get_f32u</code> definition into the same module and mark it <code class="language-plaintext highlighter-rouge">[@inline always]</code>. With inlining, the compiler could verify that nothing is allocated.</p>

<p>With both unboxed types and same-module inlining the compiler accepted <code class="language-plaintext highlighter-rouge">[@zero_alloc]</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span><span class="p">[</span><span class="o">@</span><span class="n">zero_alloc</span><span class="p">]</span> <span class="n">simd_dot_f32u</span> <span class="p">(</span><span class="n">a_data</span> <span class="o">:</span> <span class="n">bigstring</span><span class="p">)</span> <span class="n">a_byte</span>
    <span class="p">(</span><span class="n">b_data</span> <span class="o">:</span> <span class="n">bigstring</span><span class="p">)</span> <span class="n">b_byte</span> <span class="n">len</span> <span class="o">:</span> <span class="n">float32</span><span class="o">#</span> <span class="o">=</span>
  <span class="k">let</span> <span class="k">mutable</span> <span class="n">acc</span> <span class="o">=</span> <span class="nn">F32x8</span><span class="p">.</span><span class="n">zero</span> <span class="bp">()</span> <span class="k">in</span>
  <span class="k">let</span> <span class="k">mutable</span> <span class="n">kk</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">in</span>
  <span class="k">while</span> <span class="n">kk</span> <span class="o">+</span> <span class="mi">8</span> <span class="o">&lt;=</span> <span class="n">len</span> <span class="k">do</span>
    <span class="k">let</span> <span class="n">va</span> <span class="o">=</span> <span class="nn">F32x8</span><span class="p">.</span><span class="nn">Bigstring</span><span class="p">.</span><span class="n">unsafe_unaligned_get</span> <span class="n">a_data</span> <span class="o">~</span><span class="n">byte</span><span class="o">:</span><span class="p">(</span><span class="n">a_byte</span> <span class="o">+</span> <span class="n">kk</span> <span class="o">*</span> <span class="mi">4</span><span class="p">)</span> <span class="k">in</span>
    <span class="k">let</span> <span class="n">vb</span> <span class="o">=</span> <span class="nn">F32x8</span><span class="p">.</span><span class="nn">Bigstring</span><span class="p">.</span><span class="n">unsafe_unaligned_get</span> <span class="n">b_data</span> <span class="o">~</span><span class="n">byte</span><span class="o">:</span><span class="p">(</span><span class="n">b_byte</span> <span class="o">+</span> <span class="n">kk</span> <span class="o">*</span> <span class="mi">4</span><span class="p">)</span> <span class="k">in</span>
    <span class="n">acc</span> <span class="o">&lt;-</span> <span class="nn">F32x8</span><span class="p">.</span><span class="n">mul_add</span> <span class="n">va</span> <span class="n">vb</span> <span class="n">acc</span><span class="p">;</span>
    <span class="n">kk</span> <span class="o">&lt;-</span> <span class="n">kk</span> <span class="o">+</span> <span class="mi">8</span>
  <span class="k">done</span><span class="p">;</span>
  <span class="k">let</span> <span class="k">mutable</span> <span class="n">sum</span> <span class="o">=</span> <span class="nn">F32x8</span><span class="p">.</span><span class="n">dot</span> <span class="n">acc</span> <span class="p">(</span><span class="nn">F32x8</span><span class="p">.</span><span class="n">one</span> <span class="bp">()</span><span class="p">)</span> <span class="k">in</span>
  <span class="k">while</span> <span class="n">kk</span> <span class="o">&lt;</span> <span class="n">len</span> <span class="k">do</span>
    <span class="k">let</span> <span class="n">kk4</span> <span class="o">=</span> <span class="n">kk</span> <span class="o">*</span> <span class="mi">4</span> <span class="k">in</span>
    <span class="n">sum</span> <span class="o">&lt;-</span> <span class="nn">F32u</span><span class="p">.</span><span class="n">fma</span> <span class="p">(</span><span class="n">get_f32u</span> <span class="n">a_data</span> <span class="p">(</span><span class="n">a_byte</span> <span class="o">+</span> <span class="n">kk4</span><span class="p">))</span>
                    <span class="p">(</span><span class="n">get_f32u</span> <span class="n">b_data</span> <span class="p">(</span><span class="n">b_byte</span> <span class="o">+</span> <span class="n">kk4</span><span class="p">))</span> <span class="n">sum</span><span class="p">;</span>
    <span class="n">kk</span> <span class="o">&lt;-</span> <span class="n">kk</span> <span class="o">+</span> <span class="mi">1</span>
  <span class="k">done</span><span class="p">;</span>
  <span class="n">sum</span>
</code></pre></div></div>

<p>Every value has an unboxed layout. <code class="language-plaintext highlighter-rouge">acc</code> is <code class="language-plaintext highlighter-rouge">float32x8#</code> (layout <code class="language-plaintext highlighter-rouge">vec256</code>). <code class="language-plaintext highlighter-rouge">sum</code> is <code class="language-plaintext highlighter-rouge">float32#</code> (layout <code class="language-plaintext highlighter-rouge">float32</code>). <code class="language-plaintext highlighter-rouge">kk</code> is <code class="language-plaintext highlighter-rouge">int</code>. None of these can be stored in a <code class="language-plaintext highlighter-rouge">ref</code>; instead, use OxCaml’s <code class="language-plaintext highlighter-rouge">let mutable</code> instead. The compiler then verifies that the whole function allocates nothing.</p>

<p>The <code class="language-plaintext highlighter-rouge">[@zero_alloc]</code> annotation was extended to every GEMM, elementwise, and activation function, replacing all cross-module scalar accessors with inline unboxed operations and replacing <code class="language-plaintext highlighter-rouge">numel</code> (which uses <code class="language-plaintext highlighter-rouge">Array.fold_left</code>, an indirect call the compiler can’t prove allocation-free) with an inline loop. The scalar-only functions benefited most: Sigmoid dropped from 2.4 ms to 1.1 ms (54%), Tanh from 2.0 ms to 1.1 ms (45%), Softmax from 1.6 ms to 0.6 ms (63%).</p>

<h1>Graph-level optimisation</h1>

<p>At this point, I became obsessed with the idea that ONNX must do something slightly different to just following the graph operations. With this, the next performance boost came from analysis of the graph and removing redundant passes over the data. For example, two matrix multiplications added together can be combined into a single <code class="language-plaintext highlighter-rouge">GemmPairAdd</code> operation. These graph-level passes reduced the node count from 2,225 to 1,779 and brought inference from 410 ms to 230 ms. A further 1.55x speedup on top of the kernel-level gains.</p>

<h1><code class="language-plaintext highlighter-rouge">let mutable</code> notes</h1>

<p>Standard OCaml’s <code class="language-plaintext highlighter-rouge">ref</code> is a heap-allocated record:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">type</span> <span class="k">'</span><span class="n">a</span> <span class="n">ref</span> <span class="o">=</span> <span class="p">{</span> <span class="k">mutable</span> <span class="n">contents</span> <span class="o">:</span> <span class="k">'</span><span class="n">a</span> <span class="p">}</span>
</code></pre></div></div>

<p>The type parameter <code class="language-plaintext highlighter-rouge">'a</code> must have a layout <code class="language-plaintext highlighter-rouge">value</code>. It must be a pointer-sized GC-traceable value. But <code class="language-plaintext highlighter-rouge">float32x8#</code> has layout <code class="language-plaintext highlighter-rouge">vec256</code> and <code class="language-plaintext highlighter-rouge">float32#</code> has layout <code class="language-plaintext highlighter-rouge">float32</code>. The type parameter of <code class="language-plaintext highlighter-rouge">ref</code> requires layout <code class="language-plaintext highlighter-rouge">value</code>, so the compiler won’t let you write <code class="language-plaintext highlighter-rouge">ref (F32x8.zero ())</code>.  OxCaml’s <code class="language-plaintext highlighter-rouge">let mutable</code> provides mutation without allocation:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="k">mutable</span> <span class="n">acc</span> <span class="o">=</span> <span class="nn">F32x8</span><span class="p">.</span><span class="n">zero</span> <span class="bp">()</span> <span class="k">in</span>
<span class="k">while</span> <span class="o">...</span> <span class="k">do</span>
  <span class="n">acc</span> <span class="o">&lt;-</span> <span class="nn">F32x8</span><span class="p">.</span><span class="n">mul_add</span> <span class="n">va</span> <span class="n">vb</span> <span class="n">acc</span><span class="p">;</span>   <span class="c">(* mutate in-place, in register *)</span>
  <span class="o">...</span>
<span class="k">done</span>
</code></pre></div></div>

<p>The variable lives in a register or on the stack with no heap allocation, no GC interaction, no pointer indirection. This is the construct that makes zero-alloc SIMD accumulation possible at all.</p>

<h1>OxCaml value add</h1>

<p>The standard OCaml compiler produces fast code, and we can call SIMD intrinsics via C stubs without OxCaml. The boundary between “fast” and “as fast as possible” is where OxCaml’s extensions sit: unboxed floats prevent heap allocation, <code class="language-plaintext highlighter-rouge">let mutable</code> provides register-resident mutable variables, <code class="language-plaintext highlighter-rouge">[@zero_alloc]</code> provides static allocation checking to identify invisible boxing in the hot path. The code is available at <a href="https://github.com/mtelvers/oxcaml-infer">mtelvers/oxcaml-infer</a></p>

<h1>The numbers</h1>

<p>The OxCaml engine is currently single-threaded. I’m running it on a 20-core Xeon E5-2640 v4:</p>

<table>
  <thead>
    <tr>
      <th>Engine</th>
      <th>Latency</th>
      <th>Relative</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ONNX Runtime 1.24 (1 thread)</td>
      <td>88 ms</td>
      <td>1.0x</td>
    </tr>
    <tr>
      <td>OxCaml + AVX2 (initial)</td>
      <td>845 ms</td>
      <td>~10x slower</td>
    </tr>
    <tr>
      <td>OxCaml + AVX2 (optimised)</td>
      <td>200 ms</td>
      <td>~2.2x slower</td>
    </tr>
    <tr>
      <td>ONNX Runtime 1.24 (default, 8+ threads)</td>
      <td>27 ms</td>
      <td>&nbsp;</td>
    </tr>
  </tbody>
</table>

<h1>Try OxCaml yourself</h1>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code>opam switch create 5.2.0+ox <span class="nt">--repos</span> <span class="nv">ox</span><span class="o">=</span>git+https://github.com/oxcaml/opam-repository.git,default
opam <span class="nb">install </span>ocaml-protoc
git clone https://github.com/mtelvers/oxcaml-infer
<span class="nb">cd </span>oxcaml-infer
dune build
dune <span class="nb">exec </span>bin/main.exe <span class="nt">--</span> tessera_model.onnx
</code></pre></div></div>
