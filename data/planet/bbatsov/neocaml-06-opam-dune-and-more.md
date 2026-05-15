---
title: 'Neocaml 0.6: Opam, Dune, and More'
description: "When I released neocaml 0.1 last month, I thought I was more or less
  done with the (main) features for the foreseeable future. The original scope was
  deliberately small \u2014 a couple of Tree-sitter-powered OCaml major modes (for
  .ml and .mli), a REPL integration, and not much else. I was quite happy with how
  things turned out and figured the next steps would be mostly polish and bug fixes."
url: https://batsov.com/articles/2026/03/25/neocaml-0-6-opam-dune-and-more/
date: 2026-03-25T08:00:00-00:00
preview_image: https://batsov.com/assets/img/og-image.png
authors:
- Bozhidar Batsov
source:
ignore:
---

<p>When I released <a href="https://batsov.com/articles/2026/02/14/neocaml-0-1-ready-for-action/">neocaml 0.1</a>
last month, I thought I was more or less done with the (main) features for the
foreseeable future. The original scope was deliberately small — a couple of
Tree-sitter-powered OCaml major modes (for <code class="language-plaintext highlighter-rouge">.ml</code> and <code class="language-plaintext highlighter-rouge">.mli</code>), a REPL
integration, and not much else. I was quite happy with how things turned out and
figured the next steps would be mostly polish and bug fixes.</p>

<p>Versions 0.2-0.5 brought polish and bug fixes, but fundamentally the feature
set stayed the same. I was even more convinced a grand 1.0 release was just
around the corner.</p>

<p>I was wrong.</p>

<p>Of course, OCaml files don’t exist in isolation. They live alongside
<a href="https://opam.ocaml.org/">Opam</a> files that describe packages and
<a href="https://dune.readthedocs.io/">Dune</a> files that configure builds. And as I was
poking around the Tree-sitter ecosystem, I discovered that there were already
grammars for both
<a href="https://github.com/tmcgilchrist/tree-sitter-opam">Opam</a> and
<a href="https://github.com/tmcgilchrist/tree-sitter-dune">Dune</a> files. Given how
simple both formats are (Opam is mostly key-value pairs, Dune is
s-expressions), adding support for them turned out to be fairly
straightforward.</p>

<p>So here we are with neocaml 0.6, which is quite a bit bigger than I expected
originally.</p>

<p><strong>Note:</strong> One thing worth mentioning — all the new modes are completely
isolated from the core OCaml modes. They’re separate files with no hard
dependency on <code class="language-plaintext highlighter-rouge">neocaml-mode</code>, loaded only when you open the relevant file
types. I didn’t want to force them upon anyone — for me it’s convenient to get
Opam and Dune support out-of-the-box (given how ubiquitous they are in the OCaml
ecosystem), but I totally get it if someone doesn’t care about this.</p>

<p>Let me walk you through what’s new.</p>

<h2>neocaml-opam-mode</h2>

<p>The new <code class="language-plaintext highlighter-rouge">neocaml-opam-mode</code> activates automatically for <code class="language-plaintext highlighter-rouge">.opam</code> and <code class="language-plaintext highlighter-rouge">opam</code>
files. It provides:</p>

<ul>
  <li>Tree-sitter-based font-lock (field names, strings, operators, version
constraints, filter expressions, etc.)</li>
  <li>Indentation (lists, sections, option braces)</li>
  <li>Imenu for navigating variables and sections</li>
  <li>A <strong>flymake backend</strong> that runs <code class="language-plaintext highlighter-rouge">opam lint</code> on the current buffer, giving you
inline diagnostics for missing fields, deprecated constructs, and syntax
errors</li>
</ul>

