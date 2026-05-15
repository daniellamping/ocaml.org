---
title: Multi Domain OCaml on Raspberry Pi Pico 2 Microcontroller
description: Running OCaml 5 with multicore support on bare-metal Raspberry Pi Pico
  2 W (RP2350, ARM Cortex-M33).
url: https://www.tunbury.org/2025/12/31/ocaml-pico/
date: 2025-12-31T17:00:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-pico.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Running OCaml 5 with multicore support on bare-metal Raspberry Pi Pico 2 W (RP2350, ARM Cortex-M33).</p>

<p>The OCaml Arm32 backend, which <a href="https://www.tunbury.org/2025/11/27/ocaml-54-native/">I updated to OCaml 5 Domains</a>, generates ARMv7-A code (Application profile), but the Pico 2’s Cortex-M33 is ARMv8-M (Microcontroller profile). These instruction sets are compatible (both using Thumb-2), but the object file metadata differs. The linker will not mix “A” and “M” profiles.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>error: hello.o: conflicting architecture profiles A/M
</code></pre></div></div>

<p>Initially, I worked with the existing Arm32 support, compiling to assembly files from OCaml and then patching them with <code class="language-plaintext highlighter-rouge">sed</code> and reassembling with <code class="language-plaintext highlighter-rouge">arm-none-eabi-as</code> to get a Cortex-M compatible object file.</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nb">sed</span> <span class="nt">-e</span> <span class="s1">'s/.arch[[:space:]]*armv7-a/.arch armv8-m.main/'</span> <span class="se">\</span>
    <span class="nt">-e</span> <span class="s1">'s/.fpu[[:space:]]*softvfp/.fpu fpv5-sp-d16/'</span> <span class="se">\</span>
    hello.s.orig <span class="o">&gt;</span> hello.s
</code></pre></div></div>

<p>After a while, I decided to add a new architecture to the ARM backend to avoid the external processing. The Cortex-M33 has a single-precision only FPU. OCaml’s float type is double-precision (64-bit), so the hardware FPU cannot accelerate OCaml floats. The default Pico SDK linker script copies some code to RAM for faster execution, including the soft FPU. I have used a custom linker script to put everything in flash to maximise the memory available for the OCaml heap.</p>

<p>Creating a minimal runtime was relatively simple. OCaml’s calling convention puts the function pointer in r7 and calls <code class="language-plaintext highlighter-rouge">caml_c_call</code>. My function calls <code class="language-plaintext highlighter-rouge">blx r7</code> to invoke the actual C function. OCaml expects r8, r10, r11 to hold runtime state, so these are initialised with minimal structures.</p>

<ul>
  <li>r8 - trap_ptr (exception handler)</li>
  <li>r10 - alloc_ptr (allocation pointer)</li>
  <li>r11 - domain_state_ptr (runtime state)</li>
</ul>

<p>Thus, creating a simple program using OCaml syntax was now possible. It was also possible to have recursive functions to calculate a factorial; however, there was no garbage collector, no exception handling, no standard library and no multicore/domain support.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">external</span> <span class="n">pico_print</span> <span class="o">:</span> <span class="kt">string</span> <span class="o">-&gt;</span> <span class="kt">unit</span> <span class="o">=</span> <span class="s2">"pico_print"</span>

<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span> <span class="n">pico_print</span> <span class="s2">"Hello from OCaml!"</span>
</code></pre></div></div>

<p>This limited success, though, was enough to inspire me to push on to the second phase. I added per-core thread-local storage and provided a mapping between pthread and Pico SDK primitives. The Pico SDK does not provide condition variables, so I implemented a simple polling solution.</p>

<p>OCaml’s <code class="language-plaintext highlighter-rouge">Domain.spawn</code> calls <code class="language-plaintext highlighter-rouge">pthread_create()</code>, which now calls <code class="language-plaintext highlighter-rouge">multicore_launch_core1_with_stack()</code> from the Pico SDK. OCaml creates a backup thread which handles stop-the-world GC synchronisation when a domain’s main thread is blocked. On the Pico, I fake the creation of the backup thread by only creating a thread on every other call to <code class="language-plaintext highlighter-rouge">pthread_create()</code>. Since there is no backup thread, during <code class="language-plaintext highlighter-rouge">pthread_cond_wait()</code>, <code class="language-plaintext highlighter-rouge">pthread_mutex_lock</code>, even in <code class="language-plaintext highlighter-rouge">_write</code>, I poll the status of the STW interrupt flag to simulate what the backup thread would do on a real OS.</p>

<p>All of Stdlib compiles, but I only initialise 25 modules, which don’t have extensive OS dependencies.</p>

<ul>
  <li>CamlinternalFormatBasics, Stdlib, Either, Sys, Obj, Type</li>
  <li>Atomic, CamlinternalLazy, Lazy, Seq, Option, Pair, Result</li>
  <li>Bool, Char, Uchar, List, Int, Array, Bytes, String, Unit</li>
  <li>Mutex, Condition, Domain</li>
</ul>

<p>The curry functions are generated at link time by the OCaml linker. I am using Pico SDK linker, <code class="language-plaintext highlighter-rouge">arm-none-eabi-ld</code> and therefore the curry functions are not generated automatically. The workaround was to create a dummy OCaml file that uses enough partial applications to force the generation of <code class="language-plaintext highlighter-rouge">caml_curry2-8</code>, then extract them to assembly, <code class="language-plaintext highlighter-rouge">curry.s</code>, and add that to <code class="language-plaintext highlighter-rouge">libstdlib_pico.a</code> for linking.</p>

<p>As a test, I used the prime number benchmark I used for the original Arm32 work to count the number of prime numbers less than 1 million and compared the single-core and dual-core performance.</p>

<table>
  <thead>
    <tr>
      <th>Test</th>
      <th>Time</th>
      <th>Primes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Single-core</td>
      <td>21,166 ms</td>
      <td>78,498</td>
    </tr>
    <tr>
      <td>Dual-core</td>
      <td>12,350 ms</td>
      <td>78,498</td>
    </tr>
    <tr>
      <td>Speedup</td>
      <td>1.71x</td>
      <td>&nbsp;</td>
    </tr>
  </tbody>
</table>

<p>The code for this project is available in <a href="https://github.com/mtelvers/pico_ocaml">mtelvers/pico_ocaml</a> and <a href="https://github.com/mtelvers/ocaml">mtelvers/ocaml</a>.</p>
