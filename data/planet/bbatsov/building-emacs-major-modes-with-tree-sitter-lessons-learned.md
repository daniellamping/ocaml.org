---
title: 'Building Emacs Major Modes with Tree-sitter: Lessons Learned'
description: "Over the past year I\u2019ve been spending a lot of time building Tree-sitter-powered
  major modes for Emacs \u2013 clojure-ts-mode (as co-maintainer), neocaml (from scratch),
  and asciidoc-mode (also from scratch). Between the three projects I\u2019ve accumulated
  enough knowledge (and battle scars) to write about the experience. This post distills
  the key lessons for anyone thinking about writing a Tree-sitter-based major mode,
  or curious about what it\u2019s actually like."
url: https://batsov.com/articles/2026/02/27/building-emacs-major-modes-with-treesitter-lessons-learned/
date: 2026-02-27T08:00:00-00:00
preview_image: https://batsov.com/assets/img/og-image.png
authors:
- Bozhidar Batsov
source:
ignore:
---

<p>Over the past year I’ve been spending a lot of time building <a href="https://tree-sitter.github.io/tree-sitter/">Tree-sitter</a>-powered
major modes for Emacs – <a href="https://github.com/clojure-emacs/clojure-ts-mode">clojure-ts-mode</a>
(as co-maintainer), <a href="https://github.com/bbatsov/neocaml">neocaml</a> (from scratch),
and <a href="https://github.com/bbatsov/asciidoc-mode">asciidoc-mode</a> (also from scratch).
Between the three projects I’ve accumulated enough knowledge (and battle scars) to write about the
experience. This post distills the key lessons for anyone thinking about writing
a Tree-sitter-based major mode, or curious about what it’s actually like.</p>



<h2>Why Tree-sitter?</h2>

<p>Before Tree-sitter, Emacs font-locking was done with regular expressions and
indentation was handled by ad-hoc engines (SMIE, custom indent functions, or
pure regex heuristics). This works, but it has well-known problems:</p>

<ul>
  <li>
    <p><strong>Regex-based font-locking is fragile.</strong> Regexes can’t parse nested structures,
so they either under-match (missing valid code) or over-match (highlighting
inside strings and comments). Every edge case is another regex, and the patterns
become increasingly unreadable over time.</p>
  </li>
  <li>
    <p><strong>Indentation engines are complex.</strong> SMIE (the generic indentation engine for non-Tree-sitter modes) requires defining operator
precedence grammars for the language, which is hard to get right. Custom
indentation functions tend to grow into large, brittle state machines. Tuareg’s
indentation code, for example, is thousands of lines long.</p>
  </li>
</ul>

