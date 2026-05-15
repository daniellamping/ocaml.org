---
title: 'Pi Day 2026: OCaml vs OxCaml'
description: For Pi Day, I have implemented the same algorithm in both OCaml and OxCaml
  and compared the generated assembly and runtime performance.
url: https://www.tunbury.org/2026/03/14/pi-day/
date: 2026-03-14T03:14:15-00:00
preview_image: https://www.tunbury.org/images/pi.png
authors:
- Mark Elvers
source:
ignore:
---

<p>For Pi Day, I have implemented the same algorithm in both OCaml and OxCaml and compared the generated assembly and runtime performance.</p>

<p><a href="https://oxcaml.org">OxCaml</a> is a performance-focused extension of <a href="https://ocaml.org">OCaml</a> developed at Jane Street. It adds unboxed types, stack allocation, mutable bindings, and compile-time zero-allocation checking to the language. But what do these features actually look like in practice, and do they make a measurable difference?</p>

<h1>The algorithm</h1>

<p>This year, I’m going to use the <a href="https://en.wikipedia.org/wiki/Gauss%E2%80%93Legendre_algorithm">Gauss-Legendre algorithm</a> to compute pi by iteratively refining four variables. Starting from:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>a = 1,  b = 1/sqrt(2),  t = 1/4,  p = 1
</code></pre></div></div>

<p>each iteration updates them:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>a' = (a + b) / 2
b' = sqrt(a * b)
t' = t - p * (a - a')^2
p' = 2 * p
</code></pre></div></div>

<p>and pi is approximated by:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>pi ~ (a + b)^2 / (4 * t)
</code></pre></div></div>

<p>The code is deliberately structured as two functions: a <code class="language-plaintext highlighter-rouge">step</code> function that computes one iteration and returns the updated state, and a <code class="language-plaintext highlighter-rouge">gauss_legendre</code> function that iterates over <code class="language-plaintext highlighter-rouge">step</code> calls. The <code class="language-plaintext highlighter-rouge">step</code> function is marked <code class="language-plaintext highlighter-rouge">[@inline never]</code> to simulate a realistic cross-module call boundary of the kind that appears in real applications.</p>

<h1>OCaml</h1>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* pi_ocaml.ml - OCaml 5.4.0 *)</span>

<span class="k">let</span><span class="p">[</span><span class="o">@</span><span class="n">inline</span> <span class="n">never</span><span class="p">]</span> <span class="n">step</span> <span class="n">a</span> <span class="n">b</span> <span class="n">t</span> <span class="n">p</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">a'</span> <span class="o">=</span> <span class="p">(</span><span class="n">a</span> <span class="o">+.</span> <span class="n">b</span><span class="p">)</span> <span class="o">/.</span> <span class="mi">2</span><span class="o">.</span><span class="mi">0</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">b'</span> <span class="o">=</span> <span class="n">sqrt</span> <span class="p">(</span><span class="n">a</span> <span class="o">*.</span> <span class="n">b</span><span class="p">)</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">d</span> <span class="o">=</span> <span class="n">a</span> <span class="o">-.</span> <span class="n">a'</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">t'</span> <span class="o">=</span> <span class="n">t</span> <span class="o">-.</span> <span class="n">p</span> <span class="o">*.</span> <span class="n">d</span> <span class="o">*.</span> <span class="n">d</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">p'</span> <span class="o">=</span> <span class="mi">2</span><span class="o">.</span><span class="mi">0</span> <span class="o">*.</span> <span class="n">p</span> <span class="k">in</span>
  <span class="p">(</span><span class="n">a'</span><span class="o">,</span> <span class="n">b'</span><span class="o">,</span> <span class="n">t'</span><span class="o">,</span> <span class="n">p'</span><span class="p">)</span>

