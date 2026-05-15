---
title: 'Neocaml 0.1: Ready for Action'
description: "neocaml 0.1 is finally out! Almost a year after I announced the project,
  I\u2019m happy to report that it has matured to the point where I feel comfortable
  calling it ready for action. Even better - neocaml recently landed in MELPA, which
  means installing it is now as easy as:"
url: https://batsov.com/articles/2026/02/14/neocaml-0-1-ready-for-action/
date: 2026-02-14T16:34:00-00:00
preview_image: https://batsov.com/assets/img/og-image.png
authors:
- Bozhidar Batsov
source:
ignore:
---

<p><a href="https://github.com/bbatsov/neocaml">neocaml</a> 0.1 is finally out! Almost a year
after I <a href="https://batsov.com/articles/2025/03/14/neocaml-a-new-emacs-package-for-ocaml-programming/">announced the
project</a>,
I’m happy to report that it has matured to the point where I feel comfortable
calling it ready for action. Even better - <code class="language-plaintext highlighter-rouge">neocaml</code> recently landed in
<a href="https://melpa.org/#/neocaml">MELPA</a>, which means installing it is now as easy
as:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code><table class="rouge-table"><tbody><tr><td class="rouge-gutter gl"><pre class="lineno">1
</pre></td><td class="rouge-code"><pre>M-x package-install &lt;RET&gt; neocaml &lt;RET&gt;
</pre></td></tr></tbody></table></code></pre></div></div>

<p>That’s quite the journey from “a fun experimental project” to a proper Emacs
package!</p>

<h2>Why neocaml?</h2>

<p>You might be wondering what’s wrong with the existing options. The short answer -
nothing is <em>wrong</em> per se, but <code class="language-plaintext highlighter-rouge">neocaml</code> offers a different set of trade-offs:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">caml-mode</code> is ancient and barely maintained. It lacks many features that
modern Emacs users expect and it probably should have been deprecated a long time ago.</li>
  <li><code class="language-plaintext highlighter-rouge">tuareg-mode</code> is very powerful, but also very complex. It carries a lot of
legacy code and its regex-based font-locking and custom indentation engine show
their age. It’s a beast - in both the good and the bad sense of the word.</li>
  <li><code class="language-plaintext highlighter-rouge">neocaml</code> aims to be a modern, lean alternative that fully embraces
Tree-sitter. The codebase is small, well-documented, and easy to hack on. If
you’re running Emacs 29+ (and especially Emacs 30), Tree-sitter is the future
and <code class="language-plaintext highlighter-rouge">neocaml</code> is built entirely around it.</li>
</ul>

<p>Of course, <code class="language-plaintext highlighter-rouge">neocaml</code> is the youngest of the bunch and it doesn’t yet match
Tuareg’s feature completeness. But for many OCaml workflows it’s already more
than sufficient, especially when combined with LSP support.</p>

<p>I’ve started the project mostly because I thought that the existing
Emacs tooling for OCaml was somewhat behind the times - e.g. both
<code class="language-plaintext highlighter-rouge">caml-mode</code> and <code class="language-plaintext highlighter-rouge">tuareg-mode</code> have features that are no longer needed
in the era of <code class="language-plaintext highlighter-rouge">ocamllsp</code>.</p>

<p>Let me now walk you through the highlights of version 0.1.</p>

<h2>Features</h2>

<p>The current feature-set is relatively modest, but all the essential functionality
one would expect from an Emacs major mode is there.</p>

<h3>Tree-sitter-powered Syntax Highlighting</h3>

<p><code class="language-plaintext highlighter-rouge">neocaml</code> leverages Tree-sitter for syntax highlighting, which is both more
accurate and more performant than the traditional regex-based approaches used by
<code class="language-plaintext highlighter-rouge">caml-mode</code> and <code class="language-plaintext highlighter-rouge">tuareg-mode</code>. The font-locking supports 4 customizable
intensity levels (controlled via <code class="language-plaintext highlighter-rouge">treesit-font-lock-level</code>, default 3), so you
can pick the amount of color that suits your taste.</p>

<p>Both <code class="language-plaintext highlighter-rouge">.ml</code> (source) and <code class="language-plaintext highlighter-rouge">.mli</code> (interface) files get their own major modes with
dedicated highlighting rules.</p>

<h3>Tree-sitter-powered Indentation</h3>

<p>Indentation has always been tricky for OCaml modes, and I won’t pretend it’s
perfect yet, but <code class="language-plaintext highlighter-rouge">neocaml</code>’s Tree-sitter-based indentation engine is already quite
usable. It also supports cycle-indent functionality, so hitting <code class="language-plaintext highlighter-rouge">TAB</code> repeatedly
will cycle through plausible indentation levels - a nice quality-of-life feature
when the indentation rules can’t fully determine the “right” indent.</p>

<p>If you prefer, you can still delegate indentation to external tools like
<code class="language-plaintext highlighter-rouge">ocp-indent</code> or even Tuareg’s indentation functions. Still, I think most people
will be quite satisfied with the built-in indentation logic.</p>

