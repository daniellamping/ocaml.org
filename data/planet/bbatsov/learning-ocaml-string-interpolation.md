---
title: 'Learning OCaml: String Interpolation'
description: "Most programming languages I\u2019ve used have some form of string interpolation.
  Ruby has \"Hello, #{name}!\", Python has f-strings, JavaScript has template literals,
  even Haskell has a few popular interpolation libraries. It\u2019s one of those small
  conveniences you don\u2019t think about until it\u2019s gone."
url: https://batsov.com/articles/2026/03/03/learning-ocaml-string-interpolation/
date: 2026-03-03T10:00:00-00:00
preview_image: https://batsov.com/assets/img/og-image.png
authors:
- Bozhidar Batsov
source:
ignore:
---

<p>Most programming languages I’ve used have some form of string
interpolation. Ruby has <code class="language-plaintext highlighter-rouge">"Hello, #{name}!"</code>, Python has f-strings,
JavaScript has template literals, even Haskell has a few popular
interpolation libraries. It’s one of those small conveniences you don’t
think about until it’s gone.</p>

<p>OCaml doesn’t have built-in string interpolation. And here’s the funny
thing – I didn’t even notice when I was first learning the language. Looking
back at my <a href="https://batsov.com/articles/2022/08/29/ocaml-at-first-glance/">first impressions</a>
article, I complained about the comment syntax, the semicolons in lists, the
lack of list comprehensions, and a dozen other things – but never once about
string interpolation. I was happily concatenating strings with <code class="language-plaintext highlighter-rouge">^</code> and using
<code class="language-plaintext highlighter-rouge">Printf.sprintf</code> without giving it a second thought.</p>

<p>I only started thinking about this while working on my
<a href="https://batsov.com/articles/2026/03/03/ppx-for-mere-mortals/">PPX article</a> and going
through the catalog of popular PPX libraries. That’s when I stumbled upon
<code class="language-plaintext highlighter-rouge">ppx_string</code> and thought “wait, why <em>doesn’t</em> OCaml have interpolation?”</p>



<h2>Why OCaml Doesn’t Have String Interpolation</h2>

<p>The short answer: OCaml has no way to generically convert a value to a
string. There’s no universal <code class="language-plaintext highlighter-rouge">to_string</code> method, no <code class="language-plaintext highlighter-rouge">Show</code> typeclass, no
runtime reflection that would let the language figure out how to stringify
an arbitrary expression inside a string literal.</p>

<p>In Ruby, every object responds to <code class="language-plaintext highlighter-rouge">.to_s</code>. In Python, everything has
<code class="language-plaintext highlighter-rouge">__str__</code>. These languages can interpolate anything because there’s always
a fallback conversion available at runtime. OCaml’s type information is
erased at compile time, so the compiler would need to know <em>at compile
time</em> which conversion function to call for each interpolated expression –
and the language has no mechanism for that.<sup><a href="https://batsov.com/feeds/OCaml.xml#fn:1" class="footnote" rel="footnote" role="doc-noteref">1</a></sup></p>

<p>OCaml does have <code class="language-plaintext highlighter-rouge">Printf.sprintf</code>, which is actually quite nice and
type-safe:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">greeting</span> <span class="n">name</span> <span class="n">age</span> <span class="o">=</span>
  <span class="nn">Printf</span><span class="p">.</span><span class="n">sprintf</span> <span class="s2">"Hello, %s! You are %d years old."</span> <span class="n">name</span> <span class="n">age</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The format string <code class="language-plaintext highlighter-rouge">"%s ... %d"</code> is statically checked by the compiler –
if you pass an <code class="language-plaintext highlighter-rouge">int</code> where <code class="language-plaintext highlighter-rouge">%s</code> expects a string, you get a compile-time
error, not a runtime crash. That’s genuinely better than what most
dynamically typed languages offer. But it’s not interpolation – the
values aren’t inline in the string, and for complex expressions it gets
unwieldy fast.</p>

<p>There’s also plain string concatenation with <code class="language-plaintext highlighter-rouge">^</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">greeting</span> <span class="n">name</span> <span class="n">age</span> <span class="o">=</span>
  <span class="s2">"Hello, "</span> <span class="o">^</span> <span class="n">name</span> <span class="o">^</span> <span class="s2">"! You are "</span> <span class="o">^</span> <span class="n">string_of_int</span> <span class="n">age</span> <span class="o">^</span> <span class="s2">" years old."</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>This works, but it’s ugly and error-prone for anything beyond trivial cases.</p>