<span class="k">let</span><span class="p">[</span><span class="o">@</span><span class="n">inline</span> <span class="n">never</span><span class="p">]</span> <span class="n">gauss_legendre</span> <span class="bp">()</span> <span class="o">=</span>
  <span class="k">let</span> <span class="k">rec</span> <span class="n">loop</span> <span class="n">a</span> <span class="n">b</span> <span class="n">t</span> <span class="n">p</span> <span class="o">=</span> <span class="k">function</span>
    <span class="o">|</span> <span class="mi">0</span> <span class="o">-&gt;</span>
      <span class="k">let</span> <span class="n">s</span> <span class="o">=</span> <span class="n">a</span> <span class="o">+.</span> <span class="n">b</span> <span class="k">in</span>
      <span class="n">s</span> <span class="o">*.</span> <span class="n">s</span> <span class="o">/.</span> <span class="p">(</span><span class="mi">4</span><span class="o">.</span><span class="mi">0</span> <span class="o">*.</span> <span class="n">t</span><span class="p">)</span>
    <span class="o">|</span> <span class="n">i</span> <span class="o">-&gt;</span>
      <span class="k">let</span> <span class="p">(</span><span class="n">a'</span><span class="o">,</span> <span class="n">b'</span><span class="o">,</span> <span class="n">t'</span><span class="o">,</span> <span class="n">p'</span><span class="p">)</span> <span class="o">=</span> <span class="n">step</span> <span class="n">a</span> <span class="n">b</span> <span class="n">t</span> <span class="n">p</span> <span class="k">in</span>
      <span class="n">loop</span> <span class="n">a'</span> <span class="n">b'</span> <span class="n">t'</span> <span class="n">p'</span> <span class="p">(</span><span class="n">i</span> <span class="o">-</span> <span class="mi">1</span><span class="p">)</span>
  <span class="k">in</span>
  <span class="n">loop</span> <span class="mi">1</span><span class="o">.</span><span class="mi">0</span> <span class="p">(</span><span class="mi">1</span><span class="o">.</span><span class="mi">0</span> <span class="o">/.</span> <span class="n">sqrt</span> <span class="mi">2</span><span class="o">.</span><span class="mi">0</span><span class="p">)</span> <span class="mi">0</span><span class="o">.</span><span class="mi">25</span> <span class="mi">1</span><span class="o">.</span><span class="mi">0</span> <span class="mi">25</span>
</code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">step</code> function returns a <code class="language-plaintext highlighter-rouge">float * float * float * float</code> tuple.  The <code class="language-plaintext highlighter-rouge">gauss_legendre</code> function recurses, threading the four values through each iteration.</p>

<h1>OxCaml</h1>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* pi_oxcaml.ml - OxCaml 5.2.0+ox *)</span>

<span class="k">module</span> <span class="nc">F</span> <span class="o">=</span> <span class="nc">Float_u</span>
<span class="k">open</span> <span class="nn">Float_u</span><span class="p">.</span><span class="nc">O_dot</span>

<span class="k">let</span><span class="p">[</span><span class="o">@</span><span class="n">zero_alloc</span><span class="p">][</span><span class="o">@</span><span class="n">inline</span> <span class="n">never</span><span class="p">]</span> <span class="n">step</span> <span class="n">a</span> <span class="n">b</span> <span class="n">t</span> <span class="n">p</span>
  <span class="o">:</span> <span class="o">#</span><span class="p">(</span><span class="kt">float</span><span class="o">#</span> <span class="o">*</span> <span class="kt">float</span><span class="o">#</span> <span class="o">*</span> <span class="kt">float</span><span class="o">#</span> <span class="o">*</span> <span class="kt">float</span><span class="o">#</span><span class="p">)</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">a'</span> <span class="o">=</span> <span class="p">(</span><span class="n">a</span> <span class="o">+.</span> <span class="n">b</span><span class="p">)</span> <span class="o">/.</span> <span class="o">#</span><span class="mi">2</span><span class="o">.</span><span class="mi">0</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">b'</span> <span class="o">=</span> <span class="nn">F</span><span class="p">.</span><span class="n">sqrt</span> <span class="p">(</span><span class="n">a</span> <span class="o">*.</span> <span class="n">b</span><span class="p">)</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">d</span> <span class="o">=</span> <span class="n">a</span> <span class="o">-.</span> <span class="n">a'</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">t'</span> <span class="o">=</span> <span class="n">t</span> <span class="o">-.</span> <span class="n">p</span> <span class="o">*.</span> <span class="n">d</span> <span class="o">*.</span> <span class="n">d</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">p'</span> <span class="o">=</span> <span class="o">#</span><span class="mi">2</span><span class="o">.</span><span class="mi">0</span> <span class="o">*.</span> <span class="n">p</span> <span class="k">in</span>
  <span class="o">#</span><span class="p">(</span><span class="n">a'</span><span class="o">,</span> <span class="n">b'</span><span class="o">,</span> <span class="n">t'</span><span class="o">,</span> <span class="n">p'</span><span class="p">)</span>

