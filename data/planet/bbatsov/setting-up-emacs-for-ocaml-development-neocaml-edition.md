---
title: 'Setting up Emacs for OCaml Development: Neocaml Edition'
description: "A few years ago I wrote about setting up Emacs for OCaml development.
  Back then the recommended stack was tuareg-mode + merlin-mode, with Merlin providing
  the bulk of the IDE experience. A lot has changed since then \u2013 the OCaml tooling
  has evolved considerably, and I\u2019ve been working on some new tools myself. Time
  for an update."
url: https://batsov.com/articles/2026/02/24/setting-up-emacs-for-ocaml-development-neocaml-edition/
date: 2026-02-24T10:00:00-00:00
preview_image: https://batsov.com/assets/img/og-image.png
authors:
- Bozhidar Batsov
source:
ignore:
---

<p>A few years ago I wrote about <a href="https://batsov.com/articles/2022/08/23/setting-up-emacs-for-ocaml-development/">setting up Emacs for OCaml
development</a>.
Back then the recommended stack was <code class="language-plaintext highlighter-rouge">tuareg-mode</code> + <code class="language-plaintext highlighter-rouge">merlin-mode</code>, with Merlin
providing the bulk of the IDE experience. A lot has changed since then – the
OCaml tooling has evolved considerably, and I’ve been working on some new tools
myself. Time for an update.</p>

<h2>The New Stack</h2>

<p>Here’s what I recommend today:</p>

<ul>
  <li><a href="https://github.com/bbatsov/neocaml">neocaml</a> instead of <code class="language-plaintext highlighter-rouge">tuareg-mode</code></li>
  <li><a href="https://github.com/tarides/ocaml-eglot">ocaml-eglot</a> instead of <code class="language-plaintext highlighter-rouge">merlin-mode</code></li>
  <li><a href="https://www.gnu.org/software/emacs/manual/html_mono/eglot.html">Eglot</a> (built into Emacs 29+) as the LSP client</li>
</ul>

<p>The key shift is from Merlin’s custom protocol to LSP.
<a href="https://github.com/ocaml/ocaml-lsp">ocaml-lsp-server</a> has matured
significantly since my original article – it’s no longer a thin wrapper around
Merlin. It now offers project-wide renaming, semantic highlighting, Dune RPC
integration, and OCaml-specific extensions like pattern match generation and
typed holes. <code class="language-plaintext highlighter-rouge">ocaml-eglot</code> is a lightweight Emacs package by
<a href="https://tarides.com/">Tarides</a> that bridges Eglot with these OCaml-specific
LSP extensions, giving you the full Merlin feature set through a standardized
protocol.</p>

<p>And <code class="language-plaintext highlighter-rouge">neocaml</code> is my own Tree-sitter-powered OCaml major mode – modern, lean,
and built for the LSP era. You can read more about it in the <a href="https://batsov.com/articles/2026/02/14/neocaml-0-1-ready-for-action/">0.1 release
announcement</a>.</p>

<h2>The Essentials</h2>

<p>First, install the server-side tools:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
</pre></td><td class="rouge-code"><pre><span class="gp">$</span><span class="w"> </span>opam <span class="nb">install </span>ocaml-lsp-server
</pre></td></tr></tbody></table></code></pre></div></div>

<blockquote class="prompt-tip">
  <p>You no longer need to install <code class="language-plaintext highlighter-rouge">merlin</code> separately – <code class="language-plaintext highlighter-rouge">ocaml-lsp-server</code>
vendors it internally.</p>
</blockquote>

<p>Then set up Emacs:</p>

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
12
</pre></td><td class="rouge-code"><pre><span class="c1">;; Modern Tree-sitter-powered OCaml major mode</span>
<span class="p">(</span><span class="nb">use-package</span> <span class="nv">neocaml</span>
  <span class="ss">:ensure</span> <span class="no">t</span><span class="p">)</span>

<span class="c1">;; Major mode for editing Dune project files</span>
<span class="p">(</span><span class="nb">use-package</span> <span class="nv">dune</span>
  <span class="ss">:ensure</span> <span class="no">t</span><span class="p">)</span>