<p>Tree-sitter changes the game because you get a <strong>full, incremental, error-tolerant
syntax tree</strong> for free. Font-locking becomes “match this AST pattern, apply this
face”:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="c1">;; Highlight let-bound functions: match a let_binding with parameters</span>
<span class="p">(</span><span class="nv">let_binding</span> <span class="nv">pattern:</span> <span class="p">(</span><span class="nv">value_name</span><span class="p">)</span> <span class="nv">@font-lock-function-name-face</span>
             <span class="p">(</span><span class="nv">parameter</span><span class="p">)</span><span class="nb">+</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>And indentation becomes “if the parent node is X, indent by Y”:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="c1">;; Children of a let_binding are indented by neocaml-indent-offset</span>
<span class="p">((</span><span class="nv">parent-is</span> <span class="s">"let_binding"</span><span class="p">)</span> <span class="nv">parent-bol</span> <span class="nv">neocaml-indent-offset</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The rules are declarative, composable, and much easier to reason about than
regex chains.</p>

<p>In practice, <code class="language-plaintext highlighter-rouge">neocaml</code>’s entire font-lock and indentation logic fits in about 350
lines of Elisp. The equivalent in tuareg is spread across thousands of lines.
That’s the real selling point: <strong>simpler, more maintainable code that handles more
edge cases correctly</strong>.</p>

<h2>Challenges</h2>

<p>That said, Tree-sitter in Emacs is not a silver bullet. Here’s what I ran into.</p>

<h3>Every grammar is different</h3>

<p>Tree-sitter grammars are written by different authors with different philosophies.
The <a href="https://github.com/tree-sitter/tree-sitter-ocaml">tree-sitter-ocaml</a>
grammar provides a rich, detailed AST with named fields. The
<a href="https://github.com/sogaiu/tree-sitter-clojure">tree-sitter-clojure</a> grammar,
by contrast, deliberately keeps things minimal – it only models syntax, not
semantics, because Clojure’s macro system makes static semantic analysis
unreliable.<sup><a href="https://batsov.com/feeds/OCaml.xml#fn:1" class="footnote" rel="footnote" role="doc-noteref">1</a></sup> This means font-locking <code class="language-plaintext highlighter-rouge">def</code> forms in Clojure requires
predicate matching on symbol text, while in OCaml you can directly match
<code class="language-plaintext highlighter-rouge">let_binding</code> nodes with named fields.</p>

<p>To illustrate: here’s how you’d fontify a function definition in OCaml, where
the grammar gives you rich named fields:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="c1">;; OCaml: grammar provides named fields -- direct structural match</span>
<span class="p">(</span><span class="nv">let_binding</span> <span class="nv">pattern:</span> <span class="p">(</span><span class="nv">value_name</span><span class="p">)</span> <span class="nv">@font-lock-function-name-face</span>
             <span class="p">(</span><span class="nv">parameter</span><span class="p">)</span><span class="nb">+</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>And here’s the equivalent in Clojure, where the grammar only gives you lists of
symbols and you need predicate matching:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
</pre></td><td class="rouge-code"><pre><span class="c1">;; Clojure: grammar is syntax-only -- match by symbol text</span>
<span class="p">((</span><span class="nv">list_lit</span> <span class="ss">:anchor</span> <span class="p">(</span><span class="nv">sym_lit</span> <span class="nv">!namespace</span>
                            <span class="nv">name:</span> <span class="p">(</span><span class="nv">sym_name</span><span class="p">)</span> <span class="nv">@font-lock-keyword-face</span><span class="p">))</span>
 <span class="p">(</span><span class="ss">:match</span> <span class="o">,</span><span class="nv">clojure-ts--definition-keyword-regexp</span> <span class="nv">@font-lock-keyword-face</span><span class="p">))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>You can’t learn “how to write Tree-sitter queries” generically – you need to
learn each grammar individually. The best tool for this is <code class="language-plaintext highlighter-rouge">treesit-explore-mode</code>
(to visualize the full parse tree) and <code class="language-plaintext highlighter-rouge">treesit-inspect-mode</code> (to see the node
at point). Use them constantly.</p>

<h3>Grammar quality varies wildly</h3>

<p>You’re dependent on someone else providing the grammar, and quality is all over
the map. The OCaml grammar is mature and well-maintained – it’s hosted under the
official <a href="https://github.com/tree-sitter">tree-sitter</a> GitHub org. The Clojure
grammar is small and stable by design. But not every language is so lucky.</p>

<p><a href="https://github.com/bbatsov/asciidoc-mode">asciidoc-mode</a> uses a
<a href="https://github.com/cathaysia/tree-sitter-asciidoc">third-party AsciiDoc grammar</a>
that employs a dual-parser architecture – one parser for block-level structure
(headings, lists, code blocks) and another for inline formatting (bold, italic,
links). This is the same approach used by Emacs’s built-in <code class="language-plaintext highlighter-rouge">markdown-ts-mode</code>,
and it makes sense for markup languages where block and inline syntax are largely
independent.</p>

<p>The problem is that the two parsers run independently on the same text, and they
can <strong>disagree</strong>. The inline parser misinterprets <code class="language-plaintext highlighter-rouge">*</code> and <code class="language-plaintext highlighter-rouge">**</code> list markers as
emphasis delimiters, creating spurious bold spans that swallow subsequent inline
content. The workaround is to use <code class="language-plaintext highlighter-rouge">:override t</code> on all block-level font-lock
rules so they win over the incorrect inline faces:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
8
</pre></td><td class="rouge-code"><pre><span class="c1">;; Block-level rules use :override t so block-level faces win over</span>
<span class="c1">;; spurious inline emphasis nodes (the inline parser misreads `*'</span>
<span class="c1">;; list markers as emphasis delimiters).</span>
<span class="ss">:language</span> <span class="ss">'asciidoc</span>
<span class="ss">:override</span> <span class="no">t</span>
<span class="ss">:feature</span> <span class="ss">'list</span>
<span class="o">'</span><span class="p">((</span><span class="nv">ordered_list_marker</span><span class="p">)</span> <span class="nv">@font-lock-constant-face</span>
  <span class="p">(</span><span class="nv">unordered_list_marker</span><span class="p">)</span> <span class="nv">@font-lock-constant-face</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>This doesn’t fix inline elements consumed by the spurious emphasis – that
requires an upstream grammar fix. When you hit grammar-level issues like this,
you either fix them yourself (which means diving into the grammar’s JavaScript
source and C toolchain) or you live with workarounds. Either way, it’s a
reminder that your mode is only as good as the grammar underneath it.</p>

<p>Getting the font-locking right in <code class="language-plaintext highlighter-rouge">asciidoc-mode</code> was probably the most
challenging part of all three projects, precisely because of these grammar
quirks. I also ran into a subtle <code class="language-plaintext highlighter-rouge">treesit</code> behavior: the default font-lock mode
(<code class="language-plaintext highlighter-rouge">:override nil</code>) skips an entire captured range if <em>any</em> position within it
already has a face. So if you capture a parent node like <code class="language-plaintext highlighter-rouge">(inline_macro)</code> and a
child was already fontified, the whole thing gets skipped silently. The fix is
to capture specific child nodes instead:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
</pre></td><td class="rouge-code"><pre><span class="c1">;; BAD: entire node gets skipped if any child is already fontified</span>
<span class="c1">;; (inline_macro) @font-lock-function-call-face</span>

<span class="c1">;; GOOD: capture specific children</span>
<span class="p">(</span><span class="nv">inline_macro</span> <span class="p">(</span><span class="nv">macro_name</span><span class="p">)</span> <span class="nv">@font-lock-function-call-face</span><span class="p">)</span>
<span class="p">(</span><span class="nv">inline_macro</span> <span class="p">(</span><span class="nv">target</span><span class="p">)</span> <span class="nv">@font-lock-string-face</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>These issues took a lot of trial and error to diagnose. The lesson: <strong>budget
extra time for font-locking when working with less mature grammars</strong>.</p>

<h3>Grammar versions and breaking changes</h3>

<p>Grammars evolve, and breaking changes happen. <code class="language-plaintext highlighter-rouge">clojure-ts-mode</code> switched from
the stable grammar to the <a href="https://github.com/sogaiu/tree-sitter-clojure/tree/unstable-20250526">experimental
branch</a>
because the stable version had metadata nodes as children of other nodes, which
caused <code class="language-plaintext highlighter-rouge">forward-sexp</code> and <code class="language-plaintext highlighter-rouge">kill-sexp</code> to behave incorrectly. The experimental
grammar makes metadata standalone nodes, fixing the navigation issues but
requiring all queries to be updated.</p>

<p><code class="language-plaintext highlighter-rouge">neocaml</code> pins to
<a href="https://github.com/tree-sitter/tree-sitter-ocaml/tree/v0.24.0">v0.24.0</a> of the
OCaml grammar. If you don’t pin versions, a grammar update can silently break
your font-locking or indentation.</p>

<p>The takeaway: <strong>always pin your grammar version</strong>, and include a mechanism to
detect outdated grammars. <code class="language-plaintext highlighter-rouge">clojure-ts-mode</code> tests a query that changed between
versions to detect incompatible grammars at startup.</p>

<h3>Grammar delivery</h3>

<p>Users shouldn’t have to manually clone repos and compile C code to use your
mode. Both <code class="language-plaintext highlighter-rouge">neocaml</code> and <code class="language-plaintext highlighter-rouge">clojure-ts-mode</code> include grammar recipes:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
6
7
</pre></td><td class="rouge-code"><pre><span class="p">(</span><span class="nv">defconst</span> <span class="nv">neocaml-grammar-recipes</span>
  <span class="o">'</span><span class="p">((</span><span class="nv">ocaml</span> <span class="s">"https://github.com/tree-sitter/tree-sitter-ocaml"</span>
           <span class="s">"v0.24.0"</span>
           <span class="s">"grammars/ocaml/src"</span><span class="p">)</span>
    <span class="p">(</span><span class="nv">ocaml-interface</span> <span class="s">"https://github.com/tree-sitter/tree-sitter-ocaml"</span>
                     <span class="s">"v0.24.0"</span>
                     <span class="s">"grammars/interface/src"</span><span class="p">)))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>On first use, the mode checks <code class="language-plaintext highlighter-rouge">treesit-language-available-p</code> and offers to install
missing grammars via <code class="language-plaintext highlighter-rouge">treesit-install-language-grammar</code>. This works, but requires
a C compiler and Git on the user’s machine, which is not ideal.<sup><a href="https://batsov.com/feeds/OCaml.xml#fn:2" class="footnote" rel="footnote" role="doc-noteref">2</a></sup></p>

<h3>The Emacs Tree-sitter APIs are a moving target</h3>

<p>The Tree-sitter support in Emacs has been improving steadily, but each version
has its quirks:</p>

<p><strong>Emacs 29</strong> introduced Tree-sitter support but lacked several APIs. For instance,
<code class="language-plaintext highlighter-rouge">treesit-thing-settings</code> (used for structured navigation) doesn’t exist – you
need a fallback:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="c1">;; Fallback for Emacs 29 (no treesit-thing-settings)</span>
<span class="p">(</span><span class="nb">unless</span> <span class="p">(</span><span class="nb">boundp</span> <span class="ss">'treesit-thing-settings</span><span class="p">)</span>
  <span class="p">(</span><span class="nv">setq-local</span> <span class="nv">forward-sexp-function</span> <span class="nf">#'</span><span class="nv">neocaml-forward-sexp</span><span class="p">))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p><strong>Emacs 30</strong> added <code class="language-plaintext highlighter-rouge">treesit-thing-settings</code>, sentence navigation, and better
indentation support. But it also had a bug in <code class="language-plaintext highlighter-rouge">treesit-range-settings</code> offsets
(<a href="https://debbugs.gnu.org/cgi/bugreport.cgi?bug=77848">#77848</a>) that broke
embedded parsers, and another in <code class="language-plaintext highlighter-rouge">treesit-transpose-sexps</code> that required
<code class="language-plaintext highlighter-rouge">clojure-ts-mode</code> to disable its Tree-sitter-aware version.</p>

<p><strong>Emacs 31</strong> has a bug in <code class="language-plaintext highlighter-rouge">treesit-forward-comment</code> where an off-by-one error
causes <code class="language-plaintext highlighter-rouge">uncomment-region</code> to leave <code class="language-plaintext highlighter-rouge">*)</code> behind on multi-line OCaml comments. I
had to skip the affected test with a version check:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="p">(</span><span class="nb">when</span> <span class="p">(</span><span class="nb">&gt;=</span> <span class="nv">emacs-major-version</span> <span class="mi">31</span><span class="p">)</span>
  <span class="p">(</span><span class="nb">signal</span> <span class="ss">'buttercup-pending</span>
          <span class="s">"Emacs 31 treesit-forward-comment bug (off-by-one)"</span><span class="p">))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The lesson: <strong>test your mode against multiple Emacs versions</strong>, and be prepared
to write version-specific workarounds. CI that runs against Emacs 29, 30, and
snapshot is essential.</p>

<h3>No .scm file support (yet)</h3>

<p>Most Tree-sitter grammars ship with <code class="language-plaintext highlighter-rouge">.scm</code> query files for syntax highlighting
(<code class="language-plaintext highlighter-rouge">highlights.scm</code>) and indentation (<code class="language-plaintext highlighter-rouge">indents.scm</code>). Editors like Neovim and
Helix use these directly. Emacs doesn’t – you have to manually translate the
<code class="language-plaintext highlighter-rouge">.scm</code> patterns into <code class="language-plaintext highlighter-rouge">treesit-font-lock-rules</code> and <code class="language-plaintext highlighter-rouge">treesit-simple-indent-rules</code>
calls in Elisp.</p>

<p>This is tedious and error-prone. For example, here’s a rule from the OCaml
grammar’s <code class="language-plaintext highlighter-rouge">highlights.scm</code>:</p>

<div class="language-scheme highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
</pre></td><td class="rouge-code"><pre><span class="c1">;; upstream .scm (used by Neovim, Helix, etc.)</span>
<span class="p">(</span><span class="nf">constructor_name</span><span class="p">)</span> <span class="nv">@type</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>And here’s the Elisp equivalent you’d write for Emacs:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
</pre></td><td class="rouge-code"><pre><span class="c1">;; Emacs equivalent -- wrapped in treesit-font-lock-rules</span>
<span class="ss">:language</span> <span class="ss">'ocaml</span>
<span class="ss">:feature</span> <span class="ss">'type</span>
<span class="o">'</span><span class="p">((</span><span class="nv">constructor_name</span><span class="p">)</span> <span class="nv">@font-lock-type-face</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The query syntax is nearly identical, but you have to wrap everything in
<code class="language-plaintext highlighter-rouge">treesit-font-lock-rules</code> calls, map upstream capture names (<code class="language-plaintext highlighter-rouge">@type</code>) to Emacs
face names (<code class="language-plaintext highlighter-rouge">@font-lock-type-face</code>), assign features, and manage <code class="language-plaintext highlighter-rouge">:override</code>
behavior. You end up maintaining a parallel set of queries that can drift from
upstream. Emacs 31 will introduce
<a href="https://github.com/emacs-mirror/emacs/blob/master/lisp/treesit-x.el#L47"><code class="language-plaintext highlighter-rouge">define-treesit-generic-mode</code></a>
which will make it possible to use <code class="language-plaintext highlighter-rouge">.scm</code> files for font-locking, which should
help significantly. But for now, you’re hand-coding everything.</p>

<h2>Tips and tricks</h2>

<h3>Debugging font-locking</h3>

<p>When a face isn’t being applied where you expect:</p>

<ol>
  <li>Use <code class="language-plaintext highlighter-rouge">treesit-inspect-mode</code> to verify the node type at point matches your
query.</li>
  <li>Set <code class="language-plaintext highlighter-rouge">treesit--font-lock-verbose</code> to <code class="language-plaintext highlighter-rouge">t</code> to see which rules are firing.</li>
  <li>Check the font-lock feature level – your rule might be in level 4 while the
user has the default level 3. The features are assigned to levels via
<code class="language-plaintext highlighter-rouge">treesit-font-lock-feature-list</code>.</li>
  <li>Remember that <strong>rule order matters</strong>. Without <code class="language-plaintext highlighter-rouge">:override</code>, an earlier rule that
already fontified a region will prevent later rules from applying. This can be
intentional (e.g. builtin types at level 3 take precedence over generic types)
or a source of bugs.</li>
</ol>

<h3>Use the font-lock levels wisely</h3>

<p>Tree-sitter modes define four levels of font-locking via
<code class="language-plaintext highlighter-rouge">treesit-font-lock-feature-list</code>, and the default level in Emacs is 3. It’s
tempting to pile everything into levels 1–3 so users see maximum highlighting
out of the box, but resist the urge. When every token on the screen has a
different color, code starts looking like a Christmas tree and the important
things – keywords, definitions, types – stop standing out.</p>

<p>Less is more here. Here’s how <code class="language-plaintext highlighter-rouge">neocaml</code> distributes features across levels:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
</pre></td><td class="rouge-code"><pre><span class="p">(</span><span class="nv">setq-local</span> <span class="nv">treesit-font-lock-feature-list</span>
            <span class="o">'</span><span class="p">((</span><span class="nv">comment</span> <span class="nv">definition</span><span class="p">)</span>
              <span class="p">(</span><span class="kt">keyword</span> <span class="nb">string</span> <span class="nc">number</span><span class="p">)</span>
              <span class="p">(</span><span class="nv">attribute</span> <span class="nv">builtin</span> <span class="nv">constant</span> <span class="k">type</span><span class="p">)</span>
              <span class="p">(</span><span class="nv">operator</span> <span class="nv">bracket</span> <span class="nv">delimiter</span> <span class="nv">variable</span> <span class="k">function</span><span class="p">)))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>And <code class="language-plaintext highlighter-rouge">clojure-ts-mode</code> follows the same philosophy:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
</pre></td><td class="rouge-code"><pre><span class="p">(</span><span class="nv">setq-local</span> <span class="nv">treesit-font-lock-feature-list</span>
            <span class="o">'</span><span class="p">((</span><span class="nv">comment</span> <span class="nv">definition</span><span class="p">)</span>
              <span class="p">(</span><span class="kt">keyword</span> <span class="nb">string</span> <span class="nb">char</span> <span class="nc">symbol</span> <span class="nv">builtin</span> <span class="k">type</span><span class="p">)</span>
              <span class="p">(</span><span class="nv">constant</span> <span class="nc">number</span> <span class="k">quote</span> <span class="nv">metadata</span> <span class="nv">doc</span> <span class="nv">regex</span><span class="p">)</span>
              <span class="p">(</span><span class="nv">bracket</span> <span class="nv">deref</span> <span class="k">function</span> <span class="nv">tagged-literals</span><span class="p">)))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>The pattern is the same: essentials first, progressively more detail at higher
levels. This way the default experience (level 3) is clean and readable, and
users who want the full rainbow can bump <code class="language-plaintext highlighter-rouge">treesit-font-lock-level</code> to 4. Better
yet, they can use <code class="language-plaintext highlighter-rouge">treesit-font-lock-recompute-features</code> to cherry-pick
individual features regardless of level:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
5
</pre></td><td class="rouge-code"><pre><span class="c1">;; Enable 'function' (level 4) without enabling all of level 4</span>
<span class="p">(</span><span class="nv">treesit-font-lock-recompute-features</span> <span class="o">'</span><span class="p">(</span><span class="k">function</span><span class="p">)</span> <span class="no">nil</span><span class="p">)</span>

<span class="c1">;; Disable 'bracket' even if the user's level would include it</span>
<span class="p">(</span><span class="nv">treesit-font-lock-recompute-features</span> <span class="no">nil</span> <span class="o">'</span><span class="p">(</span><span class="nv">bracket</span><span class="p">))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>This gives users fine-grained control without requiring mode authors to
anticipate every preference.</p>

<h3>Debugging indentation</h3>

<p>Indentation issues are harder to diagnose because they depend on tree structure,
rule ordering, and anchor resolution:</p>

<ol>
  <li>Set <code class="language-plaintext highlighter-rouge">treesit--indent-verbose</code> to <code class="language-plaintext highlighter-rouge">t</code> – this logs which rule matched for each
line, what anchor was computed, and the final column.</li>
  <li>Use <code class="language-plaintext highlighter-rouge">treesit-explore-mode</code> to understand the parent chain. The key question
is always: “what is the parent node, and which rule matches it?”</li>
  <li>
    <p>Remember that <strong>rule order matters</strong> for indentation too – the first matching
rule wins. A typical set of rules reads top to bottom from most specific to
most general:</p>

    <div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
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
</pre></td><td class="rouge-code"><pre><span class="c1">;; Closing delimiters align with the opening construct</span>
<span class="p">((</span><span class="nv">node-is</span> <span class="s">")"</span><span class="p">)</span> <span class="nv">parent-bol</span> <span class="mi">0</span><span class="p">)</span>
<span class="p">((</span><span class="nv">node-is</span> <span class="s">"end"</span><span class="p">)</span> <span class="nv">parent-bol</span> <span class="mi">0</span><span class="p">)</span>

<span class="c1">;; then/else clauses align with their enclosing if</span>
<span class="p">((</span><span class="nv">node-is</span> <span class="s">"then_clause"</span><span class="p">)</span> <span class="nv">parent-bol</span> <span class="mi">0</span><span class="p">)</span>
<span class="p">((</span><span class="nv">node-is</span> <span class="s">"else_clause"</span><span class="p">)</span> <span class="nv">parent-bol</span> <span class="mi">0</span><span class="p">)</span>

<span class="c1">;; Bodies inside then/else are indented</span>
<span class="p">((</span><span class="nv">parent-is</span> <span class="s">"then_clause"</span><span class="p">)</span> <span class="nv">parent-bol</span> <span class="nv">neocaml-indent-offset</span><span class="p">)</span>
<span class="p">((</span><span class="nv">parent-is</span> <span class="s">"else_clause"</span><span class="p">)</span> <span class="nv">parent-bol</span> <span class="nv">neocaml-indent-offset</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div>    </div>
  </li>
  <li>
    <p>Watch out for the <strong>empty-line problem</strong>: when the cursor is on a blank line,
Tree-sitter has no node at point. The indentation engine falls back to the root
<code class="language-plaintext highlighter-rouge">compilation_unit</code> node as the parent, which typically matches the top-level
rule and gives column 0. In neocaml I solved this with a <code class="language-plaintext highlighter-rouge">no-node</code> rule that
looks at the previous line’s last token to decide indentation:</p>

    <div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
</pre></td><td class="rouge-code"><pre><span class="p">(</span><span class="nv">no-node</span> <span class="nv">prev-line</span> <span class="nv">neocaml--empty-line-offset</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div>    </div>
  </li>
</ol>

<h3>Build a comprehensive test suite</h3>

<p>This is the single most important piece of advice. Font-lock and indentation are
easy to break accidentally, and manual testing doesn’t scale. Both projects use
<a href="https://github.com/jorgenschaefer/emacs-buttercup">Buttercup</a> (a BDD testing
framework for Emacs) with custom test macros.</p>

<p><strong>Font-lock tests</strong> insert code into a buffer, run <code class="language-plaintext highlighter-rouge">font-lock-ensure</code>, and assert
that specific character ranges have the expected face:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="p">(</span><span class="nv">when-fontifying-it</span> <span class="s">"fontifies let-bound functions"</span>
  <span class="p">(</span><span class="s">"let greet name = ..."</span>
   <span class="p">(</span><span class="mi">5</span> <span class="mi">9</span> <span class="nv">font-lock-function-name-face</span><span class="p">)))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p><strong>Indentation tests</strong> insert code, run <code class="language-plaintext highlighter-rouge">indent-region</code>, and assert the result
matches the expected indentation:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
4
</pre></td><td class="rouge-code"><pre><span class="p">(</span><span class="nv">when-indenting-it</span> <span class="s">"indents a match expression"</span>
  <span class="s">"match x with"</span>
  <span class="s">"| 0 -&gt; \"zero\""</span>
  <span class="s">"| n -&gt; string_of_int n"</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p><strong>Integration tests</strong> load real source files and verify that both font-locking
and indentation survive <code class="language-plaintext highlighter-rouge">indent-region</code> on the full file. This catches
interactions between rules that unit tests miss.</p>

<p><code class="language-plaintext highlighter-rouge">neocaml</code> has 200+ automated tests and <code class="language-plaintext highlighter-rouge">clojure-ts-mode</code> has even more.
Investing in test infrastructure early pays off enormously – I can refactor
indentation rules with confidence because the suite catches regressions
immediately.</p>

<h4>A personal story on testing ROI</h4>

<p>When I became the maintainer of
<a href="https://github.com/clojure-emacs/clojure-mode">clojure-mode</a> many years ago, I
really struggled with making changes. There were no font-lock or indentation
tests, so every change was a leap of faith – you’d fix one thing and break three
others without knowing until someone filed a bug report. I spent years working
on a testing approach I was happy with, alongside many great contributors, and
the return on investment was massive.</p>

<p>The same approach – almost the same test macros – carried over directly to
<code class="language-plaintext highlighter-rouge">clojure-ts-mode</code> when we built the Tree-sitter version. And later I reused the
pattern again in <code class="language-plaintext highlighter-rouge">neocaml</code> and <code class="language-plaintext highlighter-rouge">asciidoc-mode</code>. One investment in testing
infrastructure, four projects benefiting from it.</p>

<p>I know that automated tests, for whatever reason, never gained much traction in
the Emacs community. Many popular packages have no tests at all. I hope stories
like this convince you that investing in tests is really important and pays off
– not just for the project where you write them, but for every project you build
after.</p>

<h3>Pre-compile queries</h3>

<p>This one is specific to <code class="language-plaintext highlighter-rouge">clojure-ts-mode</code> but applies broadly: compiling
Tree-sitter queries at runtime is expensive. If you’re building queries
dynamically (e.g. with <code class="language-plaintext highlighter-rouge">treesit-font-lock-rules</code> called at mode init time),
consider pre-compiling them as <code class="language-plaintext highlighter-rouge">defconst</code> values. This made a noticeable
difference in <code class="language-plaintext highlighter-rouge">clojure-ts-mode</code>’s startup time.</p>

<h2>A note on naming</h2>

<p>The Emacs community has settled on a <code class="language-plaintext highlighter-rouge">-ts-mode</code> suffix convention for
Tree-sitter-based modes: <code class="language-plaintext highlighter-rouge">python-ts-mode</code>, <code class="language-plaintext highlighter-rouge">c-ts-mode</code>, <code class="language-plaintext highlighter-rouge">ruby-ts-mode</code>, and so
on. This makes sense when both a legacy mode and a Tree-sitter mode coexist in
Emacs core – users need to choose between them. But I think the convention is
being applied too broadly, and I’m afraid the resulting name fragmentation will
haunt the community for years.</p>

<p>For new packages that don’t have a legacy counterpart, the <code class="language-plaintext highlighter-rouge">-ts-mode</code> suffix is
unnecessary. I named my packages <code class="language-plaintext highlighter-rouge">neocaml</code> (not <code class="language-plaintext highlighter-rouge">ocaml-ts-mode</code>) and
<code class="language-plaintext highlighter-rouge">asciidoc-mode</code> (not <code class="language-plaintext highlighter-rouge">adoc-ts-mode</code>) because there was no prior <code class="language-plaintext highlighter-rouge">neocaml-mode</code>
or <code class="language-plaintext highlighter-rouge">asciidoc-mode</code> to disambiguate from. The <code class="language-plaintext highlighter-rouge">-ts-</code> infix is an implementation
detail that shouldn’t leak into the user-facing name. Will we rename everything
again when Tree-sitter becomes the default and the non-TS variants are removed?</p>

<p>Be bolder with naming. If you’re building something new, give it a name that
makes sense on its own merits, not one that encodes the parsing technology in the
package name.</p>

<h2>The road ahead</h2>

<p>I think the full transition to Tree-sitter in the Emacs community will take
3–5 years, optimistically. There are hundreds of major modes out there, many
maintained by a single person in their spare time. Converting a mode from regex
to Tree-sitter isn’t just a mechanical translation – you need to understand the
grammar, rewrite font-lock and indentation rules, handle version compatibility,
and build a new test suite. That’s a lot of work.</p>

<p>Interestingly, this might be one area where agentic coding tools can genuinely
help. The structure of Tree-sitter-based major modes is fairly uniform: grammar
recipes, font-lock rules, indentation rules, navigation settings, imenu. If you
give an AI agent a grammar and a reference to a high-quality mode like
<code class="language-plaintext highlighter-rouge">clojure-ts-mode</code>, it could probably scaffold a reasonable new mode fairly
quickly. The hard parts – debugging grammar quirks, handling edge cases, getting
indentation <em>just right</em> – would still need human attention, but the boilerplate
could be automated.</p>

<p>Still, knowing the Emacs community, I wouldn’t be surprised if a full migration
never actually completes. Many old-school modes work perfectly fine, their
maintainers have no interest in Tree-sitter, and “if it ain’t broke, don’t fix
it” is a powerful force. And that’s okay – diversity of approaches is part of
what makes Emacs Emacs.</p>

<h2>Closing thoughts</h2>

<p>Tree-sitter is genuinely great for building Emacs major modes. The code is
simpler, the results are more accurate, and incremental parsing means everything
stays fast even on large files. I wouldn’t go back to regex-based font-locking
willingly.</p>

<p>But it’s not magical. Grammars are inconsistent across languages, the Emacs APIs
are still maturing, you can’t reuse <code class="language-plaintext highlighter-rouge">.scm</code> files (yet), and you’ll hit
version-specific bugs that require tedious workarounds. The testing story is
better than with regex modes – tree structures are more predictable than regex
matches – but you still need a solid test suite to avoid regressions.</p>

<p>If you’re thinking about writing a Tree-sitter-based major mode, do it. The
ecosystem needs more of them, and the experience of working with syntax trees
instead of regexes is genuinely enjoyable. Just go in with realistic
expectations, pin your grammar versions, test against multiple Emacs releases,
and build your test suite early.</p>

<p>Anyways, I wish there was an article like this one when I was starting out
with <code class="language-plaintext highlighter-rouge">clojure-ts-mode</code> and <code class="language-plaintext highlighter-rouge">neocaml</code>, so there you have it. I hope that
the lessons I’ve learned along the way will help build better modes
with Tree-sitter down the road.</p>

<p>That’s all I have for you today. Keep hacking!</p>

<div class="footnotes" role="doc-endnotes">
  <ol>
    <li>
      <p>See the excellent <a href="https://github.com/sogaiu/tree-sitter-clojure/blob/master/doc/scope.md">scope discussion</a> in the tree-sitter-clojure repo for the rationale.&nbsp;<a href="https://batsov.com/feeds/OCaml.xml#fnref:1" class="reversefootnote" role="doc-backlink">↩︎</a></p>
    </li>
    <li>
      <p>There’s ongoing discussion in the Emacs community about distributing pre-compiled grammar binaries, but nothing concrete yet.&nbsp;<a href="https://batsov.com/feeds/OCaml.xml#fnref:2" class="reversefootnote" role="doc-backlink">↩︎</a></p>
    </li>
  </ol>
</div>