<span class="k">let</span><span class="p">[</span><span class="o">@</span><span class="n">zero_alloc</span><span class="p">][</span><span class="o">@</span><span class="n">inline</span> <span class="n">never</span><span class="p">]</span> <span class="n">gauss_legendre</span> <span class="bp">()</span> <span class="o">:</span> <span class="kt">float</span><span class="o">#</span> <span class="o">=</span>
  <span class="k">let</span> <span class="k">mutable</span> <span class="n">a</span> <span class="o">=</span> <span class="o">#</span><span class="mi">1</span><span class="o">.</span><span class="mi">0</span> <span class="k">in</span>
  <span class="k">let</span> <span class="k">mutable</span> <span class="n">b</span> <span class="o">=</span> <span class="o">#</span><span class="mi">1</span><span class="o">.</span><span class="mi">0</span> <span class="o">/.</span> <span class="nn">F</span><span class="p">.</span><span class="n">sqrt</span> <span class="o">#</span><span class="mi">2</span><span class="o">.</span><span class="mi">0</span> <span class="k">in</span>
  <span class="k">let</span> <span class="k">mutable</span> <span class="n">t</span> <span class="o">=</span> <span class="o">#</span><span class="mi">0</span><span class="o">.</span><span class="mi">25</span> <span class="k">in</span>
  <span class="k">let</span> <span class="k">mutable</span> <span class="n">p</span> <span class="o">=</span> <span class="o">#</span><span class="mi">1</span><span class="o">.</span><span class="mi">0</span> <span class="k">in</span>
  <span class="k">for</span> <span class="n">_</span> <span class="o">=</span> <span class="mi">1</span> <span class="k">to</span> <span class="mi">25</span> <span class="k">do</span>
    <span class="k">let</span> <span class="o">#</span><span class="p">(</span><span class="n">a'</span><span class="o">,</span> <span class="n">b'</span><span class="o">,</span> <span class="n">t'</span><span class="o">,</span> <span class="n">p'</span><span class="p">)</span> <span class="o">=</span> <span class="n">step</span> <span class="n">a</span> <span class="n">b</span> <span class="n">t</span> <span class="n">p</span> <span class="k">in</span>
    <span class="n">a</span> <span class="o">&lt;-</span> <span class="n">a'</span><span class="p">;</span> <span class="n">b</span> <span class="o">&lt;-</span> <span class="n">b'</span><span class="p">;</span> <span class="n">t</span> <span class="o">&lt;-</span> <span class="n">t'</span><span class="p">;</span> <span class="n">p</span> <span class="o">&lt;-</span> <span class="n">p'</span>
  <span class="k">done</span><span class="p">;</span>
  <span class="k">let</span> <span class="n">s</span> <span class="o">=</span> <span class="n">a</span> <span class="o">+.</span> <span class="n">b</span> <span class="k">in</span>
  <span class="n">s</span> <span class="o">*.</span> <span class="n">s</span> <span class="o">/.</span> <span class="p">(</span><span class="o">#</span><span class="mi">4</span><span class="o">.</span><span class="mi">0</span> <span class="o">*.</span> <span class="n">t</span><span class="p">)</span>
</code></pre></div></div>

<p>Three OxCaml features are used here:</p>

<h2>unboxed types</h2>

<p>Instead of <code class="language-plaintext highlighter-rouge">float</code>, which would be a boxed 64-bit IEEE double, allocated on the heap and subject to garbage collection, a <code class="language-plaintext highlighter-rouge">float#</code> stores the value directly in a CPU register. The <code class="language-plaintext highlighter-rouge">#(float# * float# * float# * float#)</code> return type is an unboxed tuple: four values returned in four XMM registers, with no heap allocation at all.</p>

<h2>Stack-local mutable bindings</h2>

<p><code class="language-plaintext highlighter-rouge">let mutable</code> creates a mutable binding that lives on the stack.</p>

<h2>Compile-time allocation checking</h2>

<p>The <code class="language-plaintext highlighter-rouge">[@zero_alloc]</code> annotation asks the compiler to prove that the function performs no heap allocation. If any code path allocates, that is, boxing a float or creating a tuple and thereby potentially triggering the GC, the build fails with an error showing exactly where the allocation occurs.</p>

<h1>What the compiler generates</h1>

<p>Both versions perform the same arithmetic requiring the same instructions: <code class="language-plaintext highlighter-rouge">addsd</code>, <code class="language-plaintext highlighter-rouge">divsd</code>, <code class="language-plaintext highlighter-rouge">sqrtsd</code>, <code class="language-plaintext highlighter-rouge">mulsd</code>, <code class="language-plaintext highlighter-rouge">subsd</code> sequence. The difference is what happens when <code class="language-plaintext highlighter-rouge">step</code> needs to return its four results to the caller.</p>

<h2>OCaml: 104 bytes of heap allocation</h2>

<pre><code class="language-asm">    subq    $104, %r15              ; reserve 104 bytes on the minor heap
    cmpq    (%r14), %r15            ; enough space?
    jb      .L102                   ; if not → trigger GC

    ; Box each float (16 bytes each: 8-byte GC header + 8-byte value)
    movq    $1277, -8(%rbx)         ; GC tag for boxed float
    movsd   %xmm0, (%rbx)           ; store p'
    movq    $1277, -8(%rdi)         ; GC tag
    movsd   %xmm1, (%rdi)           ; store t'
    movq    $1277, -8(%rsi)         ; GC tag
    movsd   %xmm3, (%rsi)           ; store b'
    movq    $1277, -8(%rdx)         ; GC tag
    movsd   %xmm2, (%rdx)           ; store a'

    ; Build the 4-element tuple (40 bytes: 8-byte header + 4 pointers)
    movq    $4096, -8(%rax)         ; tuple header tag
    movq    %rdx, (%rax)            ; pointer → a'
    movq    %rsi, 8(%rax)           ; pointer → b'
    movq    %rdi, 16(%rax)          ; pointer → t'
    movq    %rbx, 24(%rax)          ; pointer → p'
    ret

.L102:
    call    caml_call_gc            ; garbage collection
    jmp     .L104
</code></pre>

<p>Every call to <code class="language-plaintext highlighter-rouge">step</code> allocates 104 bytes: four 16-byte boxed floats plus a 40-byte tuple to hold pointers to them. It also performs a GC boundary check and may trigger a garbage collection if the minor heap is full.</p>

<h2>OxCaml: zero allocation</h2>

<pre><code class="language-asm">    vaddsd  %xmm1, %xmm4, %xmm0   ; a' = (a + b) / 2
    vdivsd  %xmm3, %xmm0, %xmm0
    vmulsd  %xmm1, %xmm4, %xmm1   ; b' = sqrt(a * b)
    vsqrtsd %xmm1, %xmm6, %xmm1
    vsubsd  %xmm0, %xmm4, %xmm4   ; d = a - a'
    vmulsd  %xmm5, %xmm3, %xmm3   ; p' = 2 * p
    vmulsd  %xmm4, %xmm5, %xmm5   ; t' = t - p * d * d
    vmulsd  %xmm4, %xmm5, %xmm4
    vsubsd  %xmm4, %xmm2, %xmm2
    ret                           ; a' in xmm0, b' in xmm1,
                                  ; t' in xmm2, p' in xmm3
</code></pre>

<p>Just the pure arithmetic steps: four values go in via XMM registers, four come back out the same way.</p>

<h1>Benchmark</h1>

<p>The algorithm is too fast to benchmark on its own, so I run the computation in a for loop 10 million times, resulting in 250 million <code class="language-plaintext highlighter-rouge">step</code> calls and measured the results with hyperfine on Linux (10 runs, 3 warmup), showing that OxCaml is 1.44x faster.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Benchmark 1: ./pi_ocaml
  Time (mean +/- s):      2.931 s +/-  0.048 s    [User: 2.927 s, System: 0.004 s]
  Range (min ... max):    2.854 s ...  3.020 s    10 runs

Benchmark 2: ./pi_oxcaml
  Time (mean +/- s):      2.033 s +/-  0.090 s    [User: 2.018 s, System: 0.015 s]
  Range (min ... max):    1.945 s ...  2.188 s    10 runs

Summary
  ./pi_oxcaml ran
    1.44 +/- 0.07 times faster than ./pi_ocaml
</code></pre></div></div>

<h1>Results</h1>

<p>Both versions produce the correct value!</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Pi = 3.141592653589794  (error: 8.88e-16)
</code></pre></div></div>

<h1>Why <code class="language-plaintext highlighter-rouge">[@inline never]</code>?</h1>

<p>The <code class="language-plaintext highlighter-rouge">[@inline never]</code> annotation is worth mentioning as without it, the two compilers behave differently, resulting in a different comparison.</p>

<p>OxCaml inlines <code class="language-plaintext highlighter-rouge">step</code> into <code class="language-plaintext highlighter-rouge">gauss_legendre</code>, eliminating the function call entirely,  resulting in a loop which is just the register arithmetic with no call instruction at all:</p>

<pre><code class="language-asm">; OxCaml gauss_legendre WITHOUT [@inline never] - step is fully inlined
.L124:
    vmulsd  %xmm0, %xmm1, %xmm4   ; b' = sqrt(a * b)
    vsqrtsd %xmm4, %xmm5, %xmm4
    vaddsd  %xmm0, %xmm1, %xmm0   ; a' = (a + b) / 2
    vdivsd  %xmm5, %xmm0, %xmm0
    vsubsd  %xmm0, %xmm1, %xmm1   ; d = a - a'
    vmulsd  %xmm1, %xmm2, %xmm6   ; t' = t - p * d^2
    vmulsd  %xmm1, %xmm6, %xmm1
    vsubsd  %xmm1, %xmm3, %xmm3
    vmulsd  %xmm2, %xmm5, %xmm2   ; p' = 2 * p
    ...
    jmp     .L124                   ; loop — no call instruction anywhere
</code></pre>

<p>OCaml 5.4.0 does not inline <code class="language-plaintext highlighter-rouge">step</code>, even without the annotation. The loop still contains <code class="language-plaintext highlighter-rouge">call camlPi_ocaml_inline.step_274@PLT</code>, and every call still boxes four floats and builds a tuple on the heap:</p>

<pre><code class="language-asm">; OCaml gauss_legendre WITHOUT [@inline never] - step is NOT inlined
.L111:
    cmpq    $1, %rdx                ; base case check (i = 0)
    je      .L110
    movq    %rdx, (%rsp)            ; spill loop counter
    call    camlPi_ocaml_inline.step_274@PLT  ; still a real call
    movq    24(%rax), %rsi          ; unbox returned tuple
    movq    16(%rax), %rdi
    movq    8(%rax), %rbx
    movq    (%rax), %rax
    jmp     .L111                   ; loop - passing boxed pointers
</code></pre>

<p>So <code class="language-plaintext highlighter-rouge">[@inline never]</code> on the OxCaml version levels the playing field. It prevents flambda2 from optimising away the very overhead I was trying to measure. Without it, the gap would be even larger, because OxCaml would inline everything while OCaml would still be boxing.</p>

<p>In practice, <code class="language-plaintext highlighter-rouge">[@inline never]</code> also represents the realistic case: real applications are made of many modules, and calls across compilation-unit boundaries cannot be inlined by either compiler. The boxing overhead shown here occurs whenever a function in one <code class="language-plaintext highlighter-rouge">.ml</code> file returns multiple float values to a caller in another.</p>

<h1>Note</h1>

<p>I wrote this post last weekend in readiness for Pi Day, not realising that I would have the chance to use these techniques during the week in my <a href="https://www.tunbury.org/2026/03/13/oxcaml-inference/">OxCaml inference engine</a>.</p>
