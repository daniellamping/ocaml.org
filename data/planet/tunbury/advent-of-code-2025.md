---
title: Advent of Code 2025
description: With the start of Advent comes a new set of Advent of Code problems.
  My code is available at mtelvers/aoc2025.
url: https://www.tunbury.org/2025/12/12/advent-of-code/
date: 2025-12-12T18:00:00-00:00
preview_image: https://www.tunbury.org/images/aoc2025.png
authors:
- Mark Elvers
source:
ignore:
---

<p>With the start of Advent comes a new set of Advent of Code problems. My code is available at <a href="https://github.com/mtelvers/aoc2025">mtelvers/aoc2025</a>.</p>

<h1>Day 1 - Secret Entrance</h1>

<p>A dial points to 50. Follow the sequence of turns to see how many times it lands on zero. The only gotcha here was that the real input had values &gt; 100.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>L68
L30
R48
L5
R60
L55
L1
L99
R14
L82
</code></pre></div></div>

<p>Part 2 was fiddly as the corner cases needed careful consideration. Landing on zero should be counted, so start with the answer from part 1. Add the number of clicks to turn through divided by 100 (the quotient) to count the number of full rotations. Then add the cases where the turning left by the number of clicks modulo 100 would be less than zero, and the same for turning right when it would be greater than 100.</p>

<p>Note that starting at 0 and turning left 5 does not count as passing zero. So if your zero passing test is <code class="language-plaintext highlighter-rouge">position &lt; value</code>, then this is only true when <code class="language-plaintext highlighter-rouge">position &gt; 0</code>.</p>
<h1>Day 2 - Gift Shop</h1>

<p>Find repeating patterns in some number ranges.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>11-22,95-115,998-1012,1188511880-1188511890,222220-222224,
1698522-1698528,446443-446449,38593856-38593862,565653-565659,
824824821-824824827,2121212118-2121212124
</code></pre></div></div>

<h2>Part 1</h2>

<p>I decided right away to use integer comparison rather than converting numbers to strings and then comparing them. The number of digits in an integer when written in base 10 is the <code class="language-plaintext highlighter-rouge">1 + int(log10 x)</code>. For part 1, the challenge was to look for exact splits <code class="language-plaintext highlighter-rouge">11</code> or <code class="language-plaintext highlighter-rouge">123123</code>; therefore, the length must be even. We can use the divisor <code class="language-plaintext highlighter-rouge">10^(length/2)</code>, and test with <code class="language-plaintext highlighter-rouge">x / divisor = x mod divisor</code> and sum all the numbers where this is true.</p>

<h2>Part 2</h2>

<p>The problem is extended to allow any equal chunking. Thus, <code class="language-plaintext highlighter-rouge">824824824</code> is now valid as it has three chunks of 3 digits. Given the maximum length of a 64-bit integer is 20 digits, we only need the factors of the numbers 1 to 20, which could be entered as a static list. I decided to calculate these in code using a simple division test up to the square root of the number. I should memoise these results to avoid repeated recalculation. Once I had a list of factors, I folded over the list, testing each with a recursive function to verify that each chunk was equal.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">base</span> <span class="o">=</span> <span class="n">pow</span> <span class="mi">10</span> <span class="n">factor</span> <span class="k">in</span>
<span class="k">let</span> <span class="n">modulo</span> <span class="o">=</span> <span class="n">x</span> <span class="ow">mod</span> <span class="n">base</span> <span class="k">in</span>

<span class="k">let</span> <span class="k">rec</span> <span class="n">loop</span> <span class="n">v</span> <span class="o">=</span>
  <span class="k">if</span> <span class="n">v</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">then</span> <span class="bp">true</span>
  <span class="k">else</span> <span class="k">if</span> <span class="n">v</span> <span class="ow">mod</span> <span class="n">base</span> <span class="o">=</span> <span class="n">modulo</span> <span class="k">then</span> <span class="n">loop</span> <span class="p">(</span><span class="n">v</span> <span class="o">/</span> <span class="n">base</span><span class="p">)</span>
  <span class="k">else</span> <span class="bp">false</span>
<span class="k">in</span>

<span class="n">loop</span> <span class="p">(</span><span class="n">x</span> <span class="o">/</span> <span class="n">base</span><span class="p">)</span>
</code></pre></div></div>

<p>The only gotcha I found was that numbers less than 10 came out as true, so I constrained the lower bound of the range to 10.</p>
<h1>Day 3 - Lobby</h1>

<p>Sum the largest number you can make using N digits from the given sequences. The order of the digits cannot be changed.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>987654321111111
811111111111119
234234234234278
818181911112111
</code></pre></div></div>

<h2>Part 1</h2>

<p>As it was initially presented, you are only required to consider two digits. As <code class="language-plaintext highlighter-rouge">9_</code> will always be bigger than <code class="language-plaintext highlighter-rouge">8_</code>, this becomes a case of finding the largest digit available, which still leaves one digit. If there is more than one digit left, then pick the largest one. For example, given <code class="language-plaintext highlighter-rouge">818181911112111</code>, the largest first digit is <code class="language-plaintext highlighter-rouge">9</code>, followed by the largest digit in <code class="language-plaintext highlighter-rouge">11112111</code>, which is 2.</p>

<p>I pattern-matched the list of numbers to extract two digits in a recursive loop:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="k">rec</span> <span class="n">loop</span> <span class="n">max_left</span> <span class="n">max_right</span> <span class="o">=</span> <span class="k">function</span>
  <span class="o">|</span> <span class="n">l</span> <span class="o">::</span> <span class="n">r</span> <span class="o">::</span> <span class="n">tl</span> <span class="o">-&gt;</span>
    <span class="k">if</span> <span class="n">l</span> <span class="o">&gt;</span> <span class="n">max_left</span> <span class="k">then</span> <span class="n">loop</span> <span class="n">l</span> <span class="n">r</span> <span class="p">(</span><span class="n">r</span> <span class="o">::</span> <span class="n">tl</span><span class="p">)</span>
    <span class="k">else</span> <span class="k">if</span> <span class="n">r</span> <span class="o">&gt;</span> <span class="n">max_right</span> <span class="k">then</span> <span class="n">loop</span> <span class="n">max_left</span> <span class="n">r</span> <span class="p">(</span><span class="n">r</span> <span class="o">::</span> <span class="n">tl</span><span class="p">)</span>
    <span class="k">else</span> <span class="n">loop</span> <span class="n">max_left</span> <span class="n">max_right</span> <span class="p">(</span><span class="n">r</span> <span class="o">::</span> <span class="n">tl</span><span class="p">)</span>
  <span class="o">|</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="n">max_left</span><span class="o">,</span> <span class="n">max_right</span><span class="p">)</span>
