---
title: Base Fibonacci
description: "In Numberphile\u2019s latest video, Tony Padilla does a \u2018magic
  trick\u2019 with Fibonacci numbers and talks about Zeckendorf decompositions, and
  I had my laptop out even before the video ended."
url: https://www.tunbury.org/2026/01/11/base-fibonacci/
date: 2026-01-11T21:00:00-00:00
preview_image: https://www.tunbury.org/images/base-fibonacci.jpg
authors:
- Mark Elvers
source:
ignore:
---

<p>In Numberphile’s latest <a href="https://www.youtube.com/watch?v=S5FTe5KP2Cw">video</a>, Tony Padilla does a ‘magic trick’ with Fibonacci numbers and talks about Zeckendorf decompositions, and I had my laptop out even before the video ended.</p>

<p>As a summary of the video, a player is asked to pick a number up to a maximum, and mark down which rows of a table their number appears in.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>1,4,6,9,12
2,7,10
3,4,11,12
5,6,7
8,9,10,11,12
</code></pre></div></div>

<p>Let’s say I picked seven as my number; it appears in row 2 and row 4. Then, I can <em>magically</em> work out the original number by adding together the first two numbers in the row. 5 + 2 = 7. The first number in each row of the table is a Fibonacci number.</p>

<p>All numbers are the sum of one or more Fibonacci numbers, and there are typically multiple solutions. However, the Zeckendorf decomposition gives a unique solution by greedily subtracting the largest possible Fibonacci number. Let’s see that in OCaml.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">to_zeckendorf</span> <span class="n">n</span> <span class="o">=</span>
  <span class="k">let</span> <span class="k">rec</span> <span class="n">fibs</span> <span class="n">a</span> <span class="n">b</span> <span class="n">acc</span> <span class="o">=</span>
    <span class="k">if</span> <span class="n">a</span> <span class="o">&gt;</span> <span class="n">n</span> <span class="k">then</span> <span class="n">acc</span> <span class="k">else</span> <span class="n">fibs</span> <span class="n">b</span> <span class="p">(</span><span class="n">a</span> <span class="o">+</span> <span class="n">b</span><span class="p">)</span> <span class="p">(</span><span class="n">a</span> <span class="o">::</span> <span class="n">acc</span><span class="p">)</span>
  <span class="k">in</span>
  <span class="k">let</span> <span class="n">fib_list</span> <span class="o">=</span> <span class="n">fibs</span> <span class="mi">1</span> <span class="mi">2</span> <span class="bp">[]</span> <span class="k">in</span>
  
  <span class="k">let</span> <span class="k">rec</span> <span class="n">convert</span> <span class="n">remaining</span> <span class="n">fibs</span> <span class="n">acc</span> <span class="o">=</span>
    <span class="k">match</span> <span class="n">fibs</span> <span class="k">with</span>
    <span class="o">|</span> <span class="bp">[]</span> <span class="o">-&gt;</span> <span class="nn">List</span><span class="p">.</span><span class="n">rev</span> <span class="n">acc</span>
    <span class="o">|</span> <span class="n">f</span> <span class="o">::</span> <span class="n">rest</span> <span class="o">-&gt;</span>
        <span class="k">if</span> <span class="n">f</span> <span class="o">&lt;=</span> <span class="n">remaining</span> <span class="k">then</span> <span class="n">convert</span> <span class="p">(</span><span class="n">remaining</span> <span class="o">-</span> <span class="n">f</span><span class="p">)</span> <span class="n">rest</span> <span class="p">(</span><span class="mi">1</span> <span class="o">::</span> <span class="n">acc</span><span class="p">)</span>
        <span class="k">else</span> <span class="n">convert</span> <span class="n">remaining</span> <span class="n">rest</span> <span class="p">(</span><span class="mi">0</span> <span class="o">::</span> <span class="n">acc</span><span class="p">)</span>
  <span class="k">in</span>
  <span class="n">convert</span> <span class="n">n</span> <span class="n">fib_list</span> <span class="bp">[]</span>

