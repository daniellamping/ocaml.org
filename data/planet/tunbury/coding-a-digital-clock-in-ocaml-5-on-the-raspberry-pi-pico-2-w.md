---
title: Coding a Digital Clock in OCaml 5 on the Raspberry Pi Pico 2 W
description: While developing a Raspberry Pi GPIO library for the HD44780, mtelvers/gpio,
  I noticed that 8 custom characters could be used to create the elements of a 7-segment
  display. I wanted this clock on the Pi Pico RP2350 dual-core ARM Cortex-M33 using
  my ARM 32 native compiler backend.
url: https://www.tunbury.org/2026/04/07/pico-clock-code/
date: 2026-04-07T21:22:00-00:00
preview_image: https://www.tunbury.org/images/pico-clock-cad.png
authors:
- Mark Elvers
source:
ignore:
---

<p>While developing a Raspberry Pi GPIO library for the HD44780, <a href="https://github.com/mtelvers/gpio">mtelvers/gpio</a>, I noticed that 8 custom characters could be used to create the elements of a 7-segment display. I wanted this clock on the Pi Pico RP2350 dual-core ARM Cortex-M33 using my ARM 32 native compiler backend.</p>

<p>The <a href="https://github.com/mtelvers/pico_ocaml">mtelvers/pico_ocaml</a> project already had OCaml 5 running on the Pico 2 W with WiFi, TCP/IP networking, and <code class="language-plaintext highlighter-rouge">Domain.spawn</code> multicore support. The clock would be a new application building on that foundation.</p>

<h1>The LCD Driver in OCaml</h1>

<p>The HD44780 LCD uses a 4-bit parallel interface over GPIO. Rather than writing the driver in C, I implemented it entirely in OCaml with only the bare minimum GPIO primitives as C stubs:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">external</span> <span class="n">gpio_init</span> <span class="o">:</span> <span class="kt">int</span> <span class="o">-&gt;</span> <span class="kt">unit</span> <span class="o">=</span> <span class="s2">"ocaml_gpio_init"</span>
<span class="k">external</span> <span class="n">gpio_set_dir_out</span> <span class="o">:</span> <span class="kt">int</span> <span class="o">-&gt;</span> <span class="kt">unit</span> <span class="o">=</span> <span class="s2">"ocaml_gpio_set_dir_out"</span>
<span class="k">external</span> <span class="n">gpio_put</span> <span class="o">:</span> <span class="kt">int</span> <span class="o">-&gt;</span> <span class="kt">bool</span> <span class="o">-&gt;</span> <span class="kt">unit</span> <span class="o">=</span> <span class="s2">"ocaml_gpio_put"</span>
<span class="k">external</span> <span class="n">sleep_us</span> <span class="o">:</span> <span class="kt">int</span> <span class="o">-&gt;</span> <span class="kt">unit</span> <span class="o">=</span> <span class="s2">"ocaml_sleep_us"</span>
</code></pre></div></div>

<p>The LCD driver uses an immutable record to hold the pin configuration, returned from <code class="language-plaintext highlighter-rouge">Lcd.init</code> and threaded through all operations:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">type</span> <span class="n">t</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">rs</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span> <span class="n">en</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
  <span class="n">d4</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span> <span class="n">d5</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span> <span class="n">d6</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span> <span class="n">d7</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
  <span class="n">columns</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
<span class="p">}</span>

<span class="k">let</span> <span class="n">pulse_enable</span> <span class="n">t</span> <span class="o">=</span>
  <span class="n">gpio_put</span> <span class="n">t</span><span class="o">.</span><span class="n">en</span> <span class="bp">false</span><span class="p">;</span>
  <span class="n">sleep_us</span> <span class="mi">1</span><span class="p">;</span>
  <span class="n">gpio_put</span> <span class="n">t</span><span class="o">.</span><span class="n">en</span> <span class="bp">true</span><span class="p">;</span>
  <span class="n">sleep_us</span> <span class="mi">1</span><span class="p">;</span>
  <span class="n">gpio_put</span> <span class="n">t</span><span class="o">.</span><span class="n">en</span> <span class="bp">false</span><span class="p">;</span>
  <span class="n">sleep_us</span> <span class="mi">100</span>