<span class="k">in</span>
<span class="k">let</span> <span class="n">l</span><span class="o">,</span> <span class="n">r</span> <span class="o">=</span> <span class="n">loop</span> <span class="mi">0</span> <span class="mi">0</span> <span class="n">bank</span> <span class="k">in</span>
<span class="k">let</span> <span class="n">num</span> <span class="o">=</span> <span class="n">l</span> <span class="o">*</span> <span class="mi">10</span> <span class="o">+</span> <span class="n">r</span>
</code></pre></div></div>

<h2>Part 2</h2>

<p>Annoyingly, this changed the problem significantly, as it increased the number length from 2 to 12. My list approach now seemed unworkable, and I switched to using arrays.</p>

<p>Taking <code class="language-plaintext highlighter-rouge">818181911112111</code> as an example, I extracted <code class="language-plaintext highlighter-rouge">8181</code>, leaving 11 digits available and found the maximum value, which is the first <code class="language-plaintext highlighter-rouge">8</code>. Then I extracted a new subarray, <code class="language-plaintext highlighter-rouge">1818</code>, starting after the first digit matched and leaving 10 digits available. The maximum here is the <code class="language-plaintext highlighter-rouge">8</code> at index 1. Repeating this process, finding the maximum in <code class="language-plaintext highlighter-rouge">181</code>, then of <code class="language-plaintext highlighter-rouge">19</code>, and finally, all the remaining numbers must be taken to achieve the correct length.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>818181911112111

8181 -&gt; i=0 [i]=8
1818 -&gt; i=1 [i]=8
181 -&gt; i=1 [i]=8
19 -&gt; i=1 [i]=9
1 -&gt; i=0 [i]=1
1 -&gt; i=0 [i]=1
1 -&gt; i=0 [i]=1
1 -&gt; i=0 [i]=1
2 -&gt; i=0 [i]=2
1 -&gt; i=0 [i]=1
1 -&gt; i=0 [i]=1
1 -&gt; i=0 [i]=1
</code></pre></div></div>

<p>This worked out nicely, and I parameterised the function to accept the length of the number required so that I could use this code for part 1 as well.</p>

<h1>Day 4 - Paper Bale Warehouse</h1>

<p>Find the number of <code class="language-plaintext highlighter-rouge">@</code> which have fewer than four <code class="language-plaintext highlighter-rouge">@</code> as neighbours.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>..@@.@@@@.
@@@.@.@.@@
@@@@@.@.@@
@.@@@@..@.
@@.@@@@.@@
.@@@@@@@.@
.@.@.@.@@@
@.@@@.@@@@
.@@@@@@@@.
@.@.@@@.@.
</code></pre></div></div>

<p>I chose	to read the input into a Map, which I have used several times before, so I copied my implementation from AoC 2024 Day 10.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">type</span> <span class="n">coord</span> <span class="o">=</span> <span class="p">{</span> <span class="n">y</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span> <span class="n">x</span> <span class="o">:</span> <span class="kt">int</span> <span class="p">}</span>

<span class="k">module</span> <span class="nc">CoordMap</span> <span class="o">=</span> <span class="nn">Map</span><span class="p">.</span><span class="nc">Make</span> <span class="p">(</span><span class="k">struct</span>
  <span class="k">type</span> <span class="n">t</span> <span class="o">=</span> <span class="n">coord</span>

  <span class="k">let</span> <span class="n">compare</span> <span class="o">=</span> <span class="n">compare</span>
<span class="k">end</span><span class="p">)</span>
</code></pre></div></div>

<p>I set up a list of directions around the centre point.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">neighbours</span> <span class="o">=</span>
  <span class="p">[</span>
    <span class="p">{</span> <span class="n">y</span> <span class="o">=</span> <span class="mi">1</span><span class="p">;</span> <span class="n">x</span> <span class="o">=</span> <span class="o">-</span><span class="mi">1</span> <span class="p">};</span>
    <span class="p">{</span> <span class="n">y</span> <span class="o">=</span> <span class="mi">1</span><span class="p">;</span> <span class="n">x</span> <span class="o">=</span> <span class="mi">0</span> <span class="p">};</span>
    <span class="p">{</span> <span class="n">y</span> <span class="o">=</span> <span class="mi">1</span><span class="p">;</span> <span class="n">x</span> <span class="o">=</span> <span class="mi">1</span> <span class="p">};</span>
    <span class="p">{</span> <span class="n">y</span> <span class="o">=</span> <span class="mi">0</span><span class="p">;</span> <span class="n">x</span> <span class="o">=</span> <span class="o">-</span><span class="mi">1</span> <span class="p">};</span>
    <span class="p">{</span> <span class="n">y</span> <span class="o">=</span> <span class="mi">0</span><span class="p">;</span> <span class="n">x</span> <span class="o">=</span> <span class="mi">1</span> <span class="p">};</span>
    <span class="p">{</span> <span class="n">y</span> <span class="o">=</span> <span class="o">-</span><span class="mi">1</span><span class="p">;</span> <span class="n">x</span> <span class="o">=</span> <span class="o">-</span><span class="mi">1</span> <span class="p">};</span>
    <span class="p">{</span> <span class="n">y</span> <span class="o">=</span> <span class="o">-</span><span class="mi">1</span><span class="p">;</span> <span class="n">x</span> <span class="o">=</span> <span class="mi">0</span> <span class="p">};</span>
    <span class="p">{</span> <span class="n">y</span> <span class="o">=</span> <span class="o">-</span><span class="mi">1</span><span class="p">;</span> <span class="n">x</span> <span class="o">=</span> <span class="mi">1</span> <span class="p">};</span>
  <span class="p">]</span>
</code></pre></div></div>

<h2>Part 1</h2>

