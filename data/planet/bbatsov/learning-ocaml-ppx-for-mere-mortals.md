---
title: 'Learning OCaml: PPX for Mere Mortals'
description: 'When I started learning OCaml I kept running into code like this:'
url: https://batsov.com/articles/2026/03/03/ppx-for-mere-mortals/
date: 2026-03-03T08:00:00-00:00
preview_image: https://batsov.com/assets/img/og-image.png
authors:
- Bozhidar Batsov
source:
ignore:
---

<p>When I started learning OCaml I kept running into code like this:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">person</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">age</span>  <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
<span class="p">}</span> <span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">show</span><span class="o">,</span> <span class="n">eq</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>My first reaction was “what the hell is <code class="language-plaintext highlighter-rouge">[@@deriving show, eq]</code>?” Coming from
languages like Ruby and Clojure, where metaprogramming is either built into the
runtime (reflection) or baked into the language itself (macros),
OCaml’s approach felt alien. There’s no runtime reflection, no macro system in the
Lisp sense – just this mysterious <code class="language-plaintext highlighter-rouge">@@</code> syntax that somehow generates code at
compile time.</p>

<p>That mystery is <strong>PPX</strong> (PreProcessor eXtensions), and once you understand it,
a huge chunk of the OCaml ecosystem suddenly makes a lot more sense. This article
is my attempt to demystify PPX for people like me – developers who want to
<em>use</em> PPX effectively without necessarily becoming PPX authors themselves.</p>



<h2>What Is PPX and Why Should You Care?</h2>

<p>OCaml is a statically typed language with no runtime reflection. That means you
can’t do things like “iterate over all fields of a record at runtime” or
“automatically serialize any type to JSON.” The type information simply isn’t
available at runtime – it’s erased during compilation. One of my biggest
frustrations as a newcomer was not being able to just print arbitrary data for
debugging – there’s no generic <code class="language-plaintext highlighter-rouge">print</code> or <code class="language-plaintext highlighter-rouge">inspect</code> that works on any type.
That frustration was probably my first real interaction with PPX.</p>

<p>PPX solves this by generating code at <strong>compile time</strong>. When the OCaml compiler
parses your source code, it builds an Abstract Syntax Tree (AST) – a tree data
structure that represents the syntactic structure of your program. PPX rewriters
are programs that receive this AST, transform it, and return a modified AST back
to the compiler. The compiler then continues as if you had written the generated
code by hand.</p>

<p>In practical terms, this means that when you write:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">color</span> <span class="o">=</span> <span class="nc">Red</span> <span class="o">|</span> <span class="nc">Green</span> <span class="o">|</span> <span class="nc">Blue</span>
<span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">show</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The PPX rewriter generates something like this behind the scenes:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">color</span> <span class="o">=</span> <span class="nc">Red</span> <span class="o">|</span> <span class="nc">Green</span> <span class="o">|</span> <span class="nc">Blue</span>