<span class="k">let</span> <span class="n">write_4bits</span> <span class="n">t</span> <span class="n">nibble</span> <span class="o">=</span>
  <span class="n">gpio_put</span> <span class="n">t</span><span class="o">.</span><span class="n">d7</span> <span class="p">(</span><span class="n">nibble</span> <span class="ow">land</span> <span class="mi">8</span> <span class="o">&lt;&gt;</span> <span class="mi">0</span><span class="p">);</span>
  <span class="n">gpio_put</span> <span class="n">t</span><span class="o">.</span><span class="n">d6</span> <span class="p">(</span><span class="n">nibble</span> <span class="ow">land</span> <span class="mi">4</span> <span class="o">&lt;&gt;</span> <span class="mi">0</span><span class="p">);</span>
  <span class="n">gpio_put</span> <span class="n">t</span><span class="o">.</span><span class="n">d5</span> <span class="p">(</span><span class="n">nibble</span> <span class="ow">land</span> <span class="mi">2</span> <span class="o">&lt;&gt;</span> <span class="mi">0</span><span class="p">);</span>
  <span class="n">gpio_put</span> <span class="n">t</span><span class="o">.</span><span class="n">d4</span> <span class="p">(</span><span class="n">nibble</span> <span class="ow">land</span> <span class="mi">1</span> <span class="o">&lt;&gt;</span> <span class="mi">0</span><span class="p">);</span>
  <span class="n">pulse_enable</span> <span class="n">t</span>
</code></pre></div></div>

<h1>Dual-Core Architecture</h1>

<p>The final clock uses both Cortex-M33 cores:</p>

<ul>
  <li>Core 0: WiFi connection and periodic NTP time synchronisation</li>
  <li>Core 1: LCD display update loop via <code class="language-plaintext highlighter-rouge">Domain.spawn</code></li>
</ul>

<p>The cores communicate through <code class="language-plaintext highlighter-rouge">Atomic</code> values:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">base_secs</span> <span class="o">=</span> <span class="nn">Atomic</span><span class="p">.</span><span class="n">make</span> <span class="mi">0</span>
<span class="k">let</span> <span class="n">sync_ms</span> <span class="o">=</span> <span class="nn">Atomic</span><span class="p">.</span><span class="n">make</span> <span class="mi">0</span>
</code></pre></div></div>

<p>Core 0 updates these atomically when NTP succeeds. Core 1 reads them each second to compute the current time.</p>

<p>The display loop is a tail-recursive function with a boolean parameter for the blinking colon:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="k">rec</span> <span class="n">display_loop</span> <span class="n">lcd</span> <span class="n">colon</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">bs</span> <span class="o">=</span> <span class="nn">Atomic</span><span class="p">.</span><span class="n">get</span> <span class="n">base_secs</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">sm</span> <span class="o">=</span> <span class="nn">Atomic</span><span class="p">.</span><span class="n">get</span> <span class="n">sync_ms</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">elapsed</span> <span class="o">=</span> <span class="p">(</span><span class="n">time_ms</span> <span class="bp">()</span> <span class="o">-</span> <span class="n">sm</span><span class="p">)</span> <span class="o">/</span> <span class="mi">1000</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">day_secs</span> <span class="o">=</span> <span class="p">(</span><span class="n">bs</span> <span class="o">+</span> <span class="n">elapsed</span><span class="p">)</span> <span class="ow">mod</span> <span class="mi">86400</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">hours</span> <span class="o">=</span> <span class="n">day_secs</span> <span class="o">/</span> <span class="mi">3600</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">minutes</span> <span class="o">=</span> <span class="p">(</span><span class="n">day_secs</span> <span class="ow">mod</span> <span class="mi">3600</span><span class="p">)</span> <span class="o">/</span> <span class="mi">60</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">seconds</span> <span class="o">=</span> <span class="n">day_secs</span> <span class="ow">mod</span> <span class="mi">60</span> <span class="k">in</span>
  <span class="n">display_digit</span> <span class="n">lcd</span>  <span class="mi">2</span> <span class="p">(</span><span class="n">hours</span> <span class="o">/</span> <span class="mi">10</span><span class="p">);</span>
  <span class="n">display_digit</span> <span class="n">lcd</span>  <span class="mi">6</span> <span class="p">(</span><span class="n">hours</span> <span class="ow">mod</span> <span class="mi">10</span><span class="p">);</span>
  <span class="n">display_colon</span> <span class="n">lcd</span>  <span class="mi">9</span> <span class="n">colon</span><span class="p">;</span>
  <span class="c">(* ... seconds display ... *)</span>
  <span class="n">sleep_ms</span> <span class="mi">1000</span><span class="p">;</span>
  <span class="n">display_loop</span> <span class="n">lcd</span> <span class="p">(</span><span class="n">not</span> <span class="n">colon</span><span class="p">)</span>
