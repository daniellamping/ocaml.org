---
title: Weeknotes 2026 week 13
description:
url: https://jon.recoil.org/blog/2026/03/weeknotes-2026-13.html
date: 2026-03-31T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#weeknotes-2026-week-13" class="anchor"></a>Weeknotes 2026 week 13</h1>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2026-03-31</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/weeknotes.html" title="weeknotes">weeknotes</a> <a href="https://jon.recoil.org/tags/tessera.html" title="tessera">tessera</a></p></li></ul>
<h2><a href="https://jon.recoil.org/atom.xml#what-did-i-do?" class="anchor"></a>What did I do?</h2>
<p>I spent rather a long time this week working an a review of the past few months of work, writing, rewriting, checking what I've written, rearranging, scratching my head and pondering. Too long. I did do some other stuff too though:</p>
<h3><a href="https://jon.recoil.org/atom.xml#standalone-tessera-page" class="anchor"></a>Standalone TESSERA page</h3>
<p>I had built the Js_top_worker code with the intention of having it runnable in any old web page, not necessarily one hosted on the same site as the code. However, I hadn't actually demonstrated this working, so I set up a tangled-hosted site at <a href="https://jonludlam.tngl.io">https://jonludlam.tngl.io</a> and put a basic Tessera notebook there. This (fortunately) worked more-or-less first time.</p>

<figure>
  <img src="https://jon.recoil.org/tessera-standalone.png">
  <figcaption><em>Standalone TESSERA notebook</em></figcaption>
</figure>

<p>The source for this is just a single html page. The repository is on <a href="https://tangled.org/jon.recoil.org/jonludlam.tngl.io/">tangled</a>.</p>
<h3><a href="https://jon.recoil.org/atom.xml#oxmono-docs-build" class="anchor"></a>Oxmono docs build</h3>
<p>Anil pointed out that his docs build in <a href="https://github.com/avsm/oxmono">oxmono</a> had stopped working - or perhaps never started working? I'd looked at this a couple of weeks back and found it was to do with the source rendering, which wasn't working correctly in the presence of multiple implementations of virtual libraries. I had "fixed" that on the 3.22 branch by disabling the source rendering rules by default, but I found out subsequently that dune 3.22 had some issues with oxcaml, so I needed to get the fix back onto the 3.21.1 branch. With that done, I tried running the docs build again, but it's OOMing on the rather limited (7G) workers that we get for free with GHA. Now it's definitely suspect that we can build the code itself but not the docs, so there's some odoc work to do there, but we should probably just build it on one of our servers before we tackle that.</p>
<h3><a href="https://jon.recoil.org/atom.xml#potential-scrolly-improvements" class="anchor"></a>Potential scrolly improvements</h3>
<p>While writing the review, I had another look at the examples I've got in the <a href="https://jon.recoil.org/reference/odoc-scrollycode-extension/index.html" title="index">Scrollycode plugin</a> and rediscovered that they are quite underwhelming. They're obviously Claude-generated placeholders, but I've always had a particular use-case in mind. About 10 years ago I gave a series of seminars at Citrix, where, using the database code as example, I showed how we can use GADTs to improve upon what we had. The talks were structured by starting with some very simple code and improving it gradually, in much the same way that the <a href="https://pomb.us/build-your-own-react/">build-your-own-react</a> example of scrollycode does. This seems to me a good way to validate the odoc plugin, so I've dug that out. What's missing from the plugin, though, is <i>modification</i> support - currently we can only <i>add</i> code, so that will need to be fixed first.</p>
<h2><a href="https://jon.recoil.org/atom.xml#what's-next?" class="anchor"></a>What's next?</h2>
<p>I've more-or-less achieve what I set out to do with the TESSERA notebook - I've demonstrated that OCaml is a perfectly capable platform for using in a notebook context to do purely client-side processing of geospatial data. For the next steps, I'm going to need to talk to my colleagues to figure out what the most useful thing to do is for TESSERA. There's plenty to do, but to prioritise everything correctly is going to need some discussion.</p>
<p>There are some open questions / areas to investigate in the purely notebook space:</p>
<ul><li>The worker and cma.js/cmi files have a dependency tree. Maybe it'd be interesting to publish the metadata for these using the at:// protocol?</li><li>Better discoverability. You can '#require' things, but what's available?</li><li>Docs links. Each of the packages I've published has the docs available. It'd be nice to link them up somehow.</li><li>AI agent integration. Most of the notebooks I've "written" recently have been generated by Claude. Can you paste in a token and get claude in your notebooks?</li><li>Reuse/remix of notebooks. The <a href="https://dl.acm.org/doi/10.1145/3759536.3763802">Fairground</a> paper emphasises that you can treat your notebooks as <i>libraries</i>. What does this look like for these?</li><li>Persistence? The exam-question notebooks in the examples do persist your edits right now, but that's ad-hoc and specific. What's the longer-term story?</li><li>Editing beyond the code blocks. What about editing the prose in the browser?</li></ul>
<p>Some more thoughts. Marimo uses python source as the primary source of the notebooks. One easy thing to do would be to have an <i>ml</i> file as the primary source for these. We've got easy access to the comments, which are already written in odoc markup. If we did this we'd get the power of merlin in the editor, which would be pretty nice.</p>