<span class="k">let</span> <span class="n">zeck_to_string</span> <span class="n">bits</span> <span class="o">=</span>
  <span class="n">bits</span> <span class="o">|&gt;</span> <span class="nn">List</span><span class="p">.</span><span class="n">map</span> <span class="n">string_of_int</span> <span class="o">|&gt;</span> <span class="nn">String</span><span class="p">.</span><span class="n">concat</span> <span class="s2">""</span>
</code></pre></div></div>

<p>Resulting in this binary-ish string representation:</p>
<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code># zeck_to_string (to_zeckendorf 7);;
- : string = "1010"
</code></pre></div></div>

<p>What we really want, though, is the original table so we can play the game with our friends with even larger numbers.</p>

<p>The simplest approach may be to count up while generating the Fibonacci sequence. This looks reasonably efficient. The <code class="language-plaintext highlighter-rouge">max_fibs</code> constant isn’t a big constraint, as the 94th Fibonacci number is the largest which can be represented in an unsigned 64-bit integer, so we will run out of system resources long before that’s an issue.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">fib_table</span> <span class="n">hi</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">max_fibs</span> <span class="o">=</span> <span class="mi">94</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">fibs</span> <span class="o">=</span> <span class="nn">Array</span><span class="p">.</span><span class="n">make</span> <span class="n">max_fibs</span> <span class="mi">0</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">buckets</span> <span class="o">=</span> <span class="nn">Array</span><span class="p">.</span><span class="n">make</span> <span class="n">max_fibs</span> <span class="bp">[]</span> <span class="k">in</span>

  <span class="n">fibs</span><span class="o">.</span><span class="p">(</span><span class="mi">0</span><span class="p">)</span> <span class="o">&lt;-</span> <span class="mi">1</span><span class="p">;</span>
  <span class="n">fibs</span><span class="o">.</span><span class="p">(</span><span class="mi">1</span><span class="p">)</span> <span class="o">&lt;-</span> <span class="mi">2</span><span class="p">;</span>

  <span class="k">let</span> <span class="k">rec</span> <span class="n">decompose</span> <span class="n">orig</span> <span class="n">remaining</span> <span class="n">i</span> <span class="o">=</span>
    <span class="k">if</span> <span class="n">i</span> <span class="o">&lt;</span> <span class="mi">0</span> <span class="k">then</span> <span class="bp">()</span>
    <span class="k">else</span> <span class="k">if</span> <span class="n">fibs</span><span class="o">.</span><span class="p">(</span><span class="n">i</span><span class="p">)</span> <span class="o">&lt;=</span> <span class="n">remaining</span> <span class="k">then</span> <span class="p">(</span>
      <span class="n">buckets</span><span class="o">.</span><span class="p">(</span><span class="n">i</span><span class="p">)</span> <span class="o">&lt;-</span> <span class="n">orig</span> <span class="o">::</span> <span class="n">buckets</span><span class="o">.</span><span class="p">(</span><span class="n">i</span><span class="p">);</span>
      <span class="n">decompose</span> <span class="n">orig</span> <span class="p">(</span><span class="n">remaining</span> <span class="o">-</span> <span class="n">fibs</span><span class="o">.</span><span class="p">(</span><span class="n">i</span><span class="p">))</span> <span class="p">(</span><span class="n">i</span> <span class="o">-</span> <span class="mi">1</span><span class="p">)</span>
    <span class="p">)</span> <span class="k">else</span>
      <span class="n">decompose</span> <span class="n">orig</span> <span class="n">remaining</span> <span class="p">(</span><span class="n">i</span> <span class="o">-</span> <span class="mi">1</span><span class="p">)</span>
  <span class="k">in</span>

  <span class="k">let</span> <span class="k">rec</span> <span class="n">go</span> <span class="n">n</span> <span class="n">num_fibs</span> <span class="o">=</span>
    <span class="k">if</span> <span class="n">n</span> <span class="o">&gt;</span> <span class="n">hi</span> <span class="k">then</span> <span class="n">num_fibs</span>
    <span class="k">else</span>
      <span class="k">let</span> <span class="n">next</span> <span class="o">=</span> <span class="n">fibs</span><span class="o">.</span><span class="p">(</span><span class="n">num_fibs</span> <span class="o">-</span> <span class="mi">1</span><span class="p">)</span> <span class="o">+</span> <span class="n">fibs</span><span class="o">.</span><span class="p">(</span><span class="n">num_fibs</span> <span class="o">-</span> <span class="mi">2</span><span class="p">)</span> <span class="k">in</span>
      <span class="k">if</span> <span class="n">n</span> <span class="o">&gt;=</span> <span class="n">next</span> <span class="k">then</span> <span class="p">(</span>
        <span class="n">fibs</span><span class="o">.</span><span class="p">(</span><span class="n">num_fibs</span><span class="p">)</span> <span class="o">&lt;-</span> <span class="n">next</span><span class="p">;</span>
        <span class="n">decompose</span> <span class="n">n</span> <span class="n">n</span> <span class="n">num_fibs</span><span class="p">;</span>
        <span class="n">go</span> <span class="p">(</span><span class="n">n</span> <span class="o">+</span> <span class="mi">1</span><span class="p">)</span> <span class="p">(</span><span class="n">num_fibs</span> <span class="o">+</span> <span class="mi">1</span><span class="p">)</span>
      <span class="p">)</span> <span class="k">else</span> <span class="p">(</span>
        <span class="n">decompose</span> <span class="n">n</span> <span class="n">n</span> <span class="p">(</span><span class="n">num_fibs</span> <span class="o">-</span> <span class="mi">1</span><span class="p">);</span>
        <span class="n">go</span> <span class="p">(</span><span class="n">n</span> <span class="o">+</span> <span class="mi">1</span><span class="p">)</span> <span class="n">num_fibs</span>
      <span class="p">)</span>
  <span class="k">in</span>

  <span class="k">let</span> <span class="n">num_fibs</span> <span class="o">=</span> <span class="n">go</span> <span class="mi">1</span> <span class="mi">2</span> <span class="k">in</span>
  <span class="nn">Array</span><span class="p">.</span><span class="n">init</span> <span class="n">num_fibs</span> <span class="p">(</span><span class="k">fun</span> <span class="n">i</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="n">fibs</span><span class="o">.</span><span class="p">(</span><span class="n">i</span><span class="p">)</span><span class="o">,</span> <span class="nn">List</span><span class="p">.</span><span class="n">rev</span> <span class="n">buckets</span><span class="o">.</span><span class="p">(</span><span class="n">i</span><span class="p">)))</span>
  <span class="o">|&gt;</span> <span class="nn">Array</span><span class="p">.</span><span class="n">to_list</span>
