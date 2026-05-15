---
title: More OCaml on Pi Pico 2 W
description: Extending the Pico 2 implementation to add effects-based WiFi networking
  and improve the build system.
url: https://www.tunbury.org/2026/01/10/ocaml-pico/
date: 2026-01-10T21:00:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-pico.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Extending the Pico 2 implementation to add effects-based WiFi networking and improve the build system.</p>

<h1>Pio</h1>

<p>Pio is an effects-based I/O library for OCaml 5 running bare-metal on Raspberry Pi Pico 2 W. It provides an API compatible with <a href="https://github.com/ocaml-multicore/eio">Eio</a>, enabling direct-style concurrent programming with cooperative fibers and non-blocking network I/O. The Pico SDK provides lwIP and CYW43 drivers but these are not thread-safe.</p>

<p>The code matches the Eio style, for example:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* Main entry point *)</span>
<span class="nn">Pio</span><span class="p">.</span><span class="n">run</span> <span class="p">(</span><span class="k">fun</span> <span class="n">sw</span> <span class="o">-&gt;</span>
  <span class="c">(* Fork concurrent fibers *)</span>
  <span class="k">let</span> <span class="n">p1</span> <span class="o">=</span> <span class="nn">Pio</span><span class="p">.</span><span class="nn">Fiber</span><span class="p">.</span><span class="n">fork_promise</span> <span class="o">~</span><span class="n">sw</span> <span class="p">(</span><span class="k">fun</span> <span class="bp">()</span> <span class="o">-&gt;</span>
    <span class="nn">Net</span><span class="p">.</span><span class="nn">Tcp</span><span class="p">.</span><span class="n">connect</span> <span class="o">~</span><span class="n">host</span><span class="o">:</span><span class="s2">"example.com"</span> <span class="o">~</span><span class="n">port</span><span class="o">:</span><span class="mi">80</span>
    <span class="o">...</span>
  <span class="p">)</span> <span class="k">in</span>

  <span class="c">(* CPU work on Core 1 *)</span>
  <span class="k">let</span> <span class="n">d</span> <span class="o">=</span> <span class="nn">Domain</span><span class="p">.</span><span class="n">spawn</span> <span class="p">(</span><span class="k">fun</span> <span class="bp">()</span> <span class="o">-&gt;</span> <span class="n">heavy_computation</span> <span class="bp">()</span><span class="p">)</span> <span class="k">in</span>

  <span class="c">(* Await results *)</span>
  <span class="k">let</span> <span class="n">result1</span> <span class="o">=</span> <span class="nn">Pio</span><span class="p">.</span><span class="nn">Promise</span><span class="p">.</span><span class="n">await_exn</span> <span class="n">p1</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">result2</span> <span class="o">=</span> <span class="nn">Domain</span><span class="p">.</span><span class="n">join</span> <span class="n">d</span> <span class="k">in</span>
  <span class="o">...</span>
<span class="p">)</span>
</code></pre></div></div>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>  ┌─────────────────────────────────────────────────────────────┐
  │                    Pico 2 W (RP2350)                        │
  ├─────────────────────────────┬───────────────────────────────┤
  │         Core 0              │           Core 1              │
  │  ┌───────────────────────┐  │  ┌──────────────────────────┐ │
  │  │    Pio Scheduler      │  │  │    Domain.spawn          │ │
  │  │  ┌─────┐ ┌─────┐      │  │  │  ┌──────────────────┐    │ │
  │  │  │Fiber│ │Fiber│ ...  │  │  │  │ Pure computation │    │ │
  │  │  └──┬──┘ └──┬──┘      │  │  │  │ (no effects)     │    │ │
  │  │     └───┬───┘         │  │  │  └──────────────────┘    │ │
  │  │         ▼             │  │  └──────────────────────────┘ │
  │  │   Effect Handlers     │  │              │                │
  │  │   (Fork, Await,       │  │              │                │
  │  │    Tcp_*, Udp_*)      │  │              │                │
  │  └───────────────────────┘  │              │                │
  │            │                │              │                │
  │            ▼                │              │                │
  │    lwIP + CYW43 WiFi        │       Domain.join             │
  └─────────────────────────────┴───────────────────────────────┘
