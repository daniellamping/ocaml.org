---
title: '.plan-26-05: An OxCaml hacking week'
description: Deploying an OxCaml zero-allocation webserver, OCaml CI maintenance and
  opam versioning, and OCaml Workshop and FOSDEM talks
url: https://anil.recoil.org/notes/2026w5
date: 2026-02-01T00:00:00-00:00
preview_image: https://anil.recoil.org/images/videos/905d3833-a890-4ece-8ba2-cf6dbf5e2dcb.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<h2><a href="https://anil.recoil.org/news.xml#deploying-a-zero-allocation-oxcaml-webserver" class="anchor" aria-hidden="true"></a>Deploying a zero allocation OxCaml webserver</h2>
<p>I decided to spend this week on as much focussed hacking as I could, and in particular finished up switching my website to <a href="https://anil.recoil.org/notes/oxcaml-httpz">a new webserver in OxCaml</a>. This attracted a lot of attention, so I spent a surprising amount of time answering questions on the socials about it! If you're reading this site, then it also works...</p>
<p>One funny thing that happened right after deploying it was that I noticed tens of thousands of concurrent connections opened. It turns out that <a href="https://moltbook.com">Moltbook</a> had a sub-molt used by agents that track Hackernews somehow, and a bunch of them had decided to mine my website for ... something. The switch to agents dominating the Internet are arriving rapidly!</p>
<p>The actual development of httpz is happening in a new <a href="https://github.com/avsm/oxmono">oxcaml monorepo</a> I've opened. I've not abandoned my older way of publishing to opam as well, but <a href="https://jon.recoil.org">Jon Ludlam</a> and <a href="https://github.com/samoht">Thomas Gazagnaire</a> are refining <a href="https://jon.recoil.org/blog/2026/01/weeknotes-2026-04-05.html">that approach</a>. When coding in OxCaml, I need to fork almost every library, so a monorepo is the only way to go!</p>
<p>On a more human note, it was delightful to see <a href="https://jonmsterling.com">Jon Sterling</a> <a href="https://www.jonmsterling.com/2026-W05/">discuss</a> his <a href="https://www.jonmsterling.com/01JR/">Research Group Manual</a> which codifies many of the reasons why I started a blogging tradition here in my own group.</p>
<h2><a href="https://anil.recoil.org/news.xml#ocaml" class="anchor" aria-hidden="true"></a>OCaml</h2>
<p>In the land of OCaml, I proposed <a href="https://discuss.ocaml.org/t/proposal-make-the-minimum-tested-opam-2-1-and-higher/17736">dropping the minimum supported version of opam</a> to general support. The general drag of maintenance of the CI infrastructure in my group is becoming a problem; just <a href="https://www.tunbury.org/2026/01/12/opam-25/">moving to opam 2.5</a> or <a href="https://www.tunbury.org/2026/01/16/arm64-workers/">debugging arm64 issues</a> was a giant amount of work for Mark, so we have to keep on top of deprecating old things somehow. On the other hand, we're also extending some <a href="https://www.tunbury.org/2026/01/26/ocurrent-rpc/">cool uses of Capnp capabilities</a> throughout the infrastructure, which makes CLI usage of all these services easier and easier.</p>
<p>I also took the opportunity to upload <a href="https://watch.ocaml.org/c/ocaml2025/videos">all of the OCaml workshop talks</a> to &lt;watch.ocaml.org&gt;, so they're available for your browsing pleasure.</p>
<p>I caught up with <a href="https://github.com/https://roscidus.com">Thomas Leonard</a> and <a href="https://patrick.sirref.org">Patrick Ferris</a> to discuss what to do about the number of Eio issues piling up. We're all generally happy with how <a href="https://patrick.sirref.org/fellowship-roundup/index.xml">stable and usable</a> it is (I'm using it everywhere), but we'll get together after our <a href="https://roscidus.com/blog/blog/2025/11/16/libdrm-ocaml/">current set</a> of projects to do a collective push to merge our branches together. I'm pretty happy with this model of development: get some experience using it, and then make a bunch of changes after "learning by doing".</p>
<h2><a href="https://anil.recoil.org/news.xml#fosdem" class="anchor" aria-hidden="true"></a>FOSDEM</h2>
<p>Congratulations also to <a href="https://ryan.freumh.org">Ryan Gibb</a> for a tremendous showing at FOSDEM, delivering three talks to packed rooms! They're all <a href="https://watch.eeg.cl.cam.ac.uk/c/fosdem/videos">online to watch</a> and I want to particularly highlight how much I enjoyed his package management calculus one. We're working on getting this submitted to a PL conference next!</p>
<p></p><div class="video-center"><iframe title="Package managers à la carte: A Formal Model of Dependency Resolution" width="100%" height="315px" src="https://watch.eeg.cl.cam.ac.uk/videos/embed/905d3833-a890-4ece-8ba2-cf6dbf5e2dcb" frameborder="0" allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups allow-forms"></iframe></div><p></p>
<p>The live streaming support from FOSDEM was fantastic this year and I got to see everything live while also sitting in a chilly picnic in Cambridge over the weekend.</p>
<h2><a href="https://anil.recoil.org/news.xml#next-week" class="anchor" aria-hidden="true"></a>Next week</h2>
<p><img src="https://anil.recoil.org/images/running-into-anna.webp" alt="%rc" title="Anna Lapwood in the Mill!">
It was also delightful to run into Anna Lapwood in the Mill and catch up; first time I've seen her since she left Pembroke to travel the world!</p>
<p>Some fun links:</p>
<ul>
<li><a href="https://nick.recoil.org/articles/blender-falling-leaves-simulation/">Simulating falling autumn leaves in Blender</a> is a lovely post by <a href="https://nick.recoil.org">Nick Ludlam</a> that I want to follow through.</li>
<li><a href="https://digitalflapjack.com/weeknotes/performance_and_stl_files/">Ray Tracer Performance improvements</a> by <a href="https://mynameismwd.org">Michael Dales</a> continues his inevitable journey to OxCaml as he builds his OCaml raytracer.</li>
<li><a href="https://x.com/avsm/status/2016425983843189071">Tom Blomfield notes how funny it is that Cambridge handwritten exams are now the best way to assess</a>.</li>
</ul><h1>References</h1><ul><li>Madhavapeddy (2026). My (very) fast zero-allocation webserver using OxCaml. <a href="https://doi.org/10.59350/9c6bz-kb659" target="_blank"><i>10.59350/9c6bz-kb659</i></a></li></ul>