</code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">not colon</code> tail call compiles to a simple jump with an unboxed boolean.</p>

<h1>Memory: Working Within Constraints</h1>

<p>The Pico 2 W has roughly 520KB of RAM, with about 428KB available for the OCaml heap after the runtime, WiFi driver, and lwIP stack. The OCaml 5 multicore runtime needs approximately 36KB for a second domain (minor heap, stack, metadata).</p>

<p>Early in development, memory was much tighter (~413KB available) because the clock was compiling the full <code class="language-plaintext highlighter-rouge">net.ml</code> effects-based networking module despite only needing raw UDP for NTP. Splitting the raw network stubs into a lightweight <code class="language-plaintext highlighter-rouge">netif.ml</code> module freed 13KB of heap, which was enough for <code class="language-plaintext highlighter-rouge">Domain.spawn</code> to succeed without any explicit <code class="language-plaintext highlighter-rouge">Gc.compact()</code> or <code class="language-plaintext highlighter-rouge">Gc.full_major()</code> calls. The final code has zero GC workarounds.</p>

<h2>Flattened Data Structures</h2>

<p>In OCaml, each nested array is a separate heap block that goes into a size-class pool (8KB each with our optimised pool size). Nested arrays for 10 digits would create 51 small blocks across multiple size classes, potentially consuming 24KB+ in pool overhead.</p>

<p>Instead, the digit patterns are stored as a single flat array:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* 10 digits x 4 rows x 3 cols = 120 entries *)</span>
<span class="k">let</span> <span class="n">digits</span> <span class="o">=</span> <span class="p">[</span><span class="o">|</span>
  <span class="mi">1</span><span class="p">;</span><span class="mi">0</span><span class="p">;</span><span class="mi">7</span><span class="p">;</span> <span class="mi">2</span><span class="p">;</span><span class="mi">8</span><span class="p">;</span><span class="mi">6</span><span class="p">;</span> <span class="mi">2</span><span class="p">;</span><span class="mi">8</span><span class="p">;</span><span class="mi">6</span><span class="p">;</span> <span class="mi">3</span><span class="p">;</span><span class="mi">4</span><span class="p">;</span><span class="mi">5</span><span class="p">;</span>   <span class="c">(* 0 *)</span>
  <span class="mi">8</span><span class="p">;</span><span class="mi">8</span><span class="p">;</span><span class="mi">7</span><span class="p">;</span> <span class="mi">8</span><span class="p">;</span><span class="mi">8</span><span class="p">;</span><span class="mi">6</span><span class="p">;</span> <span class="mi">8</span><span class="p">;</span><span class="mi">8</span><span class="p">;</span><span class="mi">6</span><span class="p">;</span> <span class="mi">8</span><span class="p">;</span><span class="mi">8</span><span class="p">;</span><span class="mi">6</span><span class="p">;</span>   <span class="c">(* 1 *)</span>
  <span class="o">...</span>
<span class="o">|</span><span class="p">]</span>

<span class="k">let</span> <span class="n">display_digit</span> <span class="n">lcd</span> <span class="n">col</span> <span class="n">digit</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">base</span> <span class="o">=</span> <span class="n">digit</span> <span class="o">*</span> <span class="mi">12</span> <span class="k">in</span>
  <span class="k">for</span> <span class="n">row</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">to</span> <span class="mi">3</span> <span class="k">do</span>
    <span class="nn">Lcd</span><span class="p">.</span><span class="n">move_to</span> <span class="n">lcd</span> <span class="n">col</span> <span class="n">row</span><span class="p">;</span>
    <span class="k">for</span> <span class="n">c</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">to</span> <span class="mi">2</span> <span class="k">do</span>
      <span class="k">let</span> <span class="n">seg</span> <span class="o">=</span> <span class="n">digits</span><span class="o">.</span><span class="p">(</span><span class="n">base</span> <span class="o">+</span> <span class="n">row</span> <span class="o">*</span> <span class="mi">3</span> <span class="o">+</span> <span class="n">c</span><span class="p">)</span> <span class="k">in</span>
      <span class="o">...</span>
</code></pre></div></div>

