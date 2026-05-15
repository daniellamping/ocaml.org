---
title: Weeknotes May 2026 weeks 18-19
description:
url: https://jon.recoil.org/blog/2026/05/weeknotes-18-19.html
date: 2026-05-11T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#weeknotes-may-2026-weeks-18-19" class="anchor"></a>Weeknotes May 2026 weeks 18-19</h1>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2026-05-11</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/odoc.html" title="odoc">odoc</a> <a href="https://jon.recoil.org/tags/day11.html" title="day11">day11</a></p></li></ul>
<p>Over the past two weeks I've been mainly wrestling with odoc.</p>
<p>I made a release of odoc - <a href="https://github.com/ocaml/odoc/releases/3.2.0/">3.2.0</a> - and a <a href="https://github.com/ocaml/opam-repository/pull/29834">PR to opam-repository</a>. In parallel, I also just managed to make an ocaml-docs-ci job that tracked the master branch of odoc. I was pretty horrified therefore to see a glaring error in the docs build of <code>base</code>.</p>
<p>It turned out that one of the patches we made for OCaml 5.5.0, to support modular explicits, had an issue. It caught me by surprise because I didn't think that anything already existed that uses this, but in fact the new constructors in the AST are being used by</p>
