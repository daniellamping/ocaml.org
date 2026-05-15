---
title: 'Learning OCaml: Working with Records'
description: "Records are one of those things that look almost identical across ML-family
  languages, so I didn\u2019t expect many surprises when I started using them in OCaml.
  For the most part I was right \u2013 but there were a few things worth noting, especially
  if you\u2019re coming from a language where records/structs are mutable by default."
url: https://batsov.com/articles/2026/03/01/working-with-ocaml-records/
date: 2026-03-01T09:30:00-00:00
preview_image: https://batsov.com/assets/img/og-image.png
authors:
- Bozhidar Batsov
source:
ignore:
---

<p>Records are one of those things that look almost identical across ML-family
languages, so I didn’t expect many surprises when I started using them in
OCaml. For the most part I was right – but there were a few things worth
noting, especially if you’re coming from a language where records/structs
are mutable by default.</p>

<p>Let’s explore records using a fun data model – superheroes:</p>

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
  <span class="n">first_appearance</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
<span class="p">}</span>
</pre></td></tr></tbody></table></code></pre></div></div>



<h2>Creating Records</h2>

<p>Creating a record is straightforward – list all fields with their values:</p>

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
</pre></td></tr></tbody></table></code></pre></div></div>

<p>Note that you don’t specify the type anywhere – the compiler infers it from
the field names. This works great when field names are unique, but can cause
trouble when two record types in scope share a field name:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">hero</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span> <span class="n">power</span> <span class="o">:</span> <span class="kt">string</span> <span class="p">}</span>
<span class="k">type</span> <span class="n">villain</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span> <span class="n">evil_plan</span> <span class="o">:</span> <span class="kt">string</span> <span class="p">}</span>