</code></pre></div></div>

<h1>Build system</h1>

<p>In a chance conversation with David, he was surprised that I had needed to do so much manual effort to complete the build. He pointed the <code class="language-plaintext highlighter-rouge">-output-obj</code> command line option to the compiler.</p>

<p>Compiling with <code class="language-plaintext highlighter-rouge">-output-obj -without-runtime</code> automatically provides, <code class="language-plaintext highlighter-rouge">caml_program</code>, <code class="language-plaintext highlighter-rouge">caml_globals</code>, <code class="language-plaintext highlighter-rouge">caml_code_segments</code>, <code class="language-plaintext highlighter-rouge">caml_exn_*</code>, all of which I had stubs for as well as <code class="language-plaintext highlighter-rouge">caml_frametable</code> which I covered with <code class="language-plaintext highlighter-rouge">frametable.S</code> and <code class="language-plaintext highlighter-rouge">caml_curry*</code> and <code class="language-plaintext highlighter-rouge">caml_apply*</code>, which I manually created from <code class="language-plaintext highlighter-rouge">curry.ml</code>.</p>

<p>The results in a single OCaml compilation step followed by a linking step for <code class="language-plaintext highlighter-rouge">ocaml_code.o</code> + <code class="language-plaintext highlighter-rouge">libasmrun.a</code></p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>/home/mtelvers/ocaml/ocamlopt.opt <span class="se">\</span>
    <span class="nt">-I</span> /home/mtelvers/ocaml/stdlib <span class="se">\</span>
    /home/mtelvers/ocaml/stdlib/stdlib.cmxa <span class="se">\</span>
    <span class="nt">-farch</span> armv8-m.main <span class="nt">-ffpu</span> soft <span class="nt">-fthumb</span> <span class="se">\</span>
    <span class="nt">-output-obj</span> <span class="nt">-without-runtime</span> <span class="se">\</span>
    <span class="nt">-o</span> <span class="k">${</span><span class="nv">CMAKE_CURRENT_BINARY_DIR</span><span class="k">}</span>/ocaml_code.o <span class="se">\</span>
    net.ml pio.ml hello.ml
</code></pre></div></div>

<p>The only disadvantage this gave me was that it used slightly more memory than before. The increased memory requirement came from properly initialising all the stdlib modules, where I had been selective before.</p>

<p>The space was recovered by reducing <code class="language-plaintext highlighter-rouge">POOL_WSIZE</code>, the allocation size for major heap pools.</p>

<ol>
  <li>Module initialisation creates OCaml values (closures, data structures, etc.)</li>
  <li>These values are first allocated in the minor heap (8KB per domain)</li>
  <li>When the minor heap fills up, or during GC, surviving objects are promoted to the major heap</li>
  <li>The major heap grows by allocating pools of <code class="language-plaintext highlighter-rouge">POOL_WSIZE</code> words (was 16KB reduced to 8KB)</li>
  <li>Multiple objects from multiple modules share pools</li>
</ol>

<p>Objects are packed into pools by size class which is where the saving is made. By default, there are 32 size classes, and objects of different sizes cannot share the same pool; thus, there can be underutilised pools. On a normal system this would matter, but with only 520KB of RAM this is significant.</p>

<p>With <code class="language-plaintext highlighter-rouge">POOL_WSIZE</code> at 4096, 17 pools were created for a total of 272K, but with the smaller 8K pools, there are 20 allocations, but only 160K used.</p>

<p>The change is made by editing <code class="language-plaintext highlighter-rouge">let arena = 2048</code> in <code class="language-plaintext highlighter-rouge">tools/gen_sizeclasses.ml</code> and regenerating <code class="language-plaintext highlighter-rouge">runtime/caml/sizeclasses.h</code>. The blocksizes function (lines 35-47) recursively builds size classes from 128 down to 1, adding a new size class whenever the overhead would exceed 10.1%. The change increases the number of size classes from 32 to 35.</p>