<h3>Code Navigation and Imenu</h3>

<p><code class="language-plaintext highlighter-rouge">neocaml</code> provides proper structural navigation commands (<code class="language-plaintext highlighter-rouge">beginning-of-defun</code>,
<code class="language-plaintext highlighter-rouge">end-of-defun</code>, <code class="language-plaintext highlighter-rouge">forward-sexp</code>) powered by Tree-sitter, plus <code class="language-plaintext highlighter-rouge">imenu</code> integration –
navigating definitions in a buffer has never been easier.</p>

<p>The older modes provide very similar functionality as well, of course,
but the use of Tree-sitter in <code class="language-plaintext highlighter-rouge">neocaml</code> makes such commands more reliable and
robust.</p>

<h3>REPL Integration</h3>

<p>No OCaml mode would be complete without REPL (toplevel) integration. <code class="language-plaintext highlighter-rouge">neocaml-repl-minor-mode</code>
provides all the essentials:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">C-c C-z</code> - Start or switch to the OCaml REPL</li>
  <li><code class="language-plaintext highlighter-rouge">C-c C-c</code> - Send the current definition</li>
  <li><code class="language-plaintext highlighter-rouge">C-c C-r</code> - Send the selected region</li>
  <li><code class="language-plaintext highlighter-rouge">C-c C-b</code> - Send the entire buffer</li>
  <li><code class="language-plaintext highlighter-rouge">C-c C-p</code> - Send a phrase (code until <code class="language-plaintext highlighter-rouge">;;</code>)</li>
</ul>

<p>The default REPL is <code class="language-plaintext highlighter-rouge">ocaml</code>, but you can easily switch to <code class="language-plaintext highlighter-rouge">utop</code> via
<code class="language-plaintext highlighter-rouge">neocaml-repl-program-name</code>.</p>

<p>I’m still on the fence on whether I want to invest time into making the REPL-integration
more powerful or keep it as simple as possible. Right now it’s definitely not a big
priority for me, but I want to match what the other older OCaml modes offered in that regard.</p>

<h3>LSP Support</h3>

<p><code class="language-plaintext highlighter-rouge">neocaml</code> works great with <a href="https://www.gnu.org/software/emacs/manual/html_mono/eglot.html">Eglot</a> and
<code class="language-plaintext highlighter-rouge">ocamllsp</code>, automatically setting the appropriate language IDs for both <code class="language-plaintext highlighter-rouge">.ml</code> and
<code class="language-plaintext highlighter-rouge">.mli</code> files. Pair <code class="language-plaintext highlighter-rouge">neocaml</code> with
<a href="https://github.com/tarides/ocaml-eglot">ocaml-eglot</a> and you get a pretty
solid OCaml development experience.</p>

<p>The creation of LSP really simplified the lives of major mode authors like me, as now
many of the features that were historically major mode specific are provided by
LSP clients out-of-the-box.</p>

<p>That’s also another reason why you probably want a leaner major mode like <code class="language-plaintext highlighter-rouge">neocaml-mode</code>.</p>

<h3>Other Goodies</h3>

<p>But, wait, there’s more!</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">C-c C-a</code> to quickly switch between <code class="language-plaintext highlighter-rouge">.ml</code> and <code class="language-plaintext highlighter-rouge">.mli</code> files</li>
  <li>Prettify-symbols support for common OCaml operators</li>
  <li>Automatic installation of the required Tree-sitter grammars via <code class="language-plaintext highlighter-rouge">M-x neocaml-install-grammars</code></li>
  <li>Compatibility with <a href="https://github.com/ocaml/merlin">Merlin</a> for those who prefer it over LSP</li>
</ul>

<h2>The Road Ahead</h2>

<p>There’s still plenty of work to do:</p>

<ul>
  <li>Support for additional OCaml file types (e.g. <code class="language-plaintext highlighter-rouge">.mld</code>)</li>
  <li>Improvements to structured navigation using newer Emacs Tree-sitter APIs</li>
  <li>Improvements to the test suite</li>
  <li>Addressing feedback from real-world OCaml users</li>
  <li>Actually writing some fun OCaml code with <code class="language-plaintext highlighter-rouge">neocaml</code></li>
</ul>

<p>If you’re following me, you probably know that I’m passionate about both Emacs
and OCaml. I hope that <code class="language-plaintext highlighter-rouge">neocaml</code> will be my way to contribute to the awesome
OCaml community.</p>

<p>I’m not sure how quickly things will move, but I’m committed to making <code class="language-plaintext highlighter-rouge">neocaml</code>
the best OCaml editing experience on Emacs. Time will tell how far I’ll get!</p>

<h2>Give it a Try</h2>

<p>If you’re an OCaml programmer using Emacs, I’d love for you to take <code class="language-plaintext highlighter-rouge">neocaml</code> for
a spin. Install it from MELPA, kick the tires, and let me know what you think.
Bug reports, feature requests, and pull requests are all most welcome on
<a href="https://github.com/bbatsov/neocaml">GitHub</a>!</p>

<p>That’s all from me, folks! Keep hacking!</p>