<span class="c">(* Which type is this? The compiler picks the most recently defined one -- villain *)</span>
<span class="k">let</span> <span class="n">mystery</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span> <span class="o">=</span> <span class="s2">"Enigma"</span><span class="p">;</span> <span class="n">power</span> <span class="o">=</span> <span class="s2">"puzzles"</span> <span class="p">}</span>
<span class="c">(* Error: Unbound record field power *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>You can disambiguate by annotating the type:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">h</span> <span class="o">:</span> <span class="n">hero</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span> <span class="o">=</span> <span class="s2">"Enigma"</span><span class="p">;</span> <span class="n">power</span> <span class="o">=</span> <span class="s2">"puzzles"</span> <span class="p">}</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>Or by prefixing a field name with the module:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">h</span> <span class="o">=</span> <span class="p">{</span> <span class="nn">Hero</span><span class="p">.</span><span class="n">name</span> <span class="o">=</span> <span class="s2">"Enigma"</span><span class="p">;</span> <span class="n">power</span> <span class="o">=</span> <span class="s2">"puzzles"</span> <span class="p">}</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<blockquote class="prompt-tip">
  <p>In practice, this ambiguity rarely bites you because OCaml’s module system
naturally separates types into different scopes. The idiomatic approach is
to define one main record type per module (named <code class="language-plaintext highlighter-rouge">t</code> by convention), which
avoids collisions entirely.</p>
</blockquote>

<h2>Accessing Fields</h2>

<p>Field access uses the usual dot notation:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">name</span> <span class="o">=</span> <span class="n">batman</span><span class="o">.</span><span class="n">name</span>               <span class="c">(* "Bruce Wayne" *)</span>
<span class="k">let</span> <span class="n">year</span> <span class="o">=</span> <span class="n">batman</span><span class="o">.</span><span class="n">first_appearance</span>   <span class="c">(* 1939 *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>You can use fields directly in expressions:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">is_golden_age</span> <span class="n">hero</span> <span class="o">=</span> <span class="n">hero</span><span class="o">.</span><span class="n">first_appearance</span> <span class="o">&lt;</span> <span class="mi">1956</span>
<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span> <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"%s: golden age = %b</span><span class="se">\n</span><span class="s2">"</span>
  <span class="n">batman</span><span class="o">.</span><span class="n">alias</span> <span class="p">(</span><span class="n">is_golden_age</span> <span class="n">batman</span><span class="p">)</span>
<span class="c">(* Batman: golden age = true *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<h2>Functional Updates</h2>

<p>Records are immutable by default in OCaml. You can’t modify a field in
place – instead, you create a new record with some fields changed using the
<code class="language-plaintext highlighter-rouge">with</code> keyword:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">upgraded_batman</span> <span class="o">=</span> <span class="p">{</span> <span class="n">batman</span> <span class="k">with</span>
  <span class="n">powers</span> <span class="o">=</span> <span class="p">[</span><span class="nc">Gadgets</span><span class="p">;</span> <span class="nc">SuperStrength</span><span class="p">];</span>
  <span class="n">strength</span> <span class="o">=</span> <span class="nc">Enhanced</span><span class="p">;</span>
<span class="p">}</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>This creates a new <code class="language-plaintext highlighter-rouge">superhero</code> record that’s a copy of <code class="language-plaintext highlighter-rouge">batman</code> with only
the specified fields changed. The original is untouched. If you’ve used
Haskell’s record update syntax or Erlang’s map update syntax, this will feel
familiar.</p>

<p>Functional updates are especially handy when your records have many fields
and you only want to change one or two:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">justice_league</span> <span class="o">=</span> <span class="p">[</span><span class="n">batman</span><span class="p">;</span> <span class="n">superman</span><span class="p">]</span>

<span class="c">(* Promote everyone to Cosmic strength *)</span>
<span class="k">let</span> <span class="n">cosmic_league</span> <span class="o">=</span>
  <span class="nn">List</span><span class="p">.</span><span class="n">map</span> <span class="p">(</span><span class="k">fun</span> <span class="n">h</span> <span class="o">-&gt;</span> <span class="p">{</span> <span class="n">h</span> <span class="k">with</span> <span class="n">strength</span> <span class="o">=</span> <span class="nc">Cosmic</span> <span class="p">})</span> <span class="n">justice_league</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<h2>Mutable Fields</h2>

<p>While records are immutable by default, OCaml lets you mark individual
fields as <code class="language-plaintext highlighter-rouge">mutable</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">battle_stats</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">hero</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="k">mutable</span> <span class="n">wins</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
  <span class="k">mutable</span> <span class="n">losses</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
<span class="p">}</span>

<span class="k">let</span> <span class="n">stats</span> <span class="o">=</span> <span class="p">{</span> <span class="n">hero</span> <span class="o">=</span> <span class="s2">"Batman"</span><span class="p">;</span> <span class="n">wins</span> <span class="o">=</span> <span class="mi">0</span><span class="p">;</span> <span class="n">losses</span> <span class="o">=</span> <span class="mi">0</span> <span class="p">}</span>
<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span> <span class="n">stats</span><span class="o">.</span><span class="n">wins</span> <span class="o">&lt;-</span> <span class="n">stats</span><span class="o">.</span><span class="n">wins</span> <span class="o">+</span> <span class="mi">1</span>   <span class="c">(* wins is now 1 *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">&lt;-</code> operator mutates the field in place. Note that only fields
explicitly marked <code class="language-plaintext highlighter-rouge">mutable</code> can be changed this way – trying to mutate an
immutable field is a compile error.</p>

<p>I’d use mutable fields sparingly – they’re there when you need them for
performance-critical code, but immutable records with functional updates are
generally the idiomatic approach in OCaml.</p>

<h2>Pattern Matching on Records</h2>

<p>Like everything in OCaml, records work with pattern matching. You can
destructure them directly in function arguments:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">describe</span> <span class="p">{</span> <span class="n">alias</span><span class="p">;</span> <span class="n">strength</span><span class="p">;</span> <span class="n">first_appearance</span><span class="p">;</span> <span class="n">_</span> <span class="p">}</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">strength_str</span> <span class="o">=</span> <span class="k">match</span> <span class="n">strength</span> <span class="k">with</span>
    <span class="o">|</span> <span class="nc">Human</span> <span class="o">-&gt;</span> <span class="s2">"human-level"</span>
    <span class="o">|</span> <span class="nc">Enhanced</span> <span class="o">-&gt;</span> <span class="s2">"enhanced"</span>
    <span class="o">|</span> <span class="nc">Superhuman</span> <span class="o">-&gt;</span> <span class="s2">"superhuman"</span>
    <span class="o">|</span> <span class="nc">Cosmic</span> <span class="o">-&gt;</span> <span class="s2">"cosmic"</span>
  <span class="k">in</span>
  <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"%s (%d) - %s strength</span><span class="se">\n</span><span class="s2">"</span>
    <span class="n">alias</span> <span class="n">first_appearance</span> <span class="n">strength_str</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">_</code> in the pattern tells the compiler you’re intentionally ignoring the
other fields. Without it, you’d get a warning about inexhaustive field
patterns.</p>

<p>You can also pattern match in <code class="language-plaintext highlighter-rouge">let</code> bindings:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="p">{</span> <span class="n">alias</span><span class="p">;</span> <span class="n">powers</span><span class="p">;</span> <span class="n">_</span> <span class="p">}</span> <span class="o">=</span> <span class="n">batman</span> <span class="k">in</span>
<span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"%s has %d powers</span><span class="se">\n</span><span class="s2">"</span> <span class="n">alias</span> <span class="p">(</span><span class="nn">List</span><span class="p">.</span><span class="n">length</span> <span class="n">powers</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>And in <code class="language-plaintext highlighter-rouge">match</code> expressions for more complex logic:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">classify</span> <span class="o">=</span> <span class="k">function</span>
  <span class="o">|</span> <span class="p">{</span> <span class="n">strength</span> <span class="o">=</span> <span class="nc">Cosmic</span><span class="p">;</span> <span class="n">_</span> <span class="p">}</span> <span class="o">-&gt;</span> <span class="s2">"overpowered"</span>
  <span class="o">|</span> <span class="p">{</span> <span class="n">strength</span> <span class="o">=</span> <span class="nc">Human</span><span class="p">;</span> <span class="n">powers</span><span class="p">;</span> <span class="n">_</span> <span class="p">}</span> <span class="k">when</span> <span class="nn">List</span><span class="p">.</span><span class="n">length</span> <span class="n">powers</span> <span class="o">&gt;</span> <span class="mi">0</span> <span class="o">-&gt;</span>
    <span class="s2">"resourceful"</span>
  <span class="o">|</span> <span class="p">{</span> <span class="n">strength</span> <span class="o">=</span> <span class="nc">Human</span><span class="p">;</span> <span class="n">_</span> <span class="p">}</span> <span class="o">-&gt;</span> <span class="s2">"ordinary"</span>
  <span class="o">|</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="s2">"super"</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<h2>Field Punning</h2>

<p>You might have noticed the shorthand in the examples above –
<code class="language-plaintext highlighter-rouge">{ alias; strength; _ }</code> instead of <code class="language-plaintext highlighter-rouge">{ alias = alias; strength = strength; _ }</code>.
This is called “field punning” and it works both in patterns and when
constructing records:</p>

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
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">alias</span> <span class="o">=</span> <span class="s2">"Flash"</span>
<span class="k">let</span> <span class="n">name</span> <span class="o">=</span> <span class="s2">"Barry Allen"</span>
<span class="k">let</span> <span class="n">powers</span> <span class="o">=</span> <span class="p">[</span><span class="nc">Speed</span><span class="p">]</span>
<span class="k">let</span> <span class="n">strength</span> <span class="o">=</span> <span class="nc">Superhuman</span>
<span class="k">let</span> <span class="n">first_appearance</span> <span class="o">=</span> <span class="mi">1956</span>

<span class="c">(* these are equivalent *)</span>
<span class="k">let</span> <span class="n">flash_v1</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span> <span class="o">=</span> <span class="n">name</span><span class="p">;</span> <span class="n">alias</span> <span class="o">=</span> <span class="n">alias</span><span class="p">;</span> <span class="n">powers</span> <span class="o">=</span> <span class="n">powers</span><span class="p">;</span>
                 <span class="n">strength</span> <span class="o">=</span> <span class="n">strength</span><span class="p">;</span> <span class="n">first_appearance</span> <span class="o">=</span> <span class="n">first_appearance</span> <span class="p">}</span>
<span class="k">let</span> <span class="n">flash_v2</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span><span class="p">;</span> <span class="n">alias</span><span class="p">;</span> <span class="n">powers</span><span class="p">;</span> <span class="n">strength</span><span class="p">;</span> <span class="n">first_appearance</span> <span class="p">}</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>If you’re coming from JavaScript, this is the same idea as ES6’s shorthand
property names in object literals. It’s particularly nice when you’re
constructing a record from variables that already have the right names.</p>

<h2>Records and Modules</h2>

<p>The idiomatic OCaml pattern is to define one main type per module, named <code class="language-plaintext highlighter-rouge">t</code>:</p>

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
</pre></td><td class="rouge-code"><pre><span class="c">(* superhero.ml *)</span>
<span class="k">type</span> <span class="n">power</span> <span class="o">=</span> <span class="nc">Flight</span> <span class="o">|</span> <span class="nc">SuperStrength</span> <span class="o">|</span> <span class="nc">Telepathy</span> <span class="o">|</span> <span class="nc">Speed</span> <span class="o">|</span> <span class="nc">Gadgets</span>
<span class="k">type</span> <span class="n">strength</span> <span class="o">=</span> <span class="nc">Human</span> <span class="o">|</span> <span class="nc">Enhanced</span> <span class="o">|</span> <span class="nc">Superhuman</span> <span class="o">|</span> <span class="nc">Cosmic</span>

<span class="k">type</span> <span class="n">t</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">alias</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">powers</span> <span class="o">:</span> <span class="n">power</span> <span class="kt">list</span><span class="p">;</span>
  <span class="n">strength</span> <span class="o">:</span> <span class="n">strength</span><span class="p">;</span>
  <span class="n">first_appearance</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
<span class="p">}</span>