<span class="k">let</span> <span class="n">pp_color</span> <span class="n">fmt</span> <span class="o">=</span> <span class="k">function</span>    <span class="c">(* uses OCaml's Format module for pretty-printing *)</span>
  <span class="o">|</span> <span class="nc">Red</span> <span class="o">-&gt;</span> <span class="nn">Format</span><span class="p">.</span><span class="n">fprintf</span> <span class="n">fmt</span> <span class="s2">"Red"</span>
  <span class="o">|</span> <span class="nc">Green</span> <span class="o">-&gt;</span> <span class="nn">Format</span><span class="p">.</span><span class="n">fprintf</span> <span class="n">fmt</span> <span class="s2">"Green"</span>
  <span class="o">|</span> <span class="nc">Blue</span> <span class="o">-&gt;</span> <span class="nn">Format</span><span class="p">.</span><span class="n">fprintf</span> <span class="n">fmt</span> <span class="s2">"Blue"</span>

<span class="k">let</span> <span class="n">show_color</span> <span class="n">x</span> <span class="o">=</span> <span class="nn">Format</span><span class="p">.</span><span class="n">asprintf</span> <span class="s2">"%a"</span> <span class="n">pp_color</span> <span class="n">x</span>  <span class="c">(* returns the formatted string *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>You get a pretty-printer for free, derived from the type definition. No
boilerplate, no manual work, and it stays in sync with your type automatically.</p>

<p>If you’ve used Rust’s <code class="language-plaintext highlighter-rouge">#[derive(Debug, PartialEq)]</code> or Haskell’s <code class="language-plaintext highlighter-rouge">deriving
(Show, Eq)</code>, the idea is very similar. The syntax is different, but the
motivation is identical – generating repetitive code from type definitions.</p>

<h2>Why Isn’t This Built into the Language?</h2>

<p>If you’re coming from Rust, you might wonder why OCaml doesn’t just have a
built-in macro system like <code class="language-plaintext highlighter-rouge">proc_macro</code>. It’s a fair question, and the answer
says a lot about OCaml’s design philosophy.</p>

<p>OCaml has always favored a <strong>small, stable language core</strong>. The compiler is
famously lean and fast, and the language team is conservative about adding
complexity to the specification. A full macro system baked into the compiler
would be a significant undertaking – it would need to be designed, specified,
maintained, and kept compatible across versions, forever.</p>

<p>Instead, OCaml took a more minimal approach: the compiler provides just two
things – <strong>extension points</strong> and <strong>attributes</strong> – as syntactic hooks in the
AST. Everything else lives in the ecosystem. The actual PPX rewriters are
ordinary OCaml programs that happen to transform ASTs. The <a href="https://github.com/ocaml-ppx/ppxlib">ppxlib</a>
framework that ties it all together is a regular library, not part of the
compiler.</p>

<p>This has some real advantages:</p>

<ul>
  <li><strong>The ecosystem can evolve independently.</strong> ppxlib can ship new features,
fix bugs, and improve APIs without waiting for a compiler release. Compare
this to Rust, where changes to the proc macro system require the full RFC
process and a compiler update.</li>
  <li><strong>Tooling stays simple.</strong> Because <code class="language-plaintext highlighter-rouge">[@@deriving show]</code> and <code class="language-plaintext highlighter-rouge">[%blob "file"]</code>
are valid OCaml syntax, every tool – editors, formatters, documentation
generators – can parse PPX-annotated code without knowing anything about
the specific PPX. The code is always syntactically valid OCaml, even before
preprocessing.</li>
  <li><strong>The compiler stays lean.</strong> No macro expander, no hygiene system, no
special compilation phases – just a hook that says “here, transform this
AST before I type-check it.”</li>
</ul>

<p>The trade-offs are real, though. Rust’s proc macros are more tightly
integrated – you get better error messages pointing at macro-generated code,
better IDE support for macro expansions, and the macro system is a documented,
stable part of the language. With PPX, you’re sometimes left staring at
cryptic type errors in generated code and reaching for <code class="language-plaintext highlighter-rouge">dune describe pp</code> to
figure out what went wrong.</p>

<p>That said, OCaml’s approach feels very OCaml – pragmatic, minimal, and
trusting the ecosystem to build what’s needed on top of a simple foundation.
And in practice, it works remarkably well.</p>

<h2>A Bit of History</h2>

<p>PPX wasn’t OCaml’s first metaprogramming system. Before PPX, there was
<strong>Camlp4</strong> (and its fork <strong>Camlp5</strong>) – a powerful but complex preprocessor that
maintained its own parser, separate from the compiler’s parser. Camlp4 could
extend OCaml’s syntax in arbitrary ways, which sounds great in theory but was a
maintenance nightmare in practice. Every OCaml release risked breaking Camlp4,
and code using Camlp4 extensions often couldn’t be processed by standard tools
like editors and documentation generators.</p>

<p>OCaml 4.02 (2014) introduced <strong>extension points</strong> and <strong>attributes</strong> directly
into the language grammar – syntactic hooks specifically designed for
preprocessor extensions. This was a much simpler and more maintainable approach:
PPX rewriters use the compiler’s own AST, the syntax is valid OCaml (so tools
can still parse your code), and the whole thing is conceptually just “AST in,
AST out.”</p>

<p>Camlp4 was officially retired in 2019. Today, the PPX ecosystem is built on
<a href="https://github.com/ocaml-ppx/ppxlib">ppxlib</a>, a unified framework that
provides a stable API across OCaml versions and handles all the plumbing for PPX
authors.</p>

<h2>Understanding the Syntax</h2>

<p>Before diving into specific libraries, let’s decode the bracket soup. PPX uses two
syntactic mechanisms built into OCaml:</p>

<h3>Extension Nodes</h3>

<p>Extension nodes are placeholders that a PPX rewriter <strong>must</strong> replace with
generated code (compilation fails if no PPX handles them):</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
</pre></td><td class="rouge-code"><pre><span class="c">(* Expression-level extension *)</span>
<span class="k">let</span> <span class="n">pos</span> <span class="o">=</span> <span class="p">[</span><span class="o">%</span><span class="n">here</span><span class="p">]</span>            <span class="c">(* replaced with current source position *)</span>
<span class="k">let</span> <span class="n">data</span> <span class="o">=</span> <span class="p">[</span><span class="o">%</span><span class="n">blob</span> <span class="s2">"icon.png"</span><span class="p">]</span> <span class="c">(* replaced with file contents as a string *)</span>

<span class="c">(* The "let%" infix syntax is sugar for expression extensions *)</span>
<span class="k">let</span><span class="o">%</span><span class="n">bind</span> <span class="n">x</span> <span class="o">=</span> <span class="n">fetch</span> <span class="bp">()</span> <span class="k">in</span>     <span class="c">(* equivalent to [%bind let x = fetch () in ...] *)</span>
<span class="n">process</span> <span class="n">x</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<h3>Attributes</h3>

<p>Attributes attach metadata to existing code. Unlike extension nodes, the
compiler silently ignores attributes that no PPX handles:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
</pre></td><td class="rouge-code"><pre><span class="c">(* [@attr] -- attaches to the nearest expression/pattern/type *)</span>
<span class="n">x</span> <span class="o">+</span> <span class="n">y</span> <span class="p">[</span><span class="o">@</span><span class="n">warning</span> <span class="s2">"-8"</span><span class="p">]</span>

<span class="c">(* [@@attr] -- attaches to the enclosing declaration *)</span>
<span class="k">type</span> <span class="n">t</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span> <span class="n">age</span> <span class="o">:</span> <span class="kt">int</span> <span class="p">}</span>
<span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">show</span><span class="p">]</span>

<span class="c">(* [@@@attr] -- floating attribute at module level *)</span>
<span class="p">[</span><span class="o">@@@</span><span class="n">warning</span> <span class="s2">"-32"</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The one you’ll see most often is <code class="language-plaintext highlighter-rouge">[@@deriving ...]</code> on type declarations. The
distinction between <code class="language-plaintext highlighter-rouge">@</code>, <code class="language-plaintext highlighter-rouge">@@</code>, and <code class="language-plaintext highlighter-rouge">@@@</code> is about scope – one <code class="language-plaintext highlighter-rouge">@</code> for the
innermost node, two for the enclosing declaration, three for the whole
module-level.</p>

<p><strong>Tip:</strong> Don’t worry about memorizing all of this upfront. In practice, you’ll
mostly use <code class="language-plaintext highlighter-rouge">[@@deriving ...]</code> and occasionally <code class="language-plaintext highlighter-rouge">[%extension ...]</code> or <code class="language-plaintext highlighter-rouge">let%...</code> –
and the specific PPX library’s documentation will tell you exactly which syntax
to use.</p>

<h2>Using PPX with Dune</h2>

<p>To use a PPX library in your project, you add it to the <code class="language-plaintext highlighter-rouge">preprocess</code> stanza in
your <code class="language-plaintext highlighter-rouge">dune</code> file:</p>

<pre><code class="language-dune">(library
 (name mylib)
 (libraries yojson)
 (preprocess (pps ppx_deriving.show ppx_deriving.eq ppx_deriving_yojson)))
</code></pre>

<p>That’s it. List all the PPX rewriters you need after <code class="language-plaintext highlighter-rouge">pps</code>, and Dune takes
care of the rest (it even combines them into a single binary for performance). For
<code class="language-plaintext highlighter-rouge">ppx_deriving</code> plugins specifically, you use dotted names like <code class="language-plaintext highlighter-rouge">ppx_deriving.show</code>.</p>

<h2>The PPX Libraries You’ll Actually Use</h2>

<p>Let’s look at the PPX libraries that cover probably 90% of real-world use cases.</p>

<h3>ppx_deriving – The Swiss Army Knife</h3>

<p><a href="https://github.com/ocaml-ppx/ppx_deriving">ppx_deriving</a> is the community’s
general-purpose deriving framework. It comes with several built-in plugins:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">point</span> <span class="o">=</span> <span class="p">{</span> <span class="n">x</span> <span class="o">:</span> <span class="kt">float</span><span class="p">;</span> <span class="n">y</span> <span class="o">:</span> <span class="kt">float</span> <span class="p">}</span>
<span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">show</span><span class="o">,</span> <span class="n">eq</span><span class="o">,</span> <span class="n">ord</span><span class="p">]</span>

<span class="c">(* Generates:
   val pp_point : Format.formatter -&gt; point -&gt; unit
   val show_point : point -&gt; string
   val equal_point : point -&gt; point -&gt; bool
   val compare_point : point -&gt; point -&gt; int *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">show</code> is the one you’ll reach for first – it’s essentially the answer to
“how do I just print this thing?” that every OCaml newcomer asks sooner or
later. The most commonly used plugins:</p>

<table>
  <thead>
    <tr>
      <th>Plugin</th>
      <th>What it generates</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">show</code></td>
      <td><code class="language-plaintext highlighter-rouge">pp_t</code> and <code class="language-plaintext highlighter-rouge">show_t</code> – pretty-printing</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">eq</code></td>
      <td><code class="language-plaintext highlighter-rouge">equal_t</code> – structural equality</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">ord</code></td>
      <td><code class="language-plaintext highlighter-rouge">compare_t</code> – total ordering</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">make</code></td>
      <td>Smart constructors with optional/default args</td>
    </tr>
  </tbody>
</table>

<p>A neat convention: if your type is named <code class="language-plaintext highlighter-rouge">t</code> (as is idiomatic in OCaml), the
generated functions drop the type name suffix – you get <code class="language-plaintext highlighter-rouge">pp</code>, <code class="language-plaintext highlighter-rouge">show</code>, <code class="language-plaintext highlighter-rouge">equal</code>,
<code class="language-plaintext highlighter-rouge">compare</code> instead of <code class="language-plaintext highlighter-rouge">pp_t</code>, <code class="language-plaintext highlighter-rouge">show_t</code>, etc.</p>

<p>You can also customize behavior per field with attributes:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">file</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span> <span class="p">[</span><span class="o">@</span><span class="n">equal</span> <span class="k">fun</span> <span class="n">a</span> <span class="n">b</span> <span class="o">-&gt;</span>
    <span class="nn">String</span><span class="p">.</span><span class="n">lowercase_ascii</span> <span class="n">a</span> <span class="o">=</span> <span class="nn">String</span><span class="p">.</span><span class="n">lowercase_ascii</span> <span class="n">b</span><span class="p">];</span>
  <span class="n">perm</span> <span class="o">:</span> <span class="kt">int</span>    <span class="p">[</span><span class="o">@</span><span class="n">compare</span> <span class="k">fun</span> <span class="n">a</span> <span class="n">b</span> <span class="o">-&gt;</span> <span class="n">compare</span> <span class="n">b</span> <span class="n">a</span><span class="p">]</span>  <span class="c">(* reverse order *)</span>
<span class="p">}</span> <span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">eq</span><span class="o">,</span> <span class="n">ord</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>And you can derive for anonymous types inline:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
</pre></td><td class="rouge-code"><pre><span class="c">(* Sort a list of pairs by their second element *)</span>
<span class="k">let</span> <span class="n">sorted</span> <span class="o">=</span> <span class="nn">List</span><span class="p">.</span><span class="n">sort</span> <span class="p">[</span><span class="o">%</span><span class="n">derive</span><span class="o">.</span><span class="n">ord</span><span class="o">:</span> <span class="kt">int</span> <span class="o">*</span> <span class="kt">int</span><span class="p">]</span> <span class="n">pairs</span>

<span class="c">(* Print a list of pairs *)</span>
<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span> <span class="n">print_endline</span> <span class="p">([</span><span class="o">%</span><span class="n">derive</span><span class="o">.</span><span class="n">show</span><span class="o">:</span> <span class="p">(</span><span class="kt">string</span> <span class="o">*</span> <span class="kt">int</span><span class="p">)</span> <span class="kt">list</span><span class="p">]</span> <span class="n">data</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<h3>ppx_deriving_yojson – JSON Serialization</h3>

<p><a href="https://github.com/ocaml-ppx/ppx_deriving_yojson">ppx_deriving_yojson</a>
generates JSON serialization and deserialization functions using the
<a href="https://github.com/ocaml-community/yojson">Yojson</a> library:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">person</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">age</span>  <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
  <span class="n">email</span> <span class="o">:</span> <span class="kt">string</span> <span class="n">option</span><span class="p">;</span>
<span class="p">}</span> <span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">yojson</span><span class="p">]</span>

<span class="c">(* Generates:
   val person_to_yojson : person -&gt; Yojson.Safe.t
   val person_of_yojson : Yojson.Safe.t -&gt; (person, string) result *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">alice</span> <span class="o">=</span> <span class="p">{</span> <span class="n">name</span> <span class="o">=</span> <span class="s2">"Alice"</span><span class="p">;</span> <span class="n">age</span> <span class="o">=</span> <span class="mi">30</span><span class="p">;</span> <span class="n">email</span> <span class="o">=</span> <span class="nc">Some</span> <span class="s2">"alice@example.com"</span> <span class="p">}</span>

<span class="k">let</span> <span class="n">json</span> <span class="o">=</span> <span class="n">person_to_yojson</span> <span class="n">alice</span> <span class="o">|&gt;</span> <span class="nn">Yojson</span><span class="p">.</span><span class="nn">Safe</span><span class="p">.</span><span class="n">to_string</span>
<span class="c">(* {"name":"Alice","age":30,"email":"alice@example.com"} *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>You can use <code class="language-plaintext highlighter-rouge">[@@deriving to_yojson]</code> or <code class="language-plaintext highlighter-rouge">[@@deriving of_yojson]</code> if you only
need one direction.</p>

<p>This is incredibly useful in practice – writing JSON
serializers by hand for complex types is tedious and error-prone.</p>

<h3>ppx_sexp_conv – S-expression Serialization</h3>

<p>If you’re using Jane Street’s <a href="https://github.com/janestreet/core">Core</a>
library, you’ll encounter S-expression serialization everywhere.
(<strong>Tip:</strong> Jane Street bundles most of their PPXs into a single
<a href="https://github.com/janestreet/ppx_jane">ppx_jane</a> package, so you can add
just <code class="language-plaintext highlighter-rouge">ppx_jane</code> to your <code class="language-plaintext highlighter-rouge">pps</code> instead of listing each one individually.)
<a href="https://github.com/janestreet/ppx_sexp_conv">ppx_sexp_conv</a> generates
converters between OCaml types and S-expressions:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">config</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">host</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">port</span> <span class="o">:</span> <span class="kt">int</span>    <span class="p">[</span><span class="o">@</span><span class="n">default</span> <span class="mi">8080</span><span class="p">];</span>
  <span class="n">debug</span> <span class="o">:</span> <span class="kt">bool</span>  <span class="p">[</span><span class="o">@</span><span class="n">sexp</span><span class="o">.</span><span class="kt">bool</span><span class="p">];</span>
<span class="p">}</span> <span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">sexp</span><span class="p">]</span>

<span class="c">(* Generates:
   val sexp_of_config : config -&gt; Sexp.t
   val config_of_sexp : Sexp.t -&gt; config *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The attributes here are quite handy – <code class="language-plaintext highlighter-rouge">[@default 8080]</code> provides a default
value during deserialization, and <code class="language-plaintext highlighter-rouge">[@sexp.bool]</code> means the field is represented
as a present/absent atom rather than <code class="language-plaintext highlighter-rouge">(debug true)</code>.</p>

<h3>ppx_fields_conv and ppx_variants_conv – Accessor Generation</h3>

<p>Two more Jane Street PPXs that you’ll see a lot in Core-based codebases.
<a href="https://github.com/janestreet/ppx_fields_conv">ppx_fields_conv</a> generates
first-class accessors and iterators for record fields:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
</pre></td><td class="rouge-code"><pre><span class="k">type</span> <span class="n">person</span> <span class="o">=</span> <span class="p">{</span>
  <span class="n">name</span> <span class="o">:</span> <span class="kt">string</span><span class="p">;</span>
  <span class="n">age</span>  <span class="o">:</span> <span class="kt">int</span><span class="p">;</span>
<span class="p">}</span> <span class="p">[</span><span class="o">@@</span><span class="n">deriving</span> <span class="n">fields</span><span class="p">]</span>

<span class="c">(* Generates field accessors, a fold over all fields, iter, map, etc. *)</span>
<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span> <span class="nn">Fields</span><span class="p">.</span><span class="n">iter</span>
  <span class="o">~</span><span class="n">name</span><span class="o">:</span><span class="p">(</span><span class="k">fun</span> <span class="n">name</span> <span class="o">-&gt;</span> <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"name: %s</span><span class="se">\n</span><span class="s2">"</span> <span class="n">name</span><span class="p">)</span>
  <span class="o">~</span><span class="n">age</span><span class="o">:</span><span class="p">(</span><span class="k">fun</span> <span class="n">age</span> <span class="o">-&gt;</span> <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"age: %d</span><span class="se">\n</span><span class="s2">"</span> <span class="n">age</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p><a href="https://github.com/janestreet/ppx_variants_conv">ppx_variants_conv</a> does
something similar for variant types – generating constructors as functions,
fold/iter over all variants, and more.</p>

<h3>ppx_inline_test and ppx_expect – Testing</h3>

<p>These Jane Street PPXs let you write tests directly in your source files:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
</pre></td><td class="rouge-code"><pre><span class="c">(* Simple boolean test *)</span>
<span class="k">let</span><span class="o">%</span><span class="n">test</span> <span class="s2">"addition works"</span> <span class="o">=</span> <span class="mi">1</span> <span class="o">+</span> <span class="mi">1</span> <span class="o">=</span> <span class="mi">2</span>

<span class="c">(* Test with side effects *)</span>
<span class="k">let</span><span class="o">%</span><span class="n">test_unit</span> <span class="s2">"list operations"</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">l</span> <span class="o">=</span> <span class="p">[</span><span class="mi">1</span><span class="p">;</span> <span class="mi">2</span><span class="p">;</span> <span class="mi">3</span><span class="p">]</span> <span class="k">in</span>
  <span class="k">assert</span> <span class="p">(</span><span class="nn">List</span><span class="p">.</span><span class="n">length</span> <span class="n">l</span> <span class="o">=</span> <span class="mi">3</span><span class="p">);</span>
  <span class="k">assert</span> <span class="p">(</span><span class="nn">List</span><span class="p">.</span><span class="n">hd</span> <span class="n">l</span> <span class="o">=</span> <span class="mi">1</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p><a href="https://github.com/janestreet/ppx_expect">ppx_expect</a> is particularly nice –
it captures printed output and compares it against expected output:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="k">let</span><span class="o">%</span><span class="n">expect_test</span> <span class="s2">"printing"</span> <span class="o">=</span>
  <span class="nn">List</span><span class="p">.</span><span class="n">iter</span> <span class="p">(</span><span class="k">fun</span> <span class="n">x</span> <span class="o">-&gt;</span> <span class="nn">Printf</span><span class="p">.</span><span class="n">printf</span> <span class="s2">"%d "</span> <span class="n">x</span><span class="p">)</span> <span class="p">[</span><span class="mi">1</span><span class="p">;</span> <span class="mi">2</span><span class="p">;</span> <span class="mi">3</span><span class="p">];</span>
  <span class="p">[</span><span class="o">%</span><span class="n">expect</span> <span class="p">{</span><span class="o">|</span> <span class="mi">1</span> <span class="mi">2</span> <span class="mi">3</span> <span class="o">|</span><span class="p">}]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>If the output doesn’t match, the test fails and you can run <code class="language-plaintext highlighter-rouge">dune promote</code> to
automatically update the expected output in your source file. It’s a very
productive workflow for testing functions that produce output.</p>

<h3>ppx_let – Monadic Syntax</h3>

<p><a href="https://github.com/janestreet/ppx_let">ppx_let</a> provides syntactic sugar for
working with monads and other “container” types:</p>

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
</pre></td><td class="rouge-code"><pre><span class="c">(* Without ppx_let -- callback nesting hell *)</span>
<span class="k">let</span> <span class="n">result</span> <span class="o">=</span>
  <span class="n">bind</span> <span class="p">(</span><span class="n">fetch_user</span> <span class="n">id</span><span class="p">)</span> <span class="p">(</span><span class="k">fun</span> <span class="n">user</span> <span class="o">-&gt;</span>
    <span class="n">bind</span> <span class="p">(</span><span class="n">fetch_posts</span> <span class="n">user</span><span class="p">)</span> <span class="p">(</span><span class="k">fun</span> <span class="n">posts</span> <span class="o">-&gt;</span>
      <span class="n">return</span> <span class="p">(</span><span class="n">user</span><span class="o">,</span> <span class="n">posts</span><span class="p">)))</span>

<span class="c">(* With ppx_let -- reads like sequential code *)</span>
<span class="k">let</span> <span class="n">result</span> <span class="o">=</span>
  <span class="k">let</span><span class="o">%</span><span class="n">bind</span> <span class="n">user</span> <span class="o">=</span> <span class="n">fetch_user</span> <span class="n">id</span> <span class="k">in</span>
  <span class="k">let</span><span class="o">%</span><span class="n">bind</span> <span class="n">posts</span> <span class="o">=</span> <span class="n">fetch_posts</span> <span class="n">user</span> <span class="k">in</span>
  <span class="n">return</span> <span class="p">(</span><span class="n">user</span><span class="o">,</span> <span class="n">posts</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>How does <code class="language-plaintext highlighter-rouge">ppx_let</code> know which <code class="language-plaintext highlighter-rouge">bind</code> to call? It looks for a <code class="language-plaintext highlighter-rouge">Let_syntax</code>
module in scope that provides the underlying <code class="language-plaintext highlighter-rouge">bind</code> and <code class="language-plaintext highlighter-rouge">map</code> functions. In
practice, you’ll typically open a module that defines <code class="language-plaintext highlighter-rouge">Let_syntax</code> before
using <code class="language-plaintext highlighter-rouge">let%bind</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
</pre></td><td class="rouge-code"><pre><span class="k">open</span> <span class="nn">Lwt</span><span class="p">.</span><span class="nc">Syntax</span>  <span class="c">(* or Deferred.Let_syntax, etc. *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p><strong>Note:</strong> Since OCaml 4.08, the language has built-in
<a href="https://v2.ocaml.org/manual/bindingops.html">binding operators</a>
(<code class="language-plaintext highlighter-rouge">let*</code>, <code class="language-plaintext highlighter-rouge">and*</code>, <code class="language-plaintext highlighter-rouge">let+</code>, <code class="language-plaintext highlighter-rouge">and+</code>) that cover the basic use cases of <code class="language-plaintext highlighter-rouge">ppx_let</code>
without needing a preprocessor. If you’re not using Jane Street’s ecosystem,
binding operators are probably the simpler choice. <code class="language-plaintext highlighter-rouge">ppx_let</code> still offers extra
features like <code class="language-plaintext highlighter-rouge">match%bind</code>, <code class="language-plaintext highlighter-rouge">if%bind</code>, and optimized <code class="language-plaintext highlighter-rouge">let%mapn</code> though.</p>

<h3>ppx_blob – Embedding Files</h3>

<p><a href="https://github.com/johnwhitington/ppx_blob">ppx_blob</a> is beautifully simple –
it embeds a file’s contents as a string at compile time:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">icon_data</span> <span class="o">=</span> <span class="p">[</span><span class="o">%</span><span class="n">blob</span> <span class="s2">"assets/icon.png"</span><span class="p">]</span>
<span class="k">let</span> <span class="n">shader_source</span> <span class="o">=</span> <span class="p">[</span><span class="o">%</span><span class="n">blob</span> <span class="s2">"shaders/vertex.glsl"</span><span class="p">]</span>
<span class="k">let</span> <span class="n">default_config</span> <span class="o">=</span> <span class="p">[</span><span class="o">%</span><span class="n">blob</span> <span class="s2">"config/defaults.json"</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>No more worrying about file paths at runtime or packaging data files with your
binary. The file contents become part of your compiled program.</p>

<h3>ppx_string – String Interpolation</h3>

<p>One thing that’s always bugged me about OCaml is the lack of string
interpolation. <a href="https://github.com/janestreet/ppx_string">ppx_string</a> fills that
gap:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">greeting</span> <span class="n">name</span> <span class="n">age</span> <span class="o">=</span>
  <span class="p">[</span><span class="o">%</span><span class="kt">string</span> <span class="s2">"Hello, %{name}! You are %{age#Int} years old."</span><span class="p">]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">#Int</code> suffix tells the PPX to convert the value using <code class="language-plaintext highlighter-rouge">Int.to_string</code>. You
can use any module that provides a <code class="language-plaintext highlighter-rouge">to_string</code> function.</p>

<h2>Writing Your Own PPX (A Gentle Introduction)</h2>

<p>Most OCaml developers will never <em>need</em> to write a PPX, but understanding the
basics helps demystify the whole system. Let’s build a very simple one.</p>

<p>Say we want an extension <code class="language-plaintext highlighter-rouge">[%upcase ...]</code> that converts a string literal to
uppercase at compile time. Here’s the complete implementation using
<a href="https://github.com/ocaml-ppx/ppxlib">ppxlib</a>:</p>

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
</pre></td><td class="rouge-code"><pre><span class="c">(* ppx_upcase.ml *)</span>
<span class="k">open</span> <span class="nc">Ppxlib</span>

<span class="k">let</span> <span class="n">extension</span> <span class="o">=</span>
  <span class="nn">Extension</span><span class="p">.</span><span class="nn">V3</span><span class="p">.</span><span class="n">declare</span>
    <span class="s2">"upcase"</span>
    <span class="nn">Extension</span><span class="p">.</span><span class="nn">Context</span><span class="p">.</span><span class="n">expression</span>
    <span class="nn">Ast_pattern</span><span class="p">.(</span><span class="n">single_expr_payload</span> <span class="p">(</span><span class="n">estring</span> <span class="n">__</span><span class="p">))</span>
    <span class="p">(</span><span class="k">fun</span> <span class="o">~</span><span class="n">ctxt</span> <span class="n">expr</span> <span class="o">-&gt;</span>
      <span class="k">let</span> <span class="n">loc</span> <span class="o">=</span> <span class="nn">Expansion_context</span><span class="p">.</span><span class="nn">Extension</span><span class="p">.</span><span class="n">extension_point_loc</span> <span class="n">ctxt</span> <span class="k">in</span>
      <span class="k">let</span> <span class="n">uppercased</span> <span class="o">=</span> <span class="nn">String</span><span class="p">.</span><span class="n">uppercase_ascii</span> <span class="n">expr</span> <span class="k">in</span>
      <span class="nn">Ast_builder</span><span class="p">.</span><span class="nn">Default</span><span class="p">.</span><span class="n">estring</span> <span class="o">~</span><span class="n">loc</span> <span class="n">uppercased</span><span class="p">)</span>

<span class="k">let</span> <span class="n">rule</span> <span class="o">=</span> <span class="nn">Context_free</span><span class="p">.</span><span class="nn">Rule</span><span class="p">.</span><span class="n">extension</span> <span class="n">extension</span>
<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span> <span class="nn">Driver</span><span class="p">.</span><span class="n">register_transformation</span> <span class="o">~</span><span class="n">rules</span><span class="o">:</span><span class="p">[</span><span class="n">rule</span><span class="p">]</span> <span class="s2">"ppx_upcase"</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The dune file:</p>

<pre><code class="language-dune">(library
 (name ppx_upcase)
 (kind ppx_rewriter)
 (libraries ppxlib))
</code></pre>

<p>And usage:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="k">let</span> <span class="n">greeting</span> <span class="o">=</span> <span class="p">[</span><span class="o">%</span><span class="n">upcase</span> <span class="s2">"hello world"</span><span class="p">]</span>
<span class="c">(* After preprocessing: let greeting = "HELLO WORLD" *)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The key pieces are:</p>

<ol>
  <li><strong><code class="language-plaintext highlighter-rouge">Extension.V3.declare</code></strong> – registers an extension with a name, the context where it
can appear (expressions, patterns, types, etc.), the expected payload pattern,
and an expansion function.</li>
  <li><strong><code class="language-plaintext highlighter-rouge">Ast_pattern</code></strong> – a pattern-matching DSL for destructuring AST nodes. Here
<code class="language-plaintext highlighter-rouge">estring __</code> matches a string literal and captures its value.</li>
  <li><strong><code class="language-plaintext highlighter-rouge">Ast_builder.Default</code></strong> – helpers for constructing AST nodes. <code class="language-plaintext highlighter-rouge">estring</code>
builds a string literal expression.</li>
  <li><strong><code class="language-plaintext highlighter-rouge">Driver.register_transformation</code></strong> – registers the rule with ppxlib’s
driver.</li>
</ol>

<p>For more complex PPXs (especially derivers), you’ll also want to use
<strong>Metaquot</strong> (<code class="language-plaintext highlighter-rouge">ppxlib.metaquot</code>), which lets you write AST-constructing code
using actual OCaml syntax instead of manual AST builder calls:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
</pre></td><td class="rouge-code"><pre><span class="c">(* Without metaquot -- verbose *)</span>
<span class="nn">Ast_builder</span><span class="p">.</span><span class="nn">Default</span><span class="p">.</span><span class="n">pexp_apply</span> <span class="o">~</span><span class="n">loc</span>
  <span class="p">(</span><span class="nn">Ast_builder</span><span class="p">.</span><span class="nn">Default</span><span class="p">.</span><span class="n">evar</span> <span class="o">~</span><span class="n">loc</span> <span class="s2">"string_of_int"</span><span class="p">)</span>
  <span class="p">[(</span><span class="nc">Nolabel</span><span class="o">,</span> <span class="n">expr</span><span class="p">)]</span>

<span class="c">(* With metaquot -- just write OCaml *)</span>
<span class="p">[</span><span class="o">%</span><span class="n">expr</span> <span class="n">string_of_int</span> <span class="p">[</span><span class="o">%</span><span class="n">e</span> <span class="n">expr</span><span class="p">]]</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The ppxlib documentation has excellent
<a href="https://ocaml-ppx.github.io/ppxlib/ppxlib/quick_intro.html">tutorials</a> if you
want to go deeper.</p>

<h2>Debugging PPX-generated Code</h2>

<p>One practical tip: when something goes wrong with PPX-generated code and you’re
staring at a confusing type error, you can inspect what the PPX actually
generated:</p>

<div class="language-shell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
9
</pre></td><td class="rouge-code"><pre><span class="c"># Show the preprocessed output for a file (dune 3.x+)</span>
dune describe pp src/myfile.ml

<span class="c"># Alternatively, build the .pp.ml target and inspect it directly</span>
dune build src/.mylib.pp/myfile.pp.ml
<span class="nb">cat </span>_build/default/src/.mylib.pp/myfile.pp.ml

<span class="c"># Or use ocaml-print-intf to see the generated interface</span>
dune <span class="nb">exec</span> <span class="nt">--</span> ocaml-print-intf src/myfile.ml
</pre></td></tr></tbody></table></code></pre></div></div>

<p>Seeing the expanded code often makes the error immediately obvious.</p>

<h2>The PPX Ecosystem Today</h2>

<p>Most of the introductory PPX content out there was written around 2018-2019,
so it’s worth noting how things have evolved since then.</p>

<p>The big story has been <strong>ppxlib’s consolidation of the ecosystem</strong>. Back in
2019, some PPX rewriters still used the older <code class="language-plaintext highlighter-rouge">ocaml-migrate-parsetree</code> (OMP)
library, creating fragmentation. By 2021, <a href="https://discuss.ocaml.org/t/an-update-on-the-state-of-the-ppx-ecosystem-and-ppxlib-s-transition/8200">nearly all PPXs had migrated to
ppxlib</a>,
effectively ending the split. Today ppxlib is <em>the</em> way to write PPX rewriters
– there’s no real alternative to consider.</p>

<p>The transition hasn’t always been smooth, though. In 2025, ppxlib 0.36.0
bumped its internal AST to match OCaml 5.2, which changed how functions are
represented in the parse tree. This <a href="https://discuss.ocaml.org/t/breaking-ppx-api-changes-and-the-package-ecosystem/16837">broke many downstream
PPXs</a>
and temporarily split the opam universe between packages that worked with the
new version and those that didn’t. The community worked through it with
proactive patching, but it highlighted an ongoing tension in the PPX world:
ppxlib shields you from most compiler changes, but major AST overhauls still
ripple through the ecosystem.</p>

<p>On the API side, ppxlib is gradually <a href="https://ocaml.org/changelog/2025-10-01-ppxlib-0362">deprecating its copy of
<code class="language-plaintext highlighter-rouge">Ast_helper</code></a> in favor of
<code class="language-plaintext highlighter-rouge">Ast_builder</code>, with plans to remove <code class="language-plaintext highlighter-rouge">Ast_helper</code> entirely in a future 1.0.0
release. If you’re writing a new PPX today, use <code class="language-plaintext highlighter-rouge">Ast_builder</code> exclusively.</p>

<p>Meanwhile, OCaml 4.08’s built-in binding operators (<code class="language-plaintext highlighter-rouge">let*</code>, <code class="language-plaintext highlighter-rouge">let+</code>, etc.) have
reduced the need for <code class="language-plaintext highlighter-rouge">ppx_let</code> in projects that don’t use Jane Street’s
ecosystem. It’s a nice example of the language absorbing a pattern that PPX
pioneered. Perhaps one day we’ll see more of this (e.g. native string interpolation).</p>

<h2>Further Reading</h2>

<p>This article covers a lot of ground, but the PPX topic is pretty deep and complex,
so depending on how far you want to go you might want to read more on it.
Here are some of the best resources I’ve found on PPX:</p>

<ul>
  <li><a href="https://ocaml.org/docs/metaprogramming">Preprocessors and PPXs</a> – the
official OCaml documentation on metaprogramming. A solid reference, though
it assumes some comfort with the compiler internals.</li>
  <li><a href="https://tarides.com/blog/2019-05-09-an-introduction-to-ocaml-ppx-ecosystem/">An Introduction to OCaml PPX
Ecosystem</a>
– Nathan Rebours’ 2019 deep dive for Tarides. This is the most thorough
tutorial on <em>writing</em> PPX rewriters I’ve seen. Some API details have changed
since 2019 (notably the <code class="language-plaintext highlighter-rouge">Ast_helper</code> → <code class="language-plaintext highlighter-rouge">Ast_builder</code> shift), but the
concepts and approach are still excellent.</li>
  <li><a href="https://ocaml-ppx.github.io/ppxlib/ppxlib/quick_intro.html">ppxlib Quick
Introduction</a> –
ppxlib’s own getting-started guide. The best place to begin if you want to
write your own PPX.</li>
  <li><a href="http://ocamlverse.net/content/ppx.html">A Guide to PreProcessor
eXtensions</a> – OCamlverse’s
reference page with a comprehensive list of available PPX libraries.</li>
  <li><a href="https://whitequark.org/blog/2014/04/16/a-guide-to-extension-points-in-ocaml/">A Guide to Extension Points in
OCaml</a>
– Whitequark’s original 2014 guide that introduced many developers to PPX.
Historically interesting as a snapshot of the early PPX days.</li>
</ul>

<p>I was amused to see whitequark’s name pop up while I was doing research for
this article – we collaborated quite a bit back in the day on her Ruby
<a href="https://github.com/whitequark/parser">parser</a> project, which was instrumental
to <a href="https://github.com/rubocop/rubocop">RuboCop</a>. Seems you can find (former)
Rubyists in pretty much every language community.</p>

<h2>Epilogue</h2>

<p>This article turned out to be a beast! I’ve wanted to write something on the subject
for quite a while now, but I’ve kept postponing it because I was too lazy to do
all the necessary research. I’ll feel quite relieved to put it behind me!</p>

<p>PPX might look intimidating at first – all those brackets and <code class="language-plaintext highlighter-rouge">@</code> symbols can
feel like line noise. But the core idea is simple: PPX generates boilerplate
code from your type definitions at compile time. You annotate your types with
what you want (<code class="language-plaintext highlighter-rouge">show</code>, <code class="language-plaintext highlighter-rouge">eq</code>, <code class="language-plaintext highlighter-rouge">yojson</code>, <code class="language-plaintext highlighter-rouge">sexp</code>, etc.), and the PPX rewriter
produces the code you’d otherwise have to write by hand.</p>

<p>For day-to-day OCaml programming, you really only need to know:</p>

<ol>
  <li><code class="language-plaintext highlighter-rouge">[@@deriving ...]</code> on type declarations to generate useful functions</li>
  <li>How to add PPX libraries to your dune file with <code class="language-plaintext highlighter-rouge">(pps ...)</code></li>
  <li>Which PPX libraries exist for common tasks (serialization, testing,
pretty-printing)</li>
</ol>

<p>The “writing your own PPX” part is there for when you need it, but honestly most
OCaml developers get by just fine using the existing ecosystem.</p>

<p>That’s all I have for you today. Keep hacking!</p>