</code></pre></div></div>

<p>Here is the resulting table.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code># fib_table 100;;
- : (int * int list) list =
[(1, [1; 4; 6; 9; 12; 14; 17; 19; 22; 25; 27; 30; 33; 35; 38; 40; 43; 46; 48; 51; 53; 56; 59; 61; 64; 67; 69; 72; 74; 77; 80; 82; 85; 88; 90; 93; 95; 98]);
 (2, [2; 7; 10; 15; 20; 23; 28; 31; 36; 41; 44; 49; 54; 57; 62; 65; 70; 75; 78; 83; 86; 91; 96; 99]);
 (3, [3; 4; 11; 12; 16; 17; 24; 25; 32; 33; 37; 38; 45; 46; 50; 51; 58; 59; 66; 67; 71; 72; 79; 80; 87; 88; 92; 93; 100]);
 (5, [5; 6; 7; 18; 19; 20; 26; 27; 28; 39; 40; 41; 52; 53; 54; 60; 61; 62; 73; 74; 75; 81; 82; 83; 94; 95; 96]);
 (8, [8; 9; 10; 11; 12; 29; 30; 31; 32; 33; 42; 43; 44; 45; 46; 63; 64; 65; 66; 67; 84; 85; 86; 87; 88; 97; 98; 99; 100]);
 (13, [13; 14; 15; 16; 17; 18; 19; 20; 47; 48; 49; 50; 51; 52; 53; 54; 68; 69; 70; 71; 72; 73; 74; 75]);
 (21, [21; 22; 23; 24; 25; 26; 27; 28; 29; 30; 31; 32; 33; 76; 77; 78; 79; 80; 81; 82; 83; 84; 85; 86; 87; 88]);
 (34, [34; 35; 36; 37; 38; 39; 40; 41; 42; 43; 44; 45; 46; 47; 48; 49; 50; 51; 52; 53; 54]);
 (55, [55; 56; 57; 58; 59; 60; 61; 62; 63; 64; 65; 66; 67; 68; 69; 70; 71; 72; 73; 74; 75; 76; 77; 78; 79; 80; 81; 82; 83; 84; 85; 86; 87; 88]);
 (89, [89; 90; 91; 92; 93; 94; 95; 96; 97; 98; 99; 100])]
