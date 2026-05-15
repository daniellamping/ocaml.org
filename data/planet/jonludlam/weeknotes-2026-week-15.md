---
title: Weeknotes 2026 week 15
description:
url: https://jon.recoil.org/blog/2026/04/weeknotes-2026-15.html
date: 2026-04-14T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#weeknotes-2026-week-15" class="anchor"></a>Weeknotes 2026 week 15</h1>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2026-04-14</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/day11.html" title="day11">day11</a> <a href="https://jon.recoil.org/tags/odoc.html" title="odoc">odoc</a></p></li></ul>
<p>Once again, the docs CI went down. This time, something had scribbled over the docker partition and so we needed to do a full build from scratch. Fortunately the docs themselves were not in a docker volume and so we didn't have to rebuild everything to get the HTTP server up and running for ocaml.org. However, we did have to set a full build going so that we can build docs for new packages.</p>
<p>This keeps happening, and is very annoying! So, that brings me onto the next topic: day11.</p>
<h2><a href="https://jon.recoil.org/atom.xml#day11" class="anchor"></a>Day11</h2>
<p>I've posted multiple times about <a href="https://tunbury.org/">Mark Elvers'</a> <a href="https://github.com/mtelvers/day10">day10</a> project. For me, it was an obvious extension of this to get it to build docs, and, as with most of my work recently, I set Claude on the task. However, Claude failed to integrate it nicely into the codebase. Taking a closer look at day10, it's a specialised tool that does its thing really well, but isn't built in a way that makes it easy to adapt in the ways that the docs needed. Clearly separating the AI generated code from the hand-written code is very important, so rather than going that route, I decided that I'd try to build a more general day10 with Claude - day11!</p>
<p>It's already at the point where it's been able to build the docs for all packages in opam-repository relatively quickly. Running it on <code>dill</code>, which is roughly equivalent to <code>sage</code>, where the docs are currently built, it takes about 6 hours or so to build everything, where <code>sage</code> with ocaml-docs-ci takes several days.</p>
<p>Some intriguing directions we might take this in:</p>
<ul><li>It's a generic build plaform for opam packages, therefore could possibly be used for easily executing binaries from opam packages.</li><li>It can build _itself_ - including new/different dependencies. Interesting for a self-modifying tool!</li><li>It's easy to drop into a container with precisely the correct dependencies for any package, so useful for debugging build failures. This is partially implemented already.</li><li>We can already provide overlay opam-repositories for testing of new/altered packages</li></ul>
<p>One really nice test of whether the organisation of the libraries in day11 is correct is whether we can plumb it into the docs-ci ocurrent pipeline easily, and have the CLI tools for it coexist nicely.</p>
<h2><a href="https://jon.recoil.org/atom.xml#odoc-performance" class="anchor"></a>Odoc performance</h2>
<p>Running odoc on with some of the oxcaml libraries exposes some critical weaknesses of the current code - in particular <a href="https://jon.recoil.org/03/weeknotes-2026-13.html#oxmono-docs-build" title="oxmono-docs-build">performance problems</a> with particular styles of code. We can't build the docs for Anil's <a href="https://github.com/avsm/oxmono">oxmono</a> repo with GHA as it simply runs out of memory. I've therefore been investigating some of the more egregious memory problems. I've got quite a few patches already with some good results, but nothing yet that's going to make it into upstream odoc without some more testing.</p>
<h2><a href="https://jon.recoil.org/atom.xml#other-bits-and-bobs" class="anchor"></a>Other bits and bobs</h2>
<p>I had fun hour or so putting together an odoc plugin to replicate the experience of davesnx's <a href="https://davesnx.github.io/parseff">parseff site</a>. The plugin is <a href="https://tangled.org/jon.recoil.org/odoc-parseff-shell/">here</a>, and to use it, see my modified parseff repo:</p>
<div><pre class="language-shell"><code>git clone https://github.com/jonludlam/parseff.git#odoc-plugins
opam switch create . --with-doc
dune build @doc</code></pre></div>

<figure>
  <a href="https://jon.ludl.am/experiments/parseff"><img src="https://jon.recoil.org/parseff.png" alt="A screenshot of the parseff site"></a>
  <figcaption><em>A screenshot of the parseff site produced by the plugin</em></figcaption>
</figure>

<p>There are a number of advantages and disadvantages to this. As @davesnx <a href="https://sancho.dev/blog/ocaml-documentation-as-markdown">wrote</a>, his concern with the markdown output was to be able to integrate the odoc output seamlessly with an existing site, and it does this very well. However, it's at a cost - we lose links in the API docs, links to source, the source rendering itself, and so on. Whereas the plugin I made keeps all of those nice features, but is still tricky to integrate with a larger site.</p>