<h1>NTP Time Sync</h1>

<p>The clock fetches time via NTP over UDP using raw C stubs directly without the effect handlers I had developed before. This was a deliberate design choice as the effects-based <code class="language-plaintext highlighter-rouge">Net.run</code> handler allocates closures and a 4KB receive buffer on every call, while the raw path uses a 48-byte buffer (the exact NTP packet size) and creates no closures.</p>

<p>The NTP timestamp (4 bytes at offset 40 in the response) represents seconds since 1900. Since a full NTP timestamp exceeds OCaml’s 31-bit integer on 32-bit ARM, I compute seconds-of-day using modular arithmetic that stays in range:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* 2^24 mod 86400 = 15616, 2^16 mod 86400 = 65536, 2^8 mod 86400 = 256 *)</span>
<span class="k">let</span> <span class="n">raw</span> <span class="o">=</span> <span class="n">b0</span> <span class="o">*</span> <span class="mi">15616</span> <span class="o">+</span> <span class="n">b1</span> <span class="o">*</span> <span class="mi">65536</span> <span class="o">+</span> <span class="n">b2</span> <span class="o">*</span> <span class="mi">256</span> <span class="o">+</span> <span class="n">b3</span> <span class="k">in</span>
<span class="k">let</span> <span class="n">secs_of_day</span> <span class="o">=</span> <span class="n">raw</span> <span class="ow">mod</span> <span class="mi">86400</span> <span class="k">in</span>
</code></pre></div></div>

<h2>Network Polling</h2>

<p>NTP queries were strangely unreliable which was traced to the requirement for Core 0 to continue polling the lwIP network stack rather than just sleep. The CYW43 WiFi driver in polling mode needs regular <code class="language-plaintext highlighter-rouge">cyw43_arch_poll()</code> calls to maintain the connection. Without this, NTP success rates dropped below 50%. The code now polls every 100ms during the wait interval instead of a solid sleep:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">for</span> <span class="n">_</span> <span class="o">=</span> <span class="mi">1</span> <span class="k">to</span> <span class="n">resync_interval_ms</span> <span class="o">/</span> <span class="mi">100</span> <span class="k">do</span>
  <span class="nn">Netif</span><span class="p">.</span><span class="n">service_network</span> <span class="bp">()</span><span class="p">;</span>
  <span class="n">sleep_ms</span> <span class="mi">100</span>
<span class="k">done</span>
</code></pre></div></div>

<h2>PWM Slice Conflict</h2>

<p>The backlight PWM initially used GPIO 22, which shares PWM slice 11 with GPIO 23 which turned out to be the CYW43 WiFi chip’s power control pin. Calling <code class="language-plaintext highlighter-rouge">pwm_set_duty</code> on GPIO 22 disrupted the WiFi connection, resulting in 100% NTP failure. Moving the backlight to GPIO 27 (PWM slice 13, no CYW43 conflict) resolved it. On the Pico W boards, the CYW43 pins create hidden hardware constraints.</p>

<h1>Cross-Compilation Bugs in the OCaml Compiler</h1>

<p>When I first tried using <code class="language-plaintext highlighter-rouge">/</code> and <code class="language-plaintext highlighter-rouge">mod</code> operators, the program crashed immediately. The assembler warned:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>rdhi and rdlo must be different
</code></pre></div></div>

<p>OCaml compiles division by constants using a “multiply by magic reciprocal” approach. In my ARMv8-M backend (which lacks the <code class="language-plaintext highlighter-rouge">smmul</code> instruction available on ARMv6+), it falls back to <code class="language-plaintext highlighter-rouge">smull</code> (signed multiply long). The <code class="language-plaintext highlighter-rouge">smull</code> instruction writes a 64-bit result to two registers (rdlo and rdhi), and the ARM specification requires these to be different registers. The register allocator was sometimes assigning the same register (<code class="language-plaintext highlighter-rouge">r12</code>) to both.</p>

<p>My first fix attempt added register constraints to the ARM backend’s instruction selection, forcing specific physical registers for <code class="language-plaintext highlighter-rouge">smull</code> operands. The assembler warnings disappeared, but division now produced wrong results, for example, <code class="language-plaintext highlighter-rouge">65343 / 3600</code> returned 26 instead of 18.</p>

