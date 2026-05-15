---
title: Weeknotes 2026 weeks 16-17
description:
url: https://jon.recoil.org/blog/2026/04/weeknotes-2026-16-17.html
date: 2026-04-28T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#weeknotes-2026-weeks-16-17" class="anchor"></a>Weeknotes 2026 weeks 16-17</h1>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2026-04-28</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/odoc.html" title="odoc">odoc</a> <a href="https://jon.recoil.org/tags/day11.html" title="day11">day11</a></p></li></ul>
<p>A two week update this week. Most of this fortnight has been spent on different sides of the same problem: getting OCaml documentation into a state where an LLM (or a human) can actually rely on it — search, packaging, performance, and infrastructure.</p>
<h2><a href="https://jon.recoil.org/atom.xml#docs-and-llms" class="anchor"></a>Docs and LLMs</h2>
<p>It seems fairly obvious, but having docs available for libraries is very important for LLMs to be able to use them effectively, especially if they're new or private. There have been various papers on the subject — for instance the <a href="https://arxiv.org/abs/2207.05987">2022 paper on tool documentation</a> and the more recent <a href="https://arxiv.org/abs/2503.15231">work on the importance of example code</a> — as well as our own <a href="https://toao.com/blog/ai-existential-ocaml">contribution</a> to the OCaml workshop last year.</p>
<p>Examples in OCaml libraries often live in <a href="https://ocaml.org/docs/generating-documentation#generating-mld-documentation-pages">mld files</a>, and the correctness of these can be automatically tested using <a href="https://github.com/realworldocaml/mdx">mdx</a>, which works well on mld files.</p>
<p><a href="https://doc.sherlocode.com">Sherlodoc</a> deliberately skips mld files, because plain English in mld pages drowns the API hits — when indexing was originally enabled it ended up polluting the API results too much, with generic text appearing way too often. Sherlodoc doesn't do any of the interesting stemming or other BM25-style weighting, so as more mld content lands in packages we'll likely need to investigate a hybrid approach to searching the docs.</p>
<h2><a href="https://jon.recoil.org/atom.xml#oi!" class="anchor"></a>Oi!</h2>
<p>Anil's just <a href="https://anil.recoil.org/notes/2026w16">shipped</a> a neat new tool in OCaml land — <a href="https://github.com/avsm/oi">oi!</a>, built on top of <a href="https://github.com/mtelvers/day10">day10</a> and <a href="https://www.dra27.uk/blog/platform/2025/12/17/its-merged.html">relocatable OCaml</a>. This is a <i>really</i> neat little tool that you can use to run OCaml tools and scripts without having a dedicated opam setup.</p>
<p>As I've been getting my Claude-built day10 successor, <a href="https://jon.recoil.org/atom.xml#day11" title="day11">day11</a>, into shape as a replacement for the guts of the current ocaml-docs-ci, I thought I'd check to see if the libraries I'd made could work as a drop-in for the d10 part of oi (Anil's vendored copy of the relevant bits of day10). This indeed worked nicely, without huge impact on the rest of oi (aside from excising a chunk of it, of course).</p>
<p>Next step is to see if we can get decent doc support into oi. It'd be great to be able to run <code>oi doc</code> or <code>oi doc search</code> in a project and have accurate docs pop out. More on this to come!</p>
<h2><a href="https://jon.recoil.org/atom.xml#odoc-performance-investigation" class="anchor"></a>Odoc performance investigation</h2>
<p>A particular problem with getting the docs to 'just pop out' is that some docs take a long time and too much memory to build with odoc. So much so that the Github Action workflow that generates them can just die, particularly when trying to build the docs for the oxcaml branches of base and core. So I've spent a little time investigating this over the last couple of weeks, and made some quite significant progress.</p>
<p>Headline figures on my laptop: wall time 549 s → 474 s (−14%), total allocation 381 GB → 230 GB (−40%), peak RSS 2.00 GB → 1.81 GB.</p>
<p>Particularly bad for odoc was an include-expansion explosion: a single source line in container_intf.ml was being walked 10,777 times during html-generate, because each ppx_template monomorphisation produced its own Include whose expansion nested further Includes. I had previously noted this issue and tackled the most awful of the problems, but there was more to fix!</p>
<p>The includes are effectively a workaround for the lack of layout polymorphism in the current oxcaml compiler. A quick investigation showed that of the 155,828 doc comments in container_intf.ml there were only 33 unique strings, once the ppx_template duplication was de-duped. Memoising the parsing of these led to a substantial improvement.</p>
<p>In addition, many of the includes end up flattened in the resulting HTML due to their module-type being either a signature or pointing at a hidden item. Includes come with a fair bit of overhead, so rather than flattening them at the point we're generating the HTML, we can spot this pattern and flatten them much earlier in the process.</p>
<p>The single biggest html-generate win was a one-liner. <a href="https://github.com/ocaml/odoc/blob/master/src/html/link.ml#L9">segment_to_string</a> was doing <code>Format.asprintf "%a%s"</code> where the <code>%a</code> formatter just emits <code>"&lt;kind&gt;-"</code> or nothing. A direct pattern-match killed ~60% of html-generate allocation on stdlib, ~50% on core.</p>
<p>I'll tidy up these patches, double check that they don't affect the output, and see how much of an improvement they make on Anil's <a href="https://github.com/avsm/oxmono">oxmono</a> repo.</p>
<h2><a href="https://jon.recoil.org/atom.xml#day11" class="anchor"></a>Day11</h2>
<p>I mentioned <a href="https://jon.recoil.org/weeknotes-2026-15.html" title="weeknotes-2026-15">last time</a> about replacing the innards of <a href="https://github.com/ocurrent/ocaml-docs-ci">ocaml-docs-ci</a> with the day11 libraries. This has worked out really nicely. The shape of the resulting tool is that we've got an ocurrent pipeline that watches <a href="https://github.com/ocaml/opam-repository">the opam-repository</a> on GitHub, and triggers the day11 build when it notices a change. Each layer corresponds to an ocurrent job. On top of the normal UI I've added in some pages that make it easier to spot emerging problems in the docs build. My ultimate plan is to put an LLM in charge of this so it can watch the status of it every day, and then let me know if it thinks I need to do something!</p>
<ul class="at-tags"><li class="figure"><span class="at-tag">figure</span> <p>docs-ci.png Docs CI screenshot Docs CI showing the status of the current "snapshot"</p></li></ul>
<h2><a href="https://jon.recoil.org/atom.xml#improved-site" class="anchor"></a>Improved site</h2>
<p>I made some small improvements to the site too. I added a <code>@figure</code> plugin, and a margin notes plugin <code>{&amp;margin Margin notes look like this!}</code> Margin notes look like this!, and added tags to my pages. You're already reading the result — the Docs CI screenshot above uses the new <code>@figure</code> plugin, and you can see the margin notes.</p>
<p>Until next week!</p>
