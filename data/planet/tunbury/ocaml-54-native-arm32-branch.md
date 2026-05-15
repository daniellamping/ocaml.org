---
title: OCaml 5.4 native Arm32 branch
description: Recently, I have been using my Pi Zero (armv6), which has reminded me
  that OCaml 5 dropped native 32-bit support, and I wondered what it would take to
  reinstate it.
url: https://www.tunbury.org/2025/11/27/ocaml-54-native/
date: 2025-11-27T22:05:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Recently, I have been using my Pi Zero (armv6), which has reminded me that OCaml 5 dropped native 32-bit support, and I wondered what it would take to reinstate it.</p>

<p>This started as a bit of tinkering; the Pi Zero is slow with a single CPU, 512MB of RAM and SD card storage. Building OCaml 5.4 takes several hours. I’d make a change in the morning, and leave it to build/fail and come back to it the next day.</p>

<p>There was an obvious candidate to revert starting with <a href="https://github.com/ocaml/ocaml/pull/11904">PR#11904 Remove arm, i386 native-code backends</a>. However, OCaml had moved on and cleaned up, so these updates now needed to include Arm32 or reverted:
<a href="https://github.com/ocaml/ocaml/pull/12242">PR#12242 Refactor the computation of stack frame parameters</a>, and
<a href="https://github.com/ocaml/ocaml/pull/12686">PR#12686 Fix the types of C primitives and remove some that are unused</a>,
<a href="https://github.com/ocaml/ocaml/pull/13119">PR#13119 Introduce a platform-independent header for portable CFI/DWARF constructs</a>.</p>

<p>However, this only restored and updated the original Arm32 code, but that code did not implement multicore. Arm64 support was added in <a href="https://github.com/ocaml/ocaml/pull/10972">PR#10972 Arm64 multicore support</a>, and that was the template for the Arm32 implementation.</p>

<p>For debugging, I used small examples, starting with the factorial example on the homepage <a href="https://ocaml.org">ocaml.org</a>, and then working through my <a href="https://github.com/mtelvers/aoc2024">AOC</a> solutions from last year. I compiled these with <code class="language-plaintext highlighter-rouge">ocamlopt</code> and used <code class="language-plaintext highlighter-rouge">gdb</code> on the resulting code rather than trying to debug a segmentation fault in <code class="language-plaintext highlighter-rouge">ocamlopt.opt</code>. Once the compiler was working, I could use the test suite to identify the remaining issues.</p>

<p>The only test I could not get to run was <code class="language-plaintext highlighter-rouge">tests/parallel/max_domains2.ml</code>, which creates 129 domains. Realistically, this test is too large for a 32-bit machine with very limited memory.</p>

<p>I have used a trivial <a href="https://gist.github.com/mtelvers/def18d646a217c3219ba3e54c6d53bec">prime checker</a> as a benchmark, which broadly shows a 3x speed improvement between native code and byte code, and 3x speed improvement in multicore over single core on a quad core machine.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>./ocamlc.opt <span class="nt">-I</span> stdlib <span class="nt">-o</span> bench.byte bench.ml
./ocamlopt.opt <span class="nt">-I</span> stdlib <span class="nt">-o</span> bench.opt bench.ml
hyperfine <span class="s1">'./bench.opt 1'</span> <span class="s1">'./bench.opt 4'</span> <span class="s1">'./bench.byte 1'</span> <span class="s1">'./bench.byte 4'</span>
</code></pre></div></div>

<h4>Raspberry Pi 2 (4 cores, ARMv7)</h4>

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
      <td>1.61s</td>
      <td>10.3x</td>
    </tr>
    <tr>
      <td>Native</td>
      <td>1</td>
      <td>4.79s</td>
      <td>3.5x</td>
    </tr>
    <tr>
      <td>Bytecode</td>
      <td>4</td>
      <td>5.52s</td>
      <td>3.0x</td>
    </tr>
    <tr>
      <td>Bytecode</td>
      <td>1</td>
      <td>16.56s</td>
      <td>1.0x</td>
    </tr>
  </tbody>
</table>

<h4>Raspberry Pi Zero (1 core, ARMv6)</h4>

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
      <td>1</td>
      <td>9.33s</td>
      <td>2.5x</td>
    </tr>
    <tr>
      <td>Native</td>
      <td>4</td>
      <td>9.39s</td>
      <td>2.5x</td>
    </tr>
    <tr>
      <td>Bytecode</td>
      <td>4</td>
      <td>23.25s</td>
      <td>1.0x</td>
    </tr>
    <tr>
      <td>Bytecode</td>
      <td>1</td>
      <td>23.38s</td>
      <td>1.0x</td>
    </tr>
  </tbody>
</table>

<p>I have created a tidy commit history on my fork at <a href="https://github.com/mtelvers/ocaml/commits/arm32-multicore/">arm32-multicore</a>, but the actual path was nowhere near this orderly!</p>

<p>If you have a niche requirement and a spare Pi or other 32-bit Arm and want to have a play:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>git clone https://github.com/mtelvers/ocaml <span class="nt">-b</span> arm32-multicore
<span class="nb">cd </span>ocaml
./configure <span class="o">&amp;&amp;</span> make world.opt <span class="o">&amp;&amp;</span> make tests
</code></pre></div></div>