<p>Eventually, I tested the magic constant itself:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">&gt;&gt;&gt;</span> <span class="n">n</span> <span class="o">=</span> <span class="mi">65343</span>
<span class="o">&gt;&gt;&gt;</span> <span class="n">M</span> <span class="o">=</span> <span class="mh">0x6AF37C05</span>  <span class="c1"># magic constant from disassembly
</span><span class="o">&gt;&gt;&gt;</span> <span class="p">(</span><span class="n">n</span> <span class="o">*</span> <span class="n">M</span><span class="p">)</span> <span class="o">&gt;&gt;</span> <span class="mi">42</span>
<span class="mi">26</span>  <span class="c1"># Wrong! Should be 18
</span></code></pre></div></div>

<p>The magic constant was wrong! The function <code class="language-plaintext highlighter-rouge">divimm_parameters</code> in <code class="language-plaintext highlighter-rouge">cmm_helpers.ml</code> computes these constants using the Hacker’s Delight algorithm with OCaml’s <code class="language-plaintext highlighter-rouge">Nativeint</code> module. On the 64-bit host, <code class="language-plaintext highlighter-rouge">Nativeint</code> wraps at 2^64, but the 32-bit target needs constants computed with wrapping at 2^32.</p>

<p>The OCaml compiler has a module designed for exactly this purpose: <code class="language-plaintext highlighter-rouge">Targetint</code> in <code class="language-plaintext highlighter-rouge">utils/targetint.ml</code>. Its documentation says it provides “signed 32-bit integers (on 32-bit target platforms) or signed 64-bit integers (on 64-bit target platforms).” But line 64 tells the real story:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">size</span> <span class="o">=</span> <span class="nn">Sys</span><span class="p">.</span><span class="n">word_size</span>
<span class="c">(* Later, this will be set by the configure script
   in order to support cross-compilation. *)</span>
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">Targetint</code> uses <code class="language-plaintext highlighter-rouge">Sys.word_size</code>, which is the host’s word size. The TODO comment acknowledges that the intention is to fix this for cross-compilation, but it hasn’t been. On my 64-bit Raspberry Pi 5 host, <code class="language-plaintext highlighter-rouge">Targetint.size = 64</code> even when targeting 32-bit ARM.</p>

<p>This is a general cross-compilation bug, not ARM-specific, but it doesn’t manifest because there aren’t any 32-bit backends in the upstream branch. It would affect any configuration where the host word size differs from the target: 64-to-32 for me, and hypothetically 128-to-64 in the future. Division by runtime variables is unaffected as it uses a C library call (<code class="language-plaintext highlighter-rouge">__aeabi_idivmod</code> on ARM). Only division by compile-time constants triggers the multiply-by-reciprocal path.</p>

<p>To fix the issue, I need to set <code class="language-plaintext highlighter-rouge">Targetint</code> to the correct value and then use it throughout.</p>

<p>First, expose the target’s word width from <code class="language-plaintext highlighter-rouge">./configure</code>. The build system already computes <code class="language-plaintext highlighter-rouge">arch64</code> (true for 64-bit targets, false for 32-bit) and writes it to <code class="language-plaintext highlighter-rouge">Makefile.config</code>. I added it to <code class="language-plaintext highlighter-rouge">Config</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* utils/config.generated.ml.in *)</span>
<span class="k">let</span> <span class="n">arch64</span> <span class="o">=</span> <span class="o">@</span><span class="n">arch64</span><span class="o">@</span>
</code></pre></div></div>

<p>Then <code class="language-plaintext highlighter-rouge">Targetint</code> uses it instead of the host’s <code class="language-plaintext highlighter-rouge">Sys.word_size</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* utils/targetint.ml *)</span>
<span class="k">let</span> <span class="n">size</span> <span class="o">=</span> <span class="k">if</span> <span class="nn">Config</span><span class="p">.</span><span class="n">arch64</span> <span class="k">then</span> <span class="mi">64</span> <span class="k">else</span> <span class="mi">32</span>
</code></pre></div></div>