<p>The flymake backend registers automatically when <code class="language-plaintext highlighter-rouge">opam</code> is found in your PATH,
but you need to enable <code class="language-plaintext highlighter-rouge">flymake-mode</code> yourself:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
</pre></td><td class="rouge-code"><pre><span class="p">(</span><span class="nv">add-hook</span> <span class="ss">'neocaml-opam-mode-hook</span> <span class="nf">#'</span><span class="nv">flymake-mode</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p><a href="https://github.com/flycheck/flycheck">Flycheck</a> users get <code class="language-plaintext highlighter-rouge">opam lint</code> support
out of the box via Flycheck’s built-in <code class="language-plaintext highlighter-rouge">opam</code> checker — no extra configuration
needed.</p>

<p>This bridges some of the gap with <a href="https://github.com/ocaml/tuareg">Tuareg</a>,
which also bundles an Opam major mode (<code class="language-plaintext highlighter-rouge">tuareg-opam-mode</code>). The Tree-sitter-based
approach gives us more accurate highlighting, and the flymake integration is a
nice bonus on top.</p>

<h2>neocaml-dune-mode</h2>

<p><code class="language-plaintext highlighter-rouge">neocaml-dune-mode</code> handles <code class="language-plaintext highlighter-rouge">dune</code>, <code class="language-plaintext highlighter-rouge">dune-project</code>, and <code class="language-plaintext highlighter-rouge">dune-workspace</code> files
— all three use the same s-expression syntax and share a single Tree-sitter
grammar. You get:</p>

<ul>
  <li>Font-lock for stanza names, field names, action keywords, strings, module
names, library names, operators, and brackets</li>
  <li>Indentation with 1-space offset</li>
  <li>Imenu for stanza navigation</li>
  <li>Defun navigation and <code class="language-plaintext highlighter-rouge">which-func</code> support</li>
</ul>

<p>This removes the need to install the separate
<a href="https://melpa.org/#/dune">dune</a> package (the standalone <code class="language-plaintext highlighter-rouge">dune-mode</code> maintained
by the Dune developers) from MELPA. If you prefer to keep using it, that’s fine
too — neocaml’s README has
<a href="https://github.com/bbatsov/neocaml#using-the-legacy-dune-mode">instructions</a>
for overriding the <code class="language-plaintext highlighter-rouge">auto-mode-alist</code> entries.</p>

<h2>neocaml-dune-interaction-mode</h2>

<p>Beyond editing Dune files, I wanted a simple way to run Dune commands from any
neocaml buffer. <code class="language-plaintext highlighter-rouge">neocaml-dune-interaction-mode</code> is a minor mode that provides
keybindings (under <code class="language-plaintext highlighter-rouge">C-c C-d</code>) and a “Dune” menu for common operations:</p>

<table>
  <thead>
    <tr>
      <th>Keybinding</th>
      <th>Command</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">C-c C-d b</code></td>
      <td>Build</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">C-c C-d t</code></td>
      <td>Test</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">C-c C-d c</code></td>
      <td>Clean</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">C-c C-d p</code></td>
      <td>Promote</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">C-c C-d f</code></td>
      <td>Format</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">C-c C-d u</code></td>
      <td>Launch utop with project libraries</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">C-c C-d r</code></td>
      <td>Run an executable</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">C-c C-d d</code></td>
      <td>Run any Dune command</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">C-c C-d .</code></td>
      <td>Find the nearest <code class="language-plaintext highlighter-rouge">dune</code> file</td>
    </tr>
  </tbody>
</table>

<p>All commands run via Emacs’s <code class="language-plaintext highlighter-rouge">compile</code>, so you get error navigation, clickable
source locations, and the full <code class="language-plaintext highlighter-rouge">compilation-mode</code> interface for free. With a
prefix argument (<code class="language-plaintext highlighter-rouge">C-u</code>), build, test, and fmt run in <strong>watch mode</strong>
(<code class="language-plaintext highlighter-rouge">--watch</code>), automatically rebuilding when files change.</p>

<p>The <code class="language-plaintext highlighter-rouge">utop</code> command is special — it launches through <code class="language-plaintext highlighter-rouge">neocaml-repl</code>, so you get
the full REPL integration (send region, send definition, etc.) with your
project’s libraries preloaded.</p>

<p>This mode is completely independent from <code class="language-plaintext highlighter-rouge">neocaml-dune-mode</code> — it doesn’t care
which major mode you’re using. You can enable it in OCaml buffers like this:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
</pre></td><td class="rouge-code"><pre><span class="p">(</span><span class="nv">add-hook</span> <span class="ss">'neocaml-base-mode-hook</span> <span class="nf">#'</span><span class="nv">neocaml-dune-interaction-mode</span><span class="p">)</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<h2>Rough Edges</h2>

<p>Both the Opam and Dune Tree-sitter grammars are relatively young and will need
some more work for optimal results. I’ve been filing issues and contributing
patches upstream to improve them — for instance, the Dune grammar currently
<a href="https://github.com/tmcgilchrist/tree-sitter-dune/issues/9">flattens field-value
pairs</a> in a way
that makes indentation less precise than it could be, and neither grammar
supports <a href="https://github.com/tmcgilchrist/tree-sitter-dune/issues/10">variable
interpolation</a>
(<code class="language-plaintext highlighter-rouge">%{...}</code>) yet. These are very solvable problems and I expect the grammars to
improve over time.</p>

<h2>What’s Next?</h2>

<p>At this point I think I’m (finally!) out of ideas for new functionality. This
time I mean it! Neocaml now covers pretty much everything I ever wanted,
especially when paired with the awesome
<a href="https://github.com/tarides/ocaml-eglot">ocaml-eglot</a>.</p>

<p>Down the road there might be support for OCamllex (<code class="language-plaintext highlighter-rouge">.mll</code>) or Menhir (<code class="language-plaintext highlighter-rouge">.mly</code>)
files, but only if adding them doesn’t bring significant complexity — both are
mixed languages with embedded OCaml code, which makes them fundamentally harder
to support well than the simple Opam and Dune formats.</p>

<p>I hope OCaml programmers will find the new functionality useful. If you’re using neocaml,
I’d love to hear how it’s working for you — bug reports, feature requests, and
general feedback are all welcome on
<a href="https://github.com/bbatsov/neocaml">GitHub</a>. You can find the full list of
changes in the
<a href="https://github.com/bbatsov/neocaml/blob/main/CHANGELOG.md">changelog</a>.</p>

<p>As usual — update from <a href="https://melpa.org/#/neocaml">MELPA</a>, kick the tires,
and let me know what you think.</p>

<p>That’s all I have for you today! Keep hacking!</p>