<span class="k">let</span> <span class="n">create</span> <span class="o">~</span><span class="n">name</span> <span class="o">~</span><span class="n">alias</span> <span class="o">~</span><span class="n">powers</span> <span class="o">~</span><span class="n">strength</span> <span class="o">~</span><span class="n">first_appearance</span> <span class="o">=</span>
  <span class="p">{</span> <span class="n">name</span><span class="p">;</span> <span class="n">alias</span><span class="p">;</span> <span class="n">powers</span><span class="p">;</span> <span class="n">strength</span><span class="p">;</span> <span class="n">first_appearance</span> <span class="p">}</span>

<span class="k">let</span> <span class="n">is_golden_age</span> <span class="n">t</span> <span class="o">=</span> <span class="n">t</span><span class="o">.</span><span class="n">first_appearance</span> <span class="o">&lt;</span> <span class="mi">1956</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>This convention means you refer to the type as <code class="language-plaintext highlighter-rouge">Superhero.t</code> from outside
the module, and field access reads naturally as <code class="language-plaintext highlighter-rouge">hero.Superhero.alias</code> (or
just <code class="language-plaintext highlighter-rouge">hero.alias</code> if you’ve opened the module). It also avoids the field
name collision problem entirely, since each module has its own namespace.</p>

<h2>Printing Records</h2>