</code></pre></div></div>

<p>The algorithm builds up an array of lists during execution and prints the results at the end. We can’t print out row 1 in the table until the entire range has been evaluated. Upon closer examination of the table, a pattern of ranges emerges. For example, for 8, we have the ranges 8-12, 29-33, 42-46, 63-67, 84-88 and finally 97-100. There must be a pattern.</p>

<p>Here are the Fibonacci numbers less than 12.</p>

<table>
  <thead>
    <tr>
      <th>index</th>
      <th>F(n)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <td>1</td>
      <td>2</td>
    </tr>
    <tr>
      <td>2</td>
      <td>3</td>
    </tr>
    <tr>
      <td>3</td>
      <td>5</td>
    </tr>
    <tr>
      <td>4</td>
      <td>8</td>
    </tr>
  </tbody>
</table>

<p>We want all numbers containing <code class="language-plaintext highlighter-rouge">1</code>. These are <code class="language-plaintext highlighter-rouge">1</code> plus all of <code class="language-plaintext highlighter-rouge">Z + 1</code>, where <code class="language-plaintext highlighter-rouge">Z</code> is <code class="language-plaintext highlighter-rouge">{3, 5, 8}</code>, the subset of the Fibonacci sequence greater than <code class="language-plaintext highlighter-rouge">2</code>. We can’t use <code class="language-plaintext highlighter-rouge">2</code> as a Zeckendorf decomposition cannot have consecutive Fibonacci numbers (by definition).</p>

<p>Starting with the highest Fibonacci number in our subset, <code class="language-plaintext highlighter-rouge">8</code>, we cannot use <code class="language-plaintext highlighter-rouge">5</code>, but can use <code class="language-plaintext highlighter-rouge">3</code>, resulting in <code class="language-plaintext highlighter-rouge">8 + 1</code>, <code class="language-plaintext highlighter-rouge">8 + 3 + 1</code>, aka <code class="language-plaintext highlighter-rouge">9</code> and <code class="language-plaintext highlighter-rouge">12</code>. Then, taking our next highest starting number of <code class="language-plaintext highlighter-rouge">5</code>, we have only <code class="language-plaintext highlighter-rouge">5 + 1</code>, aka <code class="language-plaintext highlighter-rouge">6</code> and finally <code class="language-plaintext highlighter-rouge">3 + 1</code> aka <code class="language-plaintext highlighter-rouge">4</code>. The result is <code class="language-plaintext highlighter-rouge">1, 4, 6, 9, 12</code>.</p>

<p>Continuing to the next row in the output, we now need to find all the numbers containing <code class="language-plaintext highlighter-rouge">2</code> which are <code class="language-plaintext highlighter-rouge">Z + 2</code> where Z is <code class="language-plaintext highlighter-rouge">{5, 8}</code>. This results in <code class="language-plaintext highlighter-rouge">8 + 2</code> and <code class="language-plaintext highlighter-rouge">5 + 2</code>, resulting in <code class="language-plaintext highlighter-rouge">2, 7, 10</code>.</p>

