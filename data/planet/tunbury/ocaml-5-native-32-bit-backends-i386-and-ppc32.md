---
title: 'OCaml 5 native 32-bit backends: i386 and PPC32'
description: 'Following on from the Arm32 multicore backend, I have now ported the
  remaining two 32-bit architectures to OCaml 5 with multicore support: i386 and PowerPC
  32-bit (PPC32).'
url: https://www.tunbury.org/2026/03/03/32bit-backends/
date: 2026-03-03T14:30:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Following on from the <a href="https://www.tunbury.org/2025/11/27/ocaml-54-native/">Arm32 multicore backend</a>, I have now ported the remaining two 32-bit architectures to OCaml 5 with multicore support: i386 and PowerPC 32-bit (PPC32).</p>

<p>OCaml 5’s multicore runtime needs a per-domain state: the allocation pointer, exception handler, GC data and so on. On 64-bit platforms, there are registers to spare, but on 32-bit architectures, particularly i386, there are far fewer, and I want to retain the shared nature of the ppc64/ppc32 backend, which caused more problems.</p>

<h1>Design choices</h1>

<h2>i386: Thread-local storage via %gs</h2>

<p>The i386 architecture has only 7 general-purpose registers. I initially tried dedicating one to the domain state pointer, but with only 6 remaining registers, the graph colouring register allocator could not find a valid allocation for many programs. Instead, the i386 backend uses the <code class="language-plaintext highlighter-rouge">%gs</code> segment register to access thread-local storage (TLS). Every time a domain state is needed, the compiler emits:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>movl %gs:caml_state@ntpoff, %ebx
</code></pre></div></div>

<p>This loads the domain state pointer on demand from the thread-local <code class="language-plaintext highlighter-rouge">caml_state</code> variable. It costs an extra instruction per access but keeps all general-purpose registers available for allocation. The <code class="language-plaintext highlighter-rouge">@ntpoff</code> relocation uses the local-exec TLS model, which is the fastest TLS access pattern on Linux. This mechanism is Linux/ELF-specific; on Windows, <code class="language-plaintext highlighter-rouge">%fs</code> is reserved for the Thread Information Block and <code class="language-plaintext highlighter-rouge">%gs</code> is not available for TLS in the same way, so a Windows port would need a different approach.</p>

<h2>PPC32: Dedicated register r30</h2>

<p>PPC32 has 32 general-purpose registers, so dedicating one is affordable. Register r30 permanently holds the domain state pointer (<code class="language-plaintext highlighter-rouge">DOMAIN_STATE_PTR</code>), matching the approach used by Arm32 and the existing PPC64 backend. The allocation pointer lives in r31, and the exception handler pointer in r29. The PPC32 and PPC64 backends share the same source files (<code class="language-plaintext highlighter-rouge">emit.mlp</code>, <code class="language-plaintext highlighter-rouge">proc.ml</code>, <code class="language-plaintext highlighter-rouge">power.S</code>) with conditionals for the two modes, so keeping the same register assignments avoids divergence in shared code.</p>

<p>However, there were some challenges with position-independent code (PIC). On PPC32, calls to shared library functions go through the PLT (Procedure Linkage Table), and the PLT stubs use the GOT (Global Offset Table) to find the actual function addresses at runtime. The standard PPC32 secure-PLT convention uses r30 as the GOT base pointer, which conflicts directly with its use as <code class="language-plaintext highlighter-rouge">DOMAIN_STATE_PTR</code>. The solution was to bypass PLT stubs entirely, using a per-compilation-unit <code class="language-plaintext highlighter-rouge">.got2</code> section with PC-relative addressing for all external symbol references. This avoids the system GOT (which can overflow its 16-bit offset limit in large programs) and keeps r30 free for OCaml’s use.</p>

<p>Another interesting thing to note is that the PPC <code class="language-plaintext highlighter-rouge">bltl-</code> instruction used for allocation checks unconditionally clobbers the link register (LR) regardless of whether the branch is taken. This is per the PPC ISA specification (LR is set when LK=1), which means LR must be saved and restored in every function that has a stack frame, not just those that make explicit calls.</p>

<h1>Benchmarks</h1>

<p>Both backends were tested under QEMU using a <a href="https://gist.github.com/mtelvers/def18d646a217c3219ba3e54c6d53bec">trivial prime counter</a> as a benchmark as I used for Arm32.</p>

<h2>i386 (QEMU, 4 vCPUs)</h2>

<table>
  <thead>
    <tr>
      <th>Mode</th>
      <th>Domains</th>
      <th>Time</th>
      <th>Speedup vs slowest</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Native</td>
      <td>4</td>
      <td>0.17s</td>
      <td>6.9x</td>
    </tr>
    <tr>
      <td>Native</td>
      <td>2</td>
      <td>0.35s</td>
      <td>3.4x</td>
    </tr>
    <tr>
      <td>Native</td>
      <td>1</td>
      <td>0.46s</td>
      <td>2.6x</td>
    </tr>
    <tr>
      <td>Bytecode</td>
      <td>4</td>
      <td>0.50s</td>
      <td>2.4x</td>
    </tr>
    <tr>
      <td>Bytecode</td>
      <td>2</td>
      <td>0.69s</td>
      <td>1.7x</td>
    </tr>
    <tr>
      <td>Bytecode</td>
      <td>1</td>
      <td>1.18s</td>
      <td>1.0x</td>
    </tr>
  </tbody>
</table>

<h2>PPC32 (QEMU, 1 vCPU)</h2>

<table>
  <thead>
    <tr>
      <th>Mode</th>
      <th>Domains</th>
      <th>Time</th>
      <th>Speedup vs slowest</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Native</td>
      <td>2</td>
      <td>2.02s</td>
      <td>9.5x</td>
    </tr>
    <tr>
      <td>Native</td>
      <td>4</td>
      <td>2.09s</td>
      <td>9.2x</td>
    </tr>
    <tr>
      <td>Native</td>
      <td>1</td>
      <td>2.28s</td>
      <td>8.5x</td>
    </tr>
    <tr>
      <td>Bytecode</td>
      <td>1</td>
      <td>19.28s</td>
      <td>1.0x</td>
    </tr>
    <tr>
      <td>Bytecode</td>
      <td>4</td>
      <td>19.96s</td>
      <td>1.0x</td>
    </tr>
    <tr>
      <td>Bytecode</td>
      <td>2</td>
      <td>20.53s</td>
      <td>0.9x</td>
    </tr>
  </tbody>
</table>

<p>The i386 results show real multicore scaling: native code with 4 domains is 2.7x faster than single-domain, and nearly 7x faster than single-domain bytecode. The PPC32 machine only has a single emulated CPU, so there is no multicore scaling, but the native backend is consistently 8-10x faster than bytecode. QEMU’s <code class="language-plaintext highlighter-rouge">mac99</code> machine does not support SMP, so testing true PPC32 parallelism will need either real hardware or a different emulation platform.</p>

<h1>Test suites</h1>

<p>Both backends pass the OCaml test suite with only bytecode-related exceptions. On PPC32, the two failing tests (<code class="language-plaintext highlighter-rouge">lazy7</code> and <code class="language-plaintext highlighter-rouge">test_compact_manydomains</code>) both fail only in bytecode mode; the native backend passes everything.</p>

<h1>Try it</h1>

<p>Both backends are available on my fork:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>git clone https://github.com/mtelvers/ocaml -b arm32-multicore
cd ocaml
./configure &amp;&amp; make world.opt &amp;&amp; make tests
</code></pre></div></div>

<p>The branch now supports Arm32, i386, and PPC32 architectures.</p>