<p>With <code class="language-plaintext highlighter-rouge">Targetint</code> now correct, <code class="language-plaintext highlighter-rouge">divimm_parameters</code> is rewritten from <code class="language-plaintext highlighter-rouge">Nativeint</code> to <code class="language-plaintext highlighter-rouge">Targetint</code>. The algorithm is unchanged; only the module is swapped:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">divimm_parameters</span> <span class="n">d</span> <span class="o">=</span> <span class="nn">Targetint</span><span class="p">.(</span>
  <span class="k">let</span> <span class="n">twopsm1</span> <span class="o">=</span> <span class="n">min_int</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">nc</span> <span class="o">=</span> <span class="n">sub</span> <span class="p">(</span><span class="n">pred</span> <span class="n">twopsm1</span><span class="p">)</span> <span class="p">(</span><span class="n">unsigned_rem</span> <span class="n">twopsm1</span> <span class="n">d</span><span class="p">)</span> <span class="k">in</span>
  <span class="k">let</span> <span class="k">rec</span> <span class="n">loop</span> <span class="n">p</span> <span class="p">(</span><span class="n">q1</span><span class="o">,</span> <span class="n">r1</span><span class="p">)</span> <span class="p">(</span><span class="n">q2</span><span class="o">,</span> <span class="n">r2</span><span class="p">)</span> <span class="o">=</span>
    <span class="o">...</span>
  <span class="k">in</span> <span class="o">...</span><span class="p">)</span>
</code></pre></div></div>

<p>The sign check and sign-bit extraction in <code class="language-plaintext highlighter-rouge">div_int</code> similarly switch from <code class="language-plaintext highlighter-rouge">Nativeint</code> to <code class="language-plaintext highlighter-rouge">Targetint</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">m_neg</span> <span class="o">=</span> <span class="nn">Targetint</span><span class="p">.(</span><span class="n">compare</span> <span class="p">(</span><span class="n">of_int</span> <span class="p">(</span><span class="nn">Nativeint</span><span class="p">.</span><span class="n">to_int</span> <span class="n">m</span><span class="p">))</span> <span class="n">zero</span><span class="p">)</span> <span class="o">&lt;</span> <span class="mi">0</span> <span class="k">in</span>
<span class="o">...</span>
<span class="n">add_int</span> <span class="n">t</span> <span class="p">(</span><span class="n">lsr_int</span> <span class="n">c1</span> <span class="p">(</span><span class="nc">Cconst_int</span> <span class="p">(</span><span class="nn">Targetint</span><span class="p">.</span><span class="n">size</span> <span class="o">-</span> <span class="mi">1</span><span class="o">,</span> <span class="n">dbg</span><span class="p">))</span> <span class="n">dbg</span><span class="p">)</span> <span class="n">dbg</span><span class="p">)</span>
</code></pre></div></div>

<h1>ARMv8-M smull Register Collision</h1>

<p>In a separate, ARM-specific bug in my backend, the <code class="language-plaintext highlighter-rouge">smull</code> instruction emitted for ARMv8-M hardcoded <code class="language-plaintext highlighter-rouge">r12</code> as the low result register; however, the register allocator can also assign <code class="language-plaintext highlighter-rouge">r12</code> as the high result register. The fix in <code class="language-plaintext highlighter-rouge">emit.mlp</code> swaps to <code class="language-plaintext highlighter-rouge">r3</code> when a collision would occur:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">rdlo</span> <span class="o">=</span> <span class="k">if</span> <span class="n">i</span><span class="o">.</span><span class="n">res</span><span class="o">.</span><span class="p">(</span><span class="mi">0</span><span class="p">)</span><span class="o">.</span><span class="n">loc</span> <span class="o">=</span> <span class="nc">Reg</span> <span class="mi">8</span> <span class="k">then</span> <span class="s2">"r3"</span> <span class="k">else</span> <span class="s2">"r12"</span> <span class="k">in</span>
</code></pre></div></div>

<h1>It works!</h1>

<p>A dual-core clock is a crazy project! I’m pleased to have done it, though. The limited memory on the Pico makes using it a constant challenge, but I managed with a single <code class="language-plaintext highlighter-rouge">Gc.compact()</code> before <code class="language-plaintext highlighter-rouge">Domain.spawn</code> to ensure the heap was defragmented for the second domain’s allocation.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>=== OCaml Digital Clock ===
  Core 0: NTP sync
  Core 1: LCD display

Connecting to WiFi...
IP: 192.168.1.41
Display running on Core 1
NTP sync: 22:32:53
NTP sync: 22:33:54
...
</code></pre></div></div>

<p>The full source is at <a href="https://github.com/mtelvers/pico_ocaml">github.com/mtelvers/pico_ocaml</a>. The compiler fix has been committed to my <a href="https://github.com/mtelvers/ocaml/tree/arm32-multicore">arm32-multicore</a> branch.</p>