<p>This can be written as a recursive algorithm which requires no storage beyond the Fibonacci sequence itself. It prints the numbers as they are generated.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">fib_print</span> <span class="n">hi</span> <span class="o">=</span>
  <span class="k">let</span> <span class="k">rec</span> <span class="n">build</span> <span class="n">a</span> <span class="n">b</span> <span class="n">acc</span> <span class="o">=</span>
    <span class="k">if</span> <span class="n">a</span> <span class="o">&gt;</span> <span class="n">hi</span> <span class="k">then</span> <span class="nn">Array</span><span class="p">.</span><span class="n">of_list</span> <span class="p">(</span><span class="nn">List</span><span class="p">.</span><span class="n">rev</span> <span class="n">acc</span><span class="p">)</span>
    <span class="k">else</span> <span class="n">build</span> <span class="n">b</span> <span class="p">(</span><span class="n">a</span> <span class="o">+</span> <span class="n">b</span><span class="p">)</span> <span class="p">(</span><span class="n">a</span> <span class="o">::</span> <span class="n">acc</span><span class="p">)</span>
  <span class="k">in</span>
  <span class="k">let</span> <span class="n">fibs</span> <span class="o">=</span> <span class="n">build</span> <span class="mi">1</span> <span class="mi">2</span> <span class="bp">[]</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">n</span> <span class="o">=</span> <span class="nn">Array</span><span class="p">.</span><span class="n">length</span> <span class="n">fibs</span> <span class="k">in</span>
  <span class="nn">Array</span><span class="p">.</span><span class="n">iteri</span> <span class="p">(</span><span class="k">fun</span> <span class="n">k</span> <span class="n">fk</span> <span class="o">-&gt;</span>
    <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"%d:"</span> <span class="n">fk</span><span class="p">;</span>
    <span class="k">let</span> <span class="k">rec</span> <span class="n">go</span> <span class="n">idx</span> <span class="n">value</span> <span class="n">prev_used</span> <span class="o">=</span>
      <span class="k">if</span> <span class="n">fk</span> <span class="o">+</span> <span class="n">value</span> <span class="o">&gt;</span> <span class="n">hi</span> <span class="k">then</span> <span class="bp">()</span>
      <span class="k">else</span> <span class="k">if</span> <span class="n">idx</span> <span class="o">&lt;</span> <span class="mi">0</span> <span class="k">then</span>
        <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">" %d"</span> <span class="p">(</span><span class="n">fk</span> <span class="o">+</span> <span class="n">value</span><span class="p">)</span>
      <span class="k">else</span> <span class="k">if</span> <span class="n">idx</span> <span class="o">&gt;=</span> <span class="n">k</span> <span class="o">-</span> <span class="mi">1</span> <span class="o">&amp;&amp;</span> <span class="n">idx</span> <span class="o">&lt;=</span> <span class="n">k</span> <span class="o">+</span> <span class="mi">1</span> <span class="k">then</span>
        <span class="n">go</span> <span class="p">(</span><span class="n">idx</span> <span class="o">-</span> <span class="mi">1</span><span class="p">)</span> <span class="n">value</span> <span class="bp">false</span>
      <span class="k">else</span> <span class="p">(</span>
        <span class="n">go</span> <span class="p">(</span><span class="n">idx</span> <span class="o">-</span> <span class="mi">1</span><span class="p">)</span> <span class="n">value</span> <span class="bp">false</span><span class="p">;</span>
        <span class="k">if</span> <span class="n">not</span> <span class="n">prev_used</span> <span class="k">then</span>
          <span class="n">go</span> <span class="p">(</span><span class="n">idx</span> <span class="o">-</span> <span class="mi">1</span><span class="p">)</span> <span class="p">(</span><span class="n">value</span> <span class="o">+</span> <span class="n">fibs</span><span class="o">.</span><span class="p">(</span><span class="n">idx</span><span class="p">))</span> <span class="bp">true</span>
      <span class="p">)</span>
    <span class="k">in</span>
    <span class="n">go</span> <span class="p">(</span><span class="n">n</span> <span class="o">-</span> <span class="mi">1</span><span class="p">)</span> <span class="mi">0</span> <span class="bp">false</span><span class="p">;</span>
    <span class="n">print_newline</span> <span class="bp">()</span>
  <span class="p">)</span> <span class="n">fibs</span>
</code></pre></div></div>