<p>You can’t just <code class="language-plaintext highlighter-rouge">print</code> a record in OCaml – there’s no generic print that
works on any type. The easiest solution is <code class="language-plaintext highlighter-rouge">[@@deriving show]</code> from
<code class="language-plaintext highlighter-rouge">ppx_deriving</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">superhero</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">alias</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">powers</span> <span class="o">:</span> <span class="n">power</span> <span class="kt">list</span><span class="p">;</span>
<span class="p">}</span> <span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">show</span><span class="p">]</span>

<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span> <span class="n">print_endline</span> <span class="p">(</span><span class="n">show_superhero</span> <span class="n">batman</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>I wrote a <a href="https://batsov.com/articles/2026/03/01/printing-data-in-ocaml/">dedicated article</a>
on printing data structures if you want the full story.</p>

<h2>Records vs Tuples</h2>

<p>One question that comes up for beginners: when should you use a record
instead of a tuple?</p>

<p>Tuples are great for quick, throwaway groupings – returning two values from
a function, for example. But once you have more than two or three fields, or
when the meaning of each position isn’t obvious, records are almost always
better:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
</pre></td><td class="rouge-code"><pre><span class="c">(* Tuple -- what does each position mean? *)</span>
<span class="k">let</span> <span class="n">hero</span> <span class="o">=</span> <span class="p">(</span><span class="s2">"Bruce Wayne"</span><span class="o">,</span> <span class="s2">"Batman"</span><span class="o">,</span> <span class="mi">1939</span><span class="p">)</span>

<span class="c">(* Record -- self-documenting *)</span>
<span class="k">let</span> <span class="n">hero</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span> <span class="o">=</span> <span class="s2">"Bruce Wayne"</span><span class="p">;</span> <span class="n">alias</span> <span class="o">=</span> <span class="s2">"Batman"</span><span class="p">;</span> <span class="n">first_appearance</span> <span class="o">=</span> <span class="mi">1939</span> <span class="p">}</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>Records also give you pattern matching with named fields, functional
updates with <code class="language-plaintext highlighter-rouge">with</code>, and better error messages from the compiler. If you
find yourself reaching for a tuple with more than 2-3 elements, it’s
probably time for a record.</p>

<h2>Further Reading</h2>

<ul>
  <li><a href="https://dev.realworldocaml.org/records.html">Records</a> – the Real World
OCaml chapter on records. Goes deeper into functional updates, first-class
fields (via <code class="language-plaintext highlighter-rouge">ppx_fields_conv</code>), and the interaction between records and modules.</li>
  <li><a href="https://ocaml.org/docs/basic-data-types">Basic Data Types and Pattern Matching</a> –
the official ocaml.org guide covering records alongside other core data types.</li>
</ul>

<p>That’s all I have for you today. Keep hacking!</p>