<span class="c1">;; OCaml-specific LSP extensions via Eglot</span>
<span class="p">(</span><span class="nb">use-package</span> <span class="nv">ocaml-eglot</span>
  <span class="ss">:ensure</span> <span class="no">t</span>
  <span class="ss">:hook</span> <span class="p">(</span><span class="nv">neocaml-mode</span> <span class="o">.</span> <span class="nv">ocaml-eglot-setup</span><span class="p">))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>That’s it. Eglot ships with Emacs 29+, so there’s nothing extra to install for
the LSP client itself. When you open an OCaml file, Eglot will automatically
start <code class="language-plaintext highlighter-rouge">ocaml-lsp-server</code> and you’ll have completion, type information, code
navigation, diagnostics, and all the other goodies you’d expect.</p>

<p>Compare this to the old setup – no more <code class="language-plaintext highlighter-rouge">merlin-mode</code>, <code class="language-plaintext highlighter-rouge">merlin-eldoc</code>,
<code class="language-plaintext highlighter-rouge">flycheck-ocaml</code>, or manual Company configuration. LSP handles all of it
through a single, uniform interface.</p>

<h2>The Toplevel</h2>

<p><code class="language-plaintext highlighter-rouge">neocaml</code> includes built-in REPL integration via <code class="language-plaintext highlighter-rouge">neocaml-repl-minor-mode</code>. The
basics work well:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">C-c C-z</code> – start or switch to the OCaml toplevel</li>
  <li><code class="language-plaintext highlighter-rouge">C-c C-c</code> – send the current definition</li>
  <li><code class="language-plaintext highlighter-rouge">C-c C-r</code> – send the selected region</li>
  <li><code class="language-plaintext highlighter-rouge">C-c C-b</code> – send the entire buffer</li>
</ul>

<p>If you want <code class="language-plaintext highlighter-rouge">utop</code> specifically, you’re still better off using
<a href="https://github.com/ocaml-community/utop">utop.el</a> alongside <code class="language-plaintext highlighter-rouge">neocaml</code>. Its
main advantage is that you get code completion inside the <code class="language-plaintext highlighter-rouge">utop</code> REPL within
Emacs – something <code class="language-plaintext highlighter-rouge">neocaml</code>’s built-in REPL integration doesn’t provide:</p>

<div class="language-emacs-lisp highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
2
3
</pre></td><td class="rouge-code"><pre><span class="p">(</span><span class="nb">use-package</span> <span class="nv">utop</span>
  <span class="ss">:ensure</span> <span class="no">t</span>
  <span class="ss">:hook</span> <span class="p">(</span><span class="nv">neocaml-mode</span> <span class="o">.</span> <span class="nv">utop-minor-mode</span><span class="p">))</span>
</pre></td></tr></tbody></table></code></pre></div></div>

<p>This will shadow <code class="language-plaintext highlighter-rouge">neocaml</code>’s REPL keybindings with <code class="language-plaintext highlighter-rouge">utop</code>’s, which is the
intended behavior.</p>

<p>That said, as I’ve grown more comfortable with OCaml I find myself using the
toplevel less and less. These days I rely more on a test-driven workflow –
write a test, run it, iterate. In particular I’m partial to the workflow
described in <a href="https://discuss.ocaml.org/t/whats-your-development-workflow/10358/7">this OCaml Discuss
thread</a> –
running <code class="language-plaintext highlighter-rouge">dune runtest</code> continuously and writing expect tests for quick feedback.
It’s a more structured approach that scales better than REPL-driven development,
especially as your projects grow.</p>

<h2>Give neocaml a Try</h2>

<p>If you’re an OCaml programmer using Emacs, I’d love for you to take
<a href="https://github.com/bbatsov/neocaml">neocaml</a> for a spin. It’s available on
MELPA, so getting started is just an <code class="language-plaintext highlighter-rouge">M-x package-install</code> away. The project is
still young and I’m actively working on it – your feedback, bug reports, and
pull requests are invaluable. Let me know what works, what doesn’t, and what
you’d like to see next.</p>

<p>That’s all I have for you today. Keep hacking!</p>