<h2>ppx_string to the Rescue</h2>

<p><a href="https://github.com/janestreet/ppx_string">ppx_string</a> is a Jane Street PPX
that adds string interpolation to OCaml at compile time. The basic usage is
straightforward:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">greeting</span> <span class="n">name</span> <span class="o">=</span>
  <span class="p">[</span><span class="o">%</span><span class="kt">string</span> <span class="s2">"Hello, %{name}!"</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>For non-string types, you specify the module whose <code class="language-plaintext highlighter-rouge">to_string</code> function
should be used:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">status</span> <span class="n">name</span> <span class="n">age</span> <span class="n">score</span> <span class="o">=</span>
  <span class="p">[</span><span class="o">%</span><span class="kt">string</span> <span class="s2">"Player %{name}, age %{age#Int}, score %{score#Float}"</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">#Int</code> suffix tells the PPX to call <code class="language-plaintext highlighter-rouge">Int.to_string</code> on <code class="language-plaintext highlighter-rouge">age</code>, and
<code class="language-plaintext highlighter-rouge">#Float</code> calls <code class="language-plaintext highlighter-rouge">Float.to_string</code> on <code class="language-plaintext highlighter-rouge">score</code>. Note that <code class="language-plaintext highlighter-rouge">Int.to_string</code>,
<code class="language-plaintext highlighter-rouge">Float.to_string</code>, etc. are conventions from Jane Street’s <code class="language-plaintext highlighter-rouge">Base</code>/<code class="language-plaintext highlighter-rouge">Core</code>
libraries – OCaml’s <code class="language-plaintext highlighter-rouge">Stdlib</code> uses <code class="language-plaintext highlighter-rouge">string_of_int</code>, <code class="language-plaintext highlighter-rouge">string_of_float</code> and
so on, which won’t work with the <code class="language-plaintext highlighter-rouge">#Module</code> syntax. This is another reason
<code class="language-plaintext highlighter-rouge">ppx_string</code> really only makes sense within the Jane Street ecosystem.</p>

<p>Any module that exposes a <code class="language-plaintext highlighter-rouge">to_string</code> function works here – including your
own:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
</pre></td><td class="rouge-code"><pre><span class="k">module</span> <span class="nc">Player</span> <span class="o">=</span> <span class="k">struct</span>
  <span class="k">type</span> <span class="n">t</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span> <span class="n">score</span> <span class="o">:</span> <span class="kt">int</span> <span class="p">}</span>
  <span class="k">let</span> <span class="n">to_string</span> <span class="n">p</span> <span class="o">=</span> <span class="nn">Printf</span><span class="p">.</span><span class="n">sprintf</span> <span class="s2">"%s (%d)"</span> <span class="n">p</span><span class="o">.</span><span class="n">name</span> <span class="n">p</span><span class="o">.</span><span class="n">score</span>
<span class="k">end</span>

<span class="k">let</span> <span class="n">announce</span> <span class="n">player</span> <span class="o">=</span>
  <span class="p">[</span><span class="o">%</span><span class="kt">string</span> <span class="s2">"Next up: %{player#Player}"</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>You can also use arbitrary expressions inside the interpolation braces:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">summary</span> <span class="n">items</span> <span class="o">=</span>
  <span class="p">[</span><span class="o">%</span><span class="kt">string</span> <span class="s2">"Found %{List.length items |&gt; Int.to_string} items"</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>Though at that point you might be better off with a <code class="language-plaintext highlighter-rouge">let</code> binding or
<code class="language-plaintext highlighter-rouge">sprintf</code> for readability.</p>

<h3>Tips and Limitations</h3>

<p>A few practical things worth knowing:</p>

<ul>
  <li>You need the <code class="language-plaintext highlighter-rouge">pps</code> stanza in your dune file:
    <pre><code class="language-dune">(library
 (name mylib)
 (preprocess (pps ppx_string)))
</code></pre>
  </li>
  <li>String values interpolate directly, everything else needs a conversion
suffix. Unlike Ruby where <code class="language-plaintext highlighter-rouge">.to_s</code> is called implicitly, <code class="language-plaintext highlighter-rouge">ppx_string</code>
requires you to be explicit about non-string types. This is annoying at
first, but it’s consistent with OCaml’s philosophy of being explicit about
types.</li>
  <li>It’s a Jane Street library. If you’re already in the Jane Street
ecosystem (<code class="language-plaintext highlighter-rouge">Core</code>, <code class="language-plaintext highlighter-rouge">Base</code>, etc.), adding <code class="language-plaintext highlighter-rouge">ppx_string</code> is trivial. If you’re
not, pulling in a Jane Street dependency just for string interpolation might
feel heavy. In that case, <code class="language-plaintext highlighter-rouge">Printf.sprintf</code> is honestly fine.</li>
  <li>It doesn’t work with the <code class="language-plaintext highlighter-rouge">Format</code> module. If you’re building strings for
pretty-printing, you’ll still want <code class="language-plaintext highlighter-rouge">Format.fprintf</code> or <code class="language-plaintext highlighter-rouge">Format.asprintf</code>.
<code class="language-plaintext highlighter-rouge">ppx_string</code> is for building plain strings, not format strings.</li>
  <li>Nested interpolation doesn’t work – you can’t nest <code class="language-plaintext highlighter-rouge">%{...}</code> inside
another <code class="language-plaintext highlighter-rouge">%{...}</code>. Keep it simple.</li>
</ul>

<h2>Do You Actually Need String Interpolation?</h2>

<p>Honestly? Probably not as much as you think. I’ve been writing OCaml for a
while now without it, and it rarely bothers me. Here’s why:</p>

<ul>
  <li><strong><code class="language-plaintext highlighter-rouge">Printf.sprintf</code> is good.</strong> It’s type-safe, it’s concise enough for most
cases, and it’s available everywhere without extra dependencies.</li>
  <li><strong>Most string building in OCaml happens through <code class="language-plaintext highlighter-rouge">Format</code>.</strong> If you’re
writing pretty-printers (which you will be, thanks to <code class="language-plaintext highlighter-rouge">[@@deriving show]</code>),
you’re using <code class="language-plaintext highlighter-rouge">Format.fprintf</code>, not string concatenation or interpolation.</li>
  <li><strong>OCaml code tends to be more compute-heavy than string-heavy.</strong> Compared
to, say, a Rails app or a shell script, the typical OCaml program just
doesn’t build that many ad-hoc strings.</li>
</ul>

<p>That said, when you <em>do</em> need to build a lot of human-readable strings –
error messages, log output, CLI formatting – interpolation is genuinely
nicer than <code class="language-plaintext highlighter-rouge">sprintf</code>. If you’re in the Jane Street ecosystem, there’s no
reason not to use <code class="language-plaintext highlighter-rouge">ppx_string</code>.</p>

<h2>Closing Thoughts</h2>

<p>The lack of string interpolation in OCaml is one of those things that sounds
worse than it actually is. In practice, <code class="language-plaintext highlighter-rouge">Printf.sprintf</code> and <code class="language-plaintext highlighter-rouge">Format</code> cover
the vast majority of use cases, and the code you write with them is arguably
clearer about types than magical interpolation would be.</p>

<p>It’s also a nice example of OCaml’s general philosophy: keep the language
core small, provide solid primitives (<code class="language-plaintext highlighter-rouge">Printf</code>, <code class="language-plaintext highlighter-rouge">Format</code>), and let the PPX
ecosystem fill in the syntactic sugar for those who want it. The same pattern
plays out with <code class="language-plaintext highlighter-rouge">[@@deriving show]</code> for printing, <code class="language-plaintext highlighter-rouge">ppx_let</code> for monadic
syntax, and many other conveniences.</p>

<p>Will OCaml ever get built-in string interpolation? Maybe. There have been
discussions on the forums over the years, and the language did absorb binding
operators (<code class="language-plaintext highlighter-rouge">let*</code>, <code class="language-plaintext highlighter-rouge">let+</code>) from the PPX world. But I wouldn’t hold my breath
– and honestly, I’m not sure I’d even notice if it landed.</p>

<p>That’s all I have for you today. Keep hacking!</p>

<div class="footnotes" role="doc-endnotes">
  <ol>
    <li>
      <p>This is the same fundamental problem that makes <a href="https://batsov.com/articles/2026/03/01/printing-data-in-ocaml/">printing data structures</a> harder than in dynamically typed languages.&nbsp;<a href="https://batsov.com/feeds/OCaml.xml#fnref:1" class="reversefootnote" role="doc-backlink">↩︎</a></p>
    </li>
  </ol>
</div>