<p>Fold over the map, and where there is an <code class="language-plaintext highlighter-rouge">@</code>, I folded over the list of neighbours, counting the number with bales, which could then be summed in the outer fold.</p>

<h2>Part 2</h2>

<p>For the second part, the free bales needed to be removed, and then the calculation was repeated, trying again until no more bales could be removed.</p>

<p>At this point, I realised that the map could be simplified to a set, as there is no need to distinguish between the boundary and an empty square.</p>

<p>Therefore, rather than just counting the free bales, I added these to a set which could be subtracted from the original set and iterated.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="k">rec</span> <span class="n">part2</span> <span class="n">w</span> <span class="o">=</span>
  <span class="nn">CoordSet</span><span class="p">.</span><span class="n">fold</span>
    <span class="p">(</span><span class="k">fun</span> <span class="n">k</span> <span class="n">acc</span> <span class="o">-&gt;</span> <span class="k">if</span> <span class="n">is_free_bales</span> <span class="n">w</span> <span class="n">k</span> <span class="k">then</span> <span class="nn">CoordSet</span><span class="p">.</span><span class="n">add</span> <span class="n">k</span> <span class="n">acc</span> <span class="k">else</span> <span class="n">acc</span><span class="p">)</span>
    <span class="n">w</span> <span class="nn">CoordSet</span><span class="p">.</span><span class="n">empty</span>
  <span class="o">|&gt;</span> <span class="k">fun</span> <span class="n">free_bales</span> <span class="o">-&gt;</span>
  <span class="k">if</span> <span class="nn">CoordSet</span><span class="p">.</span><span class="n">is_empty</span> <span class="n">free_bales</span> <span class="k">then</span> <span class="nn">CoordSet</span><span class="p">.</span><span class="n">cardinal</span> <span class="n">w</span>
  <span class="k">else</span> <span class="nn">CoordSet</span><span class="p">.</span><span class="n">diff</span> <span class="n">w</span> <span class="n">free_bales</span> <span class="o">|&gt;</span> <span class="n">part2</span>

<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span>
  <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"part 2: %i</span><span class="se">\n</span><span class="s2">"</span> <span class="p">(</span><span class="nn">CoordSet</span><span class="p">.</span><span class="n">cardinal</span> <span class="n">warehouse</span> <span class="o">-</span> <span class="n">part2</span> <span class="n">warehouse</span><span class="p">)</span>
</code></pre></div></div>
<h1>Day 5 - Cafeteria</h1>

<p>Count the number of elements from the second list which appear in the list of (inclusive) ranges.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>3-5
10-14
16-20
12-18

1
5
8
11
17
32
</code></pre></div></div>

<h2>Part 1</h2>

<p>I read the input data into two variables, <code class="language-plaintext highlighter-rouge">fresh</code> as a list of pairs for the ranges and <code class="language-plaintext highlighter-rouge">ingredients</code> as an int list. For part one, it’s just a case of summing values where the ingredient falls within the range:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">part1</span> <span class="o">=</span>
  <span class="nn">List</span><span class="p">.</span><span class="n">fold_left</span>
    <span class="p">(</span><span class="k">fun</span> <span class="n">f</span> <span class="n">i</span> <span class="o">-&gt;</span>
      <span class="nn">List</span><span class="p">.</span><span class="n">find_opt</span> <span class="p">(</span><span class="k">fun</span> <span class="p">(</span><span class="n">l</span><span class="o">,</span> <span class="n">h</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">i</span> <span class="o">&gt;=</span> <span class="n">l</span> <span class="o">&amp;&amp;</span> <span class="n">i</span> <span class="o">&lt;=</span> <span class="n">h</span><span class="p">)</span> <span class="n">fresh</span> <span class="o">|&gt;</span> <span class="k">function</span>
      <span class="o">|</span> <span class="nc">Some</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="n">f</span> <span class="o">+</span> <span class="mi">1</span>
      <span class="o">|</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="n">f</span><span class="p">)</span>
    <span class="mi">0</span> <span class="n">ingredients</span>
</code></pre></div></div>

<h2>Part 2</h2>

<p>Ignoring the second list, count the values represented by the list of ranges. <code class="language-plaintext highlighter-rouge">3-5,10-14</code> would be a <code class="language-plaintext highlighter-rouge">3 + 5 = 8</code>. I didn’t verify this, but it is likely that the actual input ranges aren’t as tidy as the example data. We are told that ranges overlap, but I expect there will be ranges that entirely encompass other ranges, as well as ranges that are immediately adjacent, and so on. I wrote an <code class="language-plaintext highlighter-rouge">add</code> function to add a range to a list of ranges. I think it would have looked better using <code class="language-plaintext highlighter-rouge">type range = { low: int; high: int }</code>, but I’d come this far using pairs.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">add</span> <span class="p">(</span><span class="n">low</span><span class="o">,</span> <span class="n">high</span><span class="p">)</span> <span class="n">t</span> <span class="o">=</span>
  <span class="k">let</span> <span class="k">rec</span> <span class="n">loop</span> <span class="n">acc</span> <span class="p">(</span><span class="n">low</span><span class="o">,</span> <span class="n">high</span><span class="p">)</span> <span class="o">=</span> <span class="k">function</span>
    <span class="o">|</span> <span class="bp">[]</span> <span class="o">-&gt;</span> <span class="nn">List</span><span class="p">.</span><span class="n">rev</span> <span class="p">((</span><span class="n">low</span><span class="o">,</span> <span class="n">high</span><span class="p">)</span> <span class="o">::</span> <span class="n">acc</span><span class="p">)</span>
    <span class="o">|</span> <span class="p">(</span><span class="n">l</span><span class="o">,</span> <span class="n">h</span><span class="p">)</span> <span class="o">::</span> <span class="n">tl</span> <span class="k">when</span> <span class="n">h</span> <span class="o">+</span> <span class="mi">1</span> <span class="o">&lt;</span> <span class="n">low</span> <span class="o">-&gt;</span> <span class="n">loop</span> <span class="p">((</span><span class="n">l</span><span class="o">,</span> <span class="n">h</span><span class="p">)</span> <span class="o">::</span> <span class="n">acc</span><span class="p">)</span> <span class="p">(</span><span class="n">low</span><span class="o">,</span> <span class="n">high</span><span class="p">)</span> <span class="n">tl</span>
    <span class="o">|</span> <span class="p">(</span><span class="n">l</span><span class="o">,</span> <span class="n">h</span><span class="p">)</span> <span class="o">::</span> <span class="n">tl</span> <span class="k">when</span> <span class="n">high</span> <span class="o">+</span> <span class="mi">1</span> <span class="o">&lt;</span> <span class="n">l</span> <span class="o">-&gt;</span>
        <span class="nn">List</span><span class="p">.</span><span class="n">rev_append</span> <span class="n">acc</span> <span class="p">((</span><span class="n">low</span><span class="o">,</span> <span class="n">high</span><span class="p">)</span> <span class="o">::</span> <span class="p">(</span><span class="n">l</span><span class="o">,</span> <span class="n">h</span><span class="p">)</span> <span class="o">::</span> <span class="n">tl</span><span class="p">)</span>
    <span class="o">|</span> <span class="p">(</span><span class="n">l</span><span class="o">,</span> <span class="n">h</span><span class="p">)</span> <span class="o">::</span> <span class="n">tl</span> <span class="o">-&gt;</span> <span class="n">loop</span> <span class="n">acc</span> <span class="p">(</span><span class="n">min</span> <span class="n">l</span> <span class="n">low</span><span class="o">,</span> <span class="n">max</span> <span class="n">h</span> <span class="n">high</span><span class="p">)</span> <span class="n">tl</span>
  <span class="k">in</span>
  <span class="n">loop</span> <span class="bp">[]</span> <span class="p">(</span><span class="n">low</span><span class="o">,</span> <span class="n">high</span><span class="p">)</span> <span class="n">t</span>
</code></pre></div></div>

<p>I wrote some test cases to cover the weird cases not present in the example data.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="bp">[]</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="mi">2</span><span class="o">,</span> <span class="mi">5</span><span class="p">)</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="mi">7</span><span class="o">,</span> <span class="mi">9</span><span class="p">);;</span>                 <span class="o">#</span> <span class="n">simple</span> <span class="p">[(</span><span class="mi">2</span><span class="o">,</span> <span class="mi">5</span><span class="p">);</span> <span class="p">(</span><span class="mi">7</span><span class="o">,</span> <span class="mi">9</span><span class="p">)]</span>
<span class="bp">[]</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="mi">2</span><span class="o">,</span> <span class="mi">5</span><span class="p">)</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="mi">7</span><span class="o">,</span> <span class="mi">9</span><span class="p">)</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="mi">4</span><span class="o">,</span> <span class="mi">8</span><span class="p">);;</span>   <span class="o">#</span> <span class="n">join</span> <span class="p">[(</span><span class="mi">2</span><span class="o">,</span> <span class="mi">9</span><span class="p">)]</span>
<span class="bp">[]</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="mi">2</span><span class="o">,</span> <span class="mi">5</span><span class="p">)</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="mi">7</span><span class="o">,</span> <span class="mi">9</span><span class="p">)</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="mi">1</span><span class="o">,</span> <span class="mi">10</span><span class="p">);;</span>  <span class="o">#</span> <span class="n">encompass</span> <span class="p">[(</span><span class="mi">1</span><span class="o">,</span> <span class="mi">10</span><span class="p">)]</span>
<span class="bp">[]</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="mi">2</span><span class="o">,</span> <span class="mi">5</span><span class="p">)</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="mi">6</span><span class="o">,</span> <span class="mi">9</span><span class="p">);;</span>                 <span class="o">#</span> <span class="n">adjacent</span> <span class="p">[(</span><span class="mi">2</span><span class="o">,</span> <span class="mi">9</span><span class="p">)]</span>
</code></pre></div></div>

