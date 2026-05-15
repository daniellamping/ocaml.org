---
title: Weeknotes 2026 week 11
description:
url: https://jon.recoil.org/blog/2026/03/weeknotes-2026-11.html
date: 2026-03-18T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#weeknotes-2026-week-11" class="anchor"></a>Weeknotes 2026 week 11</h1>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2026-03-18</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/weeknotes.html" title="weeknotes">weeknotes</a> <a href="https://jon.recoil.org/tags/tessera.html" title="tessera">tessera</a></p></li></ul>
<ul class="at-tags"><li class="notanotebook"><span class="at-tag">notanotebook</span> </li></ul>
<h2><a href="https://jon.recoil.org/atom.xml#what-did-i-do?" class="anchor"></a>What did I do?</h2>
<h3><a href="https://jon.recoil.org/atom.xml#tessera" class="anchor"></a>TESSERA</h3>
<p>I Started looking at fixing my TESSERA notebook to make it actually correct:</p>
<ul><li>Removing the OCaml native PCA to replace with a tensorflow.js one. This should be much faster and probably more accurate.</li><li>Investigating the slightly-off-seeming embeddings - you can see features like roads on the embeddings and they don't <em>quite</em> match up with the map. The problem is to do with WGS84 vs UTM, which I need sort out.</li><li>Looking at UNets as part of replicating Sadiq's <a href="https://github.com/sadiqj/tessera-cnn-example">Solar Panel</a> detection example.</li></ul>
<p>Not much to show yet though.</p>
<h3><a href="https://jon.recoil.org/atom.xml#odoc-release" class="anchor"></a>Odoc release</h3>
<p>We need an Odoc release soon, partly to support markdown output in dune, and partly to fix some bugs that were discovered when doing the odoc v3 dune rules. I merged a lot of stuff this week, but we've still got Arthur's OxCaml patches to merge, which are waiting on a test fix. I also looked into OCaml 5.5 compatibility. The big item looks to be the merge of Modular Explicits, which will require a bit of work to support. I'm considering a mechanical fix to at least get odoc compiling with 5.5 for the odoc release, and a larger release later to properly support the 5.5 features.</p>
<h3><a href="https://jon.recoil.org/atom.xml#oxcaml-docs-builds" class="anchor"></a>OxCaml docs builds</h3>
<p>Anil has merged <a href="https://github.com/avsm/oxmono/pull/3">my PR</a> to his oxmono repo to do docs, and that works nicely if you're generating <i>all</i> of the docs for your monorepo, including dependencies, which is what you get when you run <code>dune build @doc-full</code>. Running <code>dune build @doc</code> doesn't quite work as there's no external place like ocaml.org where you can expect your dependency libraries docs to be. Mark's <code>day10</code> project is already building all of the oxcaml packages, so I had a look to see what it would be like to extend that to build the docs too. This is more-or-less working, but not yet as a reliable place that you can expect to be up to date. I had a thought that keeping the docs build running nicely is something that an AI agent should be able to do really well, so I'm investigating this. A neat thing it's already been able to do is to sort the build problems by number of other packages that they block, and numbers one and two were PPX issues (ppx_deriving_yojson and lwt_ppx). It was then able to come up with some patches that enabled both of these to build, unblocking all of those downstream packages. This, of course, is of wider interest than just the docs, so I need to figure out what to do with those patches.</p>
<h3><a href="https://jon.recoil.org/atom.xml#bugfixing-my-monorepo" class="anchor"></a>Bugfixing my monorepo</h3>
<p>I've been polishing the monorepo that's powering my website with a view to publishing a retrospective of the work that I've been doing over the past few months. Most importantly, adding '--warn-error' to all my odoc invocations to actually notice all the warnings it's been diligently reporting, and I've been ignoring. Looking further than my own failings though, this has really helped solidify in my mind the areas where Claude excels and the things we should be much more wary of letting it do. More on this in the retrospective!</p>