<p>With the code tested, part 2 used the <code class="language-plaintext highlighter-rouge">add</code> function to create a combined list, and then summed the difference between the high and low values + 1.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">part2</span> <span class="o">=</span>
  <span class="nn">List</span><span class="p">.</span><span class="n">fold_left</span> <span class="p">(</span><span class="k">fun</span> <span class="n">acc</span> <span class="p">(</span><span class="n">l</span><span class="o">,</span> <span class="n">h</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">add</span> <span class="p">(</span><span class="n">l</span><span class="o">,</span> <span class="n">h</span><span class="p">)</span> <span class="n">acc</span><span class="p">)</span> <span class="bp">[]</span> <span class="n">fresh</span>
  <span class="o">|&gt;</span> <span class="nn">List</span><span class="p">.</span><span class="n">fold_left</span> <span class="p">(</span><span class="k">fun</span> <span class="n">acc</span> <span class="p">(</span><span class="n">l</span><span class="o">,</span> <span class="n">h</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">acc</span> <span class="o">+</span> <span class="p">(</span><span class="n">h</span> <span class="o">-</span> <span class="n">l</span> <span class="o">+</span> <span class="mi">1</span><span class="p">))</span> <span class="mi">0</span>
</code></pre></div></div>
<h1>Day 6 - Trash Compactor</h1>

<p>Sum the cryptically presented equations.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>123 328  51 64 
 45 64  387 23 
  6 98  215 314
*   +   *   +  
</code></pre></div></div>

<h1>Part 1</h1>

<p>Apply the operator at the bottom of the column to the numbers above it and sum the results.</p>

<p>This was a straightforward case of reading a list of lines, then splitting it up into a list of lists of numbers, resulting in a kind of matrix. Use a transpose function and then apply the operator on each list using a fold operation. Note that it’s just addition and multiplication, both of which are commutative.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="k">rec</span> <span class="n">transpose</span> <span class="o">=</span> <span class="k">function</span>
  <span class="o">|</span> <span class="bp">[]</span> <span class="o">|</span> <span class="bp">[]</span> <span class="o">::</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="bp">[]</span>
  <span class="o">|</span> <span class="n">rows</span> <span class="o">-&gt;</span> <span class="nn">List</span><span class="p">.</span><span class="n">map</span> <span class="nn">List</span><span class="p">.</span><span class="n">hd</span> <span class="n">rows</span> <span class="o">::</span> <span class="n">transpose</span> <span class="p">(</span><span class="nn">List</span><span class="p">.</span><span class="n">map</span> <span class="nn">List</span><span class="p">.</span><span class="n">tl</span> <span class="n">rows</span><span class="p">)</span>
</code></pre></div></div>

<h1>Part 2</h1>

<p>It was odd in the original input that sometimes there was one space between the numbers, while other times there were two. This all became clear in part 2, as the problem was reframed that the numbers themselves were also transposed. Thus, the far right column was actually <code class="language-plaintext highlighter-rouge">4 + 431 + 623</code>.</p>

<p>Reading the input as characters and transposing it resulted in, what is in effect, the part 1 problem, but the data structure isn’t pretty.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>1  *
24  
356 
    
369+
248 
8   
    
 32*
581 
175 
    
623+
431 
  4 
</code></pre></div></div>

<p>I can see that you could write a conversion function for both the part 1 and the transposed part 2 structure into a standard format and use the same processing function to sum both datasets, but I didn’t!</p>

<p>I created a <code class="language-plaintext highlighter-rouge">split_last</code> function to from the last element from each row (list).</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="k">rec</span> <span class="n">split_last</span> <span class="o">=</span> <span class="k">function</span>
  <span class="o">|</span> <span class="bp">[]</span> <span class="o">-&gt;</span> <span class="k">assert</span> <span class="bp">false</span>
  <span class="o">|</span> <span class="p">[</span> <span class="n">x</span> <span class="p">]</span> <span class="o">-&gt;</span> <span class="p">([]</span><span class="o">,</span> <span class="n">x</span><span class="p">)</span>
  <span class="o">|</span> <span class="n">x</span> <span class="o">::</span> <span class="n">xs</span> <span class="o">-&gt;</span>
      <span class="k">let</span> <span class="n">init</span><span class="o">,</span> <span class="n">last</span> <span class="o">=</span> <span class="n">split_last</span> <span class="n">xs</span> <span class="k">in</span>
      <span class="p">(</span><span class="n">x</span> <span class="o">::</span> <span class="n">init</span><span class="o">,</span> <span class="n">last</span><span class="p">)</span>
</code></pre></div></div>

<p>This gives me the operator plus a list of characters. The list of characters can be concatenated, trimmed and converted into a number. Then, using an inelegant fold which threads the operator, the intermediate sum and the overall sum, you can calculate the answer.</p>
<h1>Day 7 - Laboratories</h1>

<p>Starting from <code class="language-plaintext highlighter-rouge">S</code>, beam down through the map, splitting at each <code class="language-plaintext highlighter-rouge">^</code>.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>.......S.......
...............
.......^.......
...............
......^.^......
...............
.....^.^.^.....
...............
....^.^...^....
...............
...^.^...^.^...
...............
..^...^.....^..
...............
.^.^.^.^.^...^.
...............
</code></pre></div></div>

<p>I read the diagram as a map of <code class="language-plaintext highlighter-rouge">(x,y)</code> coordinates, but in retrospect, a list of arrays may have been a more optimal choice.</p>

<h2>Part 1</h2>

<p>In this part, calculate how many times we reach an <code class="language-plaintext highlighter-rouge">^</code>. This is a breadth-first search tracking the number of beams at each iteration. I used a coordinate map to track the beams at each level which helpfully automatically absorbs duplicate beams.</p>

<h2>Part 2</h2>

<p>This time, follow each possible path and count how many ways there are to get to the end. This is a depth-first search where the trivial algorithm works on the test dataset, but with the actual input, the number of possibilities is too large. Therefore, I added a hashtbl to memoise the results at each level. With this, all 25 trillion ways are counted in a matter of a few milliseconds.</p>
<h1>Day 8 - Playground</h1>

<p>Compute the distance between vectors in 3D space and build them into a graph by linking the closest pairs.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>162,817,812
57,618,57
906,360,560
592,479,940
352,342,300
466,668,158
542,29,236
431,825,988
739,650,466
52,470,668
216,146,977
819,987,18
117,168,530
805,96,715
346,949,466
970,615,88
941,993,340
862,61,35
984,92,344
425,690,689
</code></pre></div></div>

<p>I read the input in as a list of vectors <code class="language-plaintext highlighter-rouge">type vector = { x : float; y : float; z : float }</code>. Next, I computed a list of distances between all the pairs, resulting in a <code class="language-plaintext highlighter-rouge">((vector * vector) * float) list</code>. A network is a set of vectors, and overall, there is a set of networks. I couldn’t decide on the best way to store this, so for expediency, I went with sets.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">module</span> <span class="nc">Network</span> <span class="o">=</span> <span class="nn">Set</span><span class="p">.</span><span class="nc">Make</span> <span class="p">(</span><span class="k">struct</span>
  <span class="k">type</span> <span class="n">t</span> <span class="o">=</span> <span class="n">vector</span>

  <span class="k">let</span> <span class="n">compare</span> <span class="o">=</span> <span class="n">compare</span>
<span class="k">end</span><span class="p">)</span>

<span class="k">module</span> <span class="nc">NetworkSet</span> <span class="o">=</span> <span class="nn">Set</span><span class="p">.</span><span class="nc">Make</span> <span class="p">(</span><span class="nc">Network</span><span class="p">)</span> 
</code></pre></div></div>

<p>With this, I wrote a function to join two nodes together. This first checks if either node already existed in any network. If neither node exists, create a new network with those two nodes. If one node exists in any network, then add the other node. If both nodes exist, then union the two networks together. As adding a value to a set is idempotent, it is not necessary to distinguish which value needs to be added: <code class="language-plaintext highlighter-rouge">|&gt; Network.add v1 |&gt; Network.add v2</code></p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">join</span> <span class="n">v1</span> <span class="n">v2</span> <span class="n">acc</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">s1</span><span class="o">,</span> <span class="n">s2</span> <span class="o">=</span>
    <span class="nn">NetworkSet</span><span class="p">.</span><span class="n">partition</span> <span class="p">(</span><span class="k">fun</span> <span class="n">vs</span> <span class="o">-&gt;</span> <span class="nn">Network</span><span class="p">.</span><span class="n">mem</span> <span class="n">v1</span> <span class="n">vs</span> <span class="o">||</span> <span class="nn">Network</span><span class="p">.</span><span class="n">mem</span> <span class="n">v2</span> <span class="n">vs</span><span class="p">)</span> <span class="n">acc</span>
  <span class="k">in</span>  
  <span class="nn">NetworkSet</span><span class="p">.</span><span class="n">singleton</span>
    <span class="p">(</span><span class="k">match</span> <span class="nn">NetworkSet</span><span class="p">.</span><span class="n">cardinal</span> <span class="n">s1</span> <span class="k">with</span>
    <span class="o">|</span> <span class="mi">0</span> <span class="o">-&gt;</span> <span class="nn">Network</span><span class="p">.(</span><span class="n">singleton</span> <span class="n">v1</span> <span class="o">|&gt;</span> <span class="n">add</span> <span class="n">v2</span><span class="p">)</span>
    <span class="o">|</span> <span class="mi">1</span> <span class="o">-&gt;</span> <span class="nn">NetworkSet</span><span class="p">.</span><span class="n">choose</span> <span class="n">s1</span> <span class="o">|&gt;</span> <span class="nn">Network</span><span class="p">.</span><span class="n">add</span> <span class="n">v1</span> <span class="o">|&gt;</span> <span class="nn">Network</span><span class="p">.</span><span class="n">add</span> <span class="n">v2</span>
    <span class="o">|</span> <span class="mi">2</span> <span class="o">-&gt;</span> <span class="nn">NetworkSet</span><span class="p">.</span><span class="n">fold</span> <span class="p">(</span><span class="k">fun</span> <span class="n">vs</span> <span class="n">acc</span> <span class="o">-&gt;</span> <span class="nn">Network</span><span class="p">.</span><span class="n">union</span> <span class="n">acc</span> <span class="n">vs</span><span class="p">)</span> <span class="n">s1</span> <span class="nn">Network</span><span class="p">.</span><span class="n">empty</span>
    <span class="o">|</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="k">assert</span> <span class="bp">false</span><span class="p">)</span>
  <span class="o">|&gt;</span> <span class="nn">NetworkSet</span><span class="p">.</span><span class="n">union</span> <span class="n">s2</span>

</code></pre></div></div>

<h1>Part 1</h1>

<p>Take the first 1000 vector pairs and add them to the <code class="language-plaintext highlighter-rouge">NetworkSet</code>, then convert the <code class="language-plaintext highlighter-rouge">NetworkSet</code> into a list of the size of each network, sort the list, take the first three and fold over them to get the answer.</p>

<h1>Part 2</h1>

<p>Continue adding vector pairs until all the vectors are connected then find the produce of the x coordinate of the final two vectors. I used a recursive function to repeatedly add pairs until the size of the network equalled the total number of vectors.</p>
<h1>Day 9 - Movie Theatre</h1>

<p>The input is a set of vertices. Draw the largest rectangle between any pair.</p>

<p>The vertices were specified as a list.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>7,1
11,1
11,7
9,7
9,5
2,5
2,3
7,3
</code></pre></div></div>

<p>Visually, this is:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>..............
.......#...#..
..............
..#....#......
..............
..#......#....
..............
.........#.#..
..............
</code></pre></div></div>

<h2>Part 1</h2>

<p>This couldn’t have been easier, particularly following day 8, as the input parser and combination generator are the same. Calculate the area of all the rectangles, then sort the list to find the largest.</p>

<h2>Part 2</h2>

<p>The extension was that the rectangle must be within the polygon defined by the input list of vertices. The input coordinates are in the range 0-100,000 on both x and y; therefore, we must do this mathematically, as the set will be too large.</p>

<p>To test if a polygon is contained within another polygon, then all vertices of A must be inside B, and none of the edges of A must cross the edges of B.</p>

<p>I used the ray casting algorithm to determine if a point was in a polygon. Due to the way the coordinate grid works, the code is somewhat messy, as all the boundaries are contained within the shape. Then test all pairs of edges to see if they crossed using the cross product to see if the endpoints lie on opposite sides of the infinite line defined by the other segment.</p>

<h1>Day 10 - Factory</h1>

<p>The input is a pattern of lights, followed by a list of buttons and which lights they turn on and finally a list of counter values.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[.##.] (3) (1,3) (2) (2,3) (0,2) (0,1) {3,5,4,7}
[...#.] (0,2,3,4) (2,3) (0,4) (0,1,2) (1,2,3,4) {7,5,12,7,2}
[.###.#] (0,1,2,3,4) (0,3,4) (0,1,2,4,5) (1,2) {10,11,11,5,10,5}
</code></pre></div></div>

<h1>Part 1</h1>

<p>Press the buttons to toggle the lights on/off until you achieve the target pattern. The lights are a target bit pattern (but in reverse order), and the button positions are bit positions. So, <code class="language-plaintext highlighter-rouge">(1,3)</code> means toggle bits 1 and 3. The problem then becomes a breadth-first search through all the possible options. Starting at 0, xor that once for each button, then xor each of those with all the buttons again. This width grows quickly, but there aren’t many bit positions, so it only takes a few iterations to cover all the possible values. I used a set of integers to store the values at each iteration.</p>

<h1>Part 2</h1>

<p>In part two, there are n counters set to zero; you need to increment the counters until you get to the values specified in the final field of the input data. Pressing button <code class="language-plaintext highlighter-rouge">(1,3)</code> increments counters 1 and 3 by one. You might view this as an extension of the first problem, but since the counter target values range from 1 to 300, the problem depth is too great to be solved naively using a BFS.</p>

<p>Looking at the first example in more detail, I rewrote it like this:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>btn | 0 1 2 3 | index
----+---------+----
3   | 0 0 0 1 | 5
1,3 | 0 1 0 1 | 4
2   | 0 0 1 0 | 3
2,3 | 0 0 1 1 | 2
0,2 | 1 0 1 0 | 1
0,1 | 1 1 0 0 | 0
----+---------+----
    | 3 5 4 7
</code></pre></div></div>

<p>From that matrix, a set of equations can be written as</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>v0 + v1 = 3
v0 + v4 = 5
v1 + v2 + v3 = 4
v2 + v4 + v5 = 7
</code></pre></div></div>

<p>These linear equations need to be solved, and the minimum sum solution found. I used the package <a href="https://opam.ocaml.org/packages/lp/">lp</a> to do this.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">#</span><span class="n">require</span> <span class="s2">"lp"</span><span class="p">;;</span>
<span class="o">#</span><span class="n">require</span> <span class="s2">"lp-glpk"</span><span class="p">;;</span>
<span class="k">open</span> <span class="nc">Lp</span>

<span class="k">let</span> <span class="n">v</span> <span class="o">=</span> <span class="nn">Array</span><span class="p">.</span><span class="n">init</span> <span class="mi">6</span> <span class="p">(</span><span class="k">fun</span> <span class="n">i</span> <span class="o">-&gt;</span> <span class="n">var</span> <span class="o">~</span><span class="n">integer</span><span class="o">:</span><span class="bp">true</span> <span class="p">(</span><span class="nn">Printf</span><span class="p">.</span><span class="n">sprintf</span> <span class="s2">"v%d"</span> <span class="n">i</span><span class="p">))</span>
  
<span class="k">let</span> <span class="n">sum</span> <span class="n">indices</span> <span class="o">=</span> 
  <span class="nn">List</span><span class="p">.</span><span class="n">fold_left</span> <span class="p">(</span><span class="k">fun</span> <span class="n">acc</span> <span class="n">i</span> <span class="o">-&gt;</span> <span class="n">acc</span> <span class="o">++</span> <span class="n">v</span><span class="o">.</span><span class="p">(</span><span class="n">i</span><span class="p">))</span> <span class="p">(</span><span class="n">c</span> <span class="mi">0</span><span class="o">.</span><span class="mi">0</span><span class="p">)</span> <span class="n">indices</span> 
  
<span class="k">let</span> <span class="n">obj</span> <span class="o">=</span> <span class="n">minimize</span> <span class="p">(</span><span class="n">sum</span> <span class="p">[</span><span class="mi">0</span><span class="p">;</span> <span class="mi">1</span><span class="p">;</span> <span class="mi">2</span><span class="p">;</span> <span class="mi">3</span><span class="p">;</span> <span class="mi">4</span><span class="p">;</span> <span class="mi">5</span><span class="p">])</span>   <span class="c">(* sum of all variables *)</span>

<span class="k">let</span> <span class="n">constraints</span> <span class="o">=</span> <span class="p">[</span>
  <span class="n">sum</span> <span class="p">[</span><span class="mi">0</span><span class="p">;</span> <span class="mi">1</span><span class="p">]</span> <span class="o">=~</span> <span class="n">c</span> <span class="mi">3</span><span class="o">.</span><span class="mi">0</span><span class="p">;</span>       <span class="c">(* v0 + v1 = 3 *)</span>
  <span class="n">sum</span> <span class="p">[</span><span class="mi">0</span><span class="p">;</span> <span class="mi">4</span><span class="p">]</span> <span class="o">=~</span> <span class="n">c</span> <span class="mi">5</span><span class="o">.</span><span class="mi">0</span><span class="p">;</span>       <span class="c">(* v0 + v4 = 5 *)</span>
  <span class="n">sum</span> <span class="p">[</span><span class="mi">1</span><span class="p">;</span> <span class="mi">2</span><span class="p">;</span> <span class="mi">3</span><span class="p">]</span> <span class="o">=~</span> <span class="n">c</span> <span class="mi">4</span><span class="o">.</span><span class="mi">0</span><span class="p">;</span>    <span class="c">(* v1 + v2 + v3 = 4 *)</span>
  <span class="n">sum</span> <span class="p">[</span><span class="mi">2</span><span class="p">;</span> <span class="mi">4</span><span class="p">;</span> <span class="mi">5</span><span class="p">]</span> <span class="o">=~</span> <span class="n">c</span> <span class="mi">7</span><span class="o">.</span><span class="mi">0</span><span class="p">;</span>    <span class="c">(* v2 + v4 + v5 = 7 *)</span>
<span class="p">]</span>
  
<span class="k">let</span> <span class="n">problem</span> <span class="o">=</span> <span class="n">make</span> <span class="n">obj</span> <span class="n">constraints</span>

<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span>
  <span class="k">match</span> <span class="nn">Lp_glpk</span><span class="p">.</span><span class="n">solve</span> <span class="n">problem</span> <span class="k">with</span>
  <span class="o">|</span> <span class="nc">Ok</span> <span class="p">(</span><span class="n">obj_val</span><span class="o">,</span> <span class="n">xs</span><span class="p">)</span> <span class="o">-&gt;</span>
      <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"Minimum: %.2f</span><span class="se">\n</span><span class="s2">"</span> <span class="n">obj_val</span><span class="p">;</span>
      <span class="nn">Array</span><span class="p">.</span><span class="n">iteri</span> <span class="p">(</span><span class="k">fun</span> <span class="n">i</span> <span class="n">var</span> <span class="o">-&gt;</span>
        <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"v%d = %.2f</span><span class="se">\n</span><span class="s2">"</span> <span class="n">i</span> <span class="p">(</span><span class="nn">PMap</span><span class="p">.</span><span class="n">find</span> <span class="n">var</span> <span class="n">xs</span><span class="p">)</span>
      <span class="p">)</span> <span class="n">v</span>
  <span class="o">|</span> <span class="nc">Error</span> <span class="n">msg</span> <span class="o">-&gt;</span>
      <span class="n">print_endline</span> <span class="n">msg</span>
</code></pre></div></div>

<p>This gives the solution as 10.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>GLPK Simplex Optimizer 5.0
4 rows, 6 columns, 10 non-zeros
      0: obj =   0.000000000e+00 inf =   1.900e+01 (4)
      4: obj =   1.000000000e+01 inf =   0.000e+00 (0)
OPTIMAL LP SOLUTION FOUND
GLPK Integer Optimizer 5.0
4 rows, 6 columns, 10 non-zeros
6 integer variables, none of which are binary
Integer optimization begins...
Long-step dual simplex will be used
+     4: mip =     not found yet &gt;=              -inf        (1; 0)
+     4: &gt;&gt;&gt;&gt;&gt;   1.000000000e+01 &gt;=   1.000000000e+01   0.0% (1; 0)
+     4: mip =   1.000000000e+01 &gt;=     tree is empty   0.0% (0; 1)
INTEGER OPTIMAL SOLUTION FOUND
Minimum: 10.00
v0 = 3.00
v1 = 0.00
v2 = 4.00
v3 = 0.00
v4 = 2.00
v5 = 1.00
</code></pre></div></div>

<p>All that is left is to sum the answer for each line of input.</p>
<h1>Day 11 - Reactor</h1>

<p>Count the number of paths to traverse a graph.</p>

<h1>Part 1</h1>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>aaa: you hhh
you: bbb ccc
bbb: ddd eee
ccc: ddd eee fff
ddd: ggg
eee: out
fff: out
ggg: out
hhh: ccc fff iii
iii: out
</code></pre></div></div>

<p>In the first part, the task was to count the number of many ways to get from <code class="language-plaintext highlighter-rouge">you</code> to <code class="language-plaintext highlighter-rouge">out</code>. There aren’t many, so a simple depth-first search worked out of the box.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">module</span> <span class="nc">Outputs</span> <span class="o">=</span> <span class="nn">Set</span><span class="p">.</span><span class="nc">Make</span> <span class="p">(</span><span class="nc">String</span><span class="p">)</span>
<span class="k">module</span> <span class="nc">Racks</span> <span class="o">=</span> <span class="nn">Map</span><span class="p">.</span><span class="nc">Make</span> <span class="p">(</span><span class="nc">String</span><span class="p">)</span>

<span class="k">let</span> <span class="k">rec</span> <span class="n">dfs</span> <span class="o">=</span> <span class="k">function</span>
  <span class="o">|</span> <span class="s2">"out"</span> <span class="o">-&gt;</span> <span class="mi">1</span>
  <span class="o">|</span> <span class="n">r</span> <span class="o">-&gt;</span> <span class="nn">Outputs</span><span class="p">.</span><span class="n">fold</span> <span class="p">(</span><span class="k">fun</span> <span class="n">o</span> <span class="n">acc</span> <span class="o">-&gt;</span> <span class="n">acc</span> <span class="o">+</span> <span class="n">dfs</span> <span class="n">o</span><span class="p">)</span> <span class="p">(</span><span class="nn">Racks</span><span class="p">.</span><span class="n">find</span> <span class="n">r</span> <span class="n">racks</span><span class="p">)</span> <span class="mi">0</span>

<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span> <span class="n">dfs</span> <span class="s2">"you"</span> <span class="o">|&gt;</span> <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"Part 1: %i</span><span class="se">\n</span><span class="s2">"</span>
</code></pre></div></div>

<h1>Part 2</h1>

<p>The examples for the second part unusually gave new data. However, the puzzle input was the same. The new example data removed the <code class="language-plaintext highlighter-rouge">you</code> node and added an <code class="language-plaintext highlighter-rouge">svr</code> node. The question is now, how many ways from <code class="language-plaintext highlighter-rouge">svr</code> to <code class="language-plaintext highlighter-rouge">out</code>, but passing through <code class="language-plaintext highlighter-rouge">fft</code> and <code class="language-plaintext highlighter-rouge">dac</code>?</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>svr: aaa bbb
aaa: fft
fft: ccc
bbb: tty
tty: ccc
ccc: ddd eee
ddd: hub
hub: fff
eee: dac
dac: fff
fff: ggg hhh
ggg: out
hhh: out
</code></pre></div></div>

<p>On my actual dataset, the number of ways from <code class="language-plaintext highlighter-rouge">svr</code> to <code class="language-plaintext highlighter-rouge">out</code> was vast (45 quadrillion), so we definitely need memoisation. The key here was to realise that it was a DAG and so either <code class="language-plaintext highlighter-rouge">dac</code> to <code class="language-plaintext highlighter-rouge">fft</code> was possible or <code class="language-plaintext highlighter-rouge">fft</code> to <code class="language-plaintext highlighter-rouge">dac</code> was possible, but not both.</p>

<p>Using a DFS I calculated the number of paths between the key components and simplied the graph to four nodes. Since <code class="language-plaintext highlighter-rouge">dac</code> to <code class="language-plaintext highlighter-rouge">fft</code> has zero paths, the path must be <code class="language-plaintext highlighter-rouge">svr</code> to <code class="language-plaintext highlighter-rouge">fft</code> to <code class="language-plaintext highlighter-rouge">dac</code> to <code class="language-plaintext highlighter-rouge">out</code>. Thus the solution is <code class="language-plaintext highlighter-rouge">1 * 1 * 2 = 2</code>.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>                   ┌─────┐
                   │ svr │
                   └──┬──┘
            ┌─────────┴─────────┐
            │                   │
          2 │                   │ 1
            │                   │
            ▼         0         ▼
         ┌─────┐ ──────────► ┌─────┐
         │ dac │      1      │ fft │
         └──┬──┘ ◄────────── └──┬──┘
            │                   │
          2 │                   │ 4
            │                   │
            │      ┌─────┐      │
            └────► │ out │ ◄────┘
                   └─────┘
</code></pre></div></div>

<h1>Day 12 - Christmas Tree Farm</h1>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>0:
###
##.
##.

1:
###
##.
.##

2:
.##
###
##.

3:
##.
###
##.

4:
###
#..
###

5:
###
.#.
###

4x4: 0 0 0 0 2 0
12x5: 1 0 1 0 2 2
12x5: 1 0 1 0 3 2
</code></pre></div></div>

<p>This is a packing problem. Given this input, <code class="language-plaintext highlighter-rouge">12x5: 1 0 1 0 2 2</code>, take a 12x5 grid and try to place 1 copy of shape 0, 1 copy of shape 2, 2 copies each of shapes 4 and 5.</p>

<p>On face value, this is a variation on the pentominoes problem, and the packing does not need to be complete. Fortunately, I looked at the real dataset before coding up a depth-first search to place the objects.</p>

<p>My first line of actual input is <code class="language-plaintext highlighter-rouge">45x41: 52 43 45 41 47 59</code>, still with 3x3 shapes to be placed. This is a massive problem space. Google has shown that Knuth’s Dancing Links is a common approach for this, and OCaml/opam has a <a href="https://opam.ocaml.org/packages/combine/">combine</a> package that implements this. I read the input data and passed it to the library to solve. However, the problem was too large.</p>

<p>As there are so many ways to pack the shapes, may there always be a solution at this scale? I used a simplistic area calculation to try this. I calculated the area of each shape, multiplied it by the number of copies and compared it to the area of the grid. Rightly or wrongly, this gave the correct answer to the problem on the real dataset (but not on the test input)</p>
