---
title: '.plan-26-10: Streaming TESSERA working, biodiversity action papers, and FPL
  takes off'
description: TESSERA streaming in the browser, planetary programming at WG2.8, biodiversity
  action papers, FP Launchpad opens, and Docker CACM buzz
url: https://anil.recoil.org/notes/2026w10
date: 2026-03-08T00:00:00-00:00
preview_image: https://anil.recoil.org/images/videos/08aafc87-9aea-48e3-8c41-a2fe1b94fea4.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<h2><a href="https://anil.recoil.org/news.xml#tessera-streaming-into-the-browser-and-oxcaml-hacking" class="anchor" aria-hidden="true"></a>TESSERA streaming into the browser and OxCaml hacking</h2>
<p>I've completed a working cut at a streaming interface for TESSERA embeddings, and it's unexpectedly addictive to zoom around the world staring at false colours!  The amazing thing about this interface is that it's <em>entirely</em> browser based, using JavaScript, WebGPU and WASM to perform all the analysis on the client side. The Zarr embeddings are chunked and served over HTTP, using range requests to retrieve the minimum amount of data.</p>
<p></p><div class="video-center"><iframe title="Tessera Zarr streaming preview" width="100%" height="315px" src="https://crank.recoil.org/videos/embed/08aafc87-9aea-48e3-8c41-a2fe1b94fea4" frameborder="0" allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups allow-forms"></iframe></div><p></p>
<p>The video shows classification workflows, but I've also got <a href="https://toao.com/blog/can-we-really-see-brambles-from-space">solar panel detection</a> working using Sadiq's patch based embeddings. You can try it for yourself when I release this properly next week!</p>
<p>There are obvious limitations to how much we can do in the browser; for most serious work we will need a server running, but my goal here is to see if we can embed TESSERA into the <a href="https://anil.recoil.org/ideas/living-iucn-redlist">living dashboard</a> that <a href="https://shaneweisz.com">Shane Weisz</a> is working on. I've also just received a drop of <a href="https://digitalflapjack.com/weeknotes/fractional_life_progress/">areas-of-habitats</a> from <a href="https://mynameismwd.org">Michael Dales</a> which I'll have a go at integrating next week.</p>
<h3><a href="https://anil.recoil.org/news.xml#its-tee-time-with-multiple-browsers-in-development" class="anchor" aria-hidden="true"></a>It's TEE time with multiple browsers in development</h3>
<p>What's blocking everyone from using Zarr and TESSERA? Well, we need to transcode a petabyte of embeddings from the old numpy format into Zarr, which is a difficult parallelisation problem. I'm making steady progress on an <a href="https://oxcaml.org">OxCaml</a> pipeline for this with a from-scratch OxCaml-Zarr implementation that <a href="https://www.tunbury.org/">Mark Elvers</a> helped me kick off.  Mark also published his <a href="https://github.com/mtelvers/ocaml-tessera">ocaml-tessera</a> pipeline which I'm going to import to OxCaml next week as well, so that we can do both model training and tile inference in OCaml!</p>
<p>For more production-oriented usecases, <a href="https://svr-sk818-web.cl.cam.ac.uk/keshav/wiki/index.php/Main_Page">Srinivasan Keshav</a> has published a server with his excellent <a href="https://tee.cl.cam.ac.uk">Tessera Embeddings Explorer</a>. We've been building our implementations independently and swapping ideas for user interfaces and analyses, which has been a very productive way of experimenting for different users. His implementation is in use for several downstream tasks projects and should be what people use, while mine is heading towards more dynamic mobile/browser workflows. You can grab that code at <a href="https://github.com/ucam-eo/tee">https://github.com/ucam-eo/tee</a>, complete with convenient Dockerfile.</p>
<h3><a href="https://anil.recoil.org/news.xml#discussing-tessera-programming-models-at-wg28" class="anchor" aria-hidden="true"></a>Discussing TESSERA programming models at WG2.8</h3>
<p><img src="https://anil.recoil.org/images/wg28-26-2.webp" alt="%rc" title="Viana de Castelo had the loveliest venue and hotel this week">
I got invited to <a href="https://ifip-wg28.github.io/">WG2.8</a> again, so <a href="https://simon.peytonjones.org/">Simon Peyton Jones</a> and I trooped off there from Cambridge earlier in the week. I had to leave early due to some family matters, but I got to present 'planetary programming' to the assembled gods of functional programming!</p>
<p>I began my <a href="https://www.cl.cam.ac.uk/~avsm2/slides/wg28-2026-tessera.pdf">talk</a> by presenting the streaming browser demo above (which, thanks to the <a href="https://github.com/avsm/oxmono/blob/main/avsm/httpz-perma-proxy">perma caching httpz oxcaml proxy</a> I hacked up lets the remote tiles be cached on my laptop so the app works offline too). I used the opportunity to posit a high-level programming design problem I've encountered when coding with TESSERA...</p>
<p>There's a contradictory tension in the styles of programming that we
conventionally embark on with the workflows needed by machine learning. In our
<a href="https://anil.recoil.org/papers/2025-fairground">FAIRground</a> paper we describe a purely functional Python
variant that represents conventional 'forward programming'. You can do lots of
nice things when the language is pure, such as enabling incremental live
computations. This forward programming style would be good for building a global computational wiki for example.</p>
<p>However, when we program with observational embeddings like TESSERA, we're
doing 'backwards programming'. The units we're dealing with are 128-dimensional
self-supervised representations that have been learnt from primary satellite
data, and the job of the program is to help cluster these higher dimensional
structures into some semblance of useful meaning. We do this via downstream
classifiers, segmenters and regression tasks. This is a very different
programming style from forward programming even though it requires a similar
amount of CPU.</p>
<p><img src="https://anil.recoil.org/images/wg28-26-3.webp" alt="%rc" title="There was no danger of losing weight in Portugal">
The ultimate goal of <em>both</em> of these scientific programming styles is to establish <em>causal</em> relationships; that is, we want to form (or reinforce or falsify) a theory of how the world works that can be tested by the scientific method. So I wondered: how do we combine these three styles into a programming language? This is a very general question, but I figured there was no better place to ask than a room full of people who have designed dozens if not hundreds of languages between them.</p>
<p><a href="https://www.cl.cam.ac.uk/~avsm2/slides/wg28-2026-tessera.pdf"> <img src="https://anil.recoil.org/images/fwd-back-causal-ss-1.webp" alt="%c" title="Three sorts of relations we are trying to program"> </a></p>
<ul>
<li><a href="http://www.ccs.neu.edu/home/amal/">Amal Ahmed</a> said this really looked like a <a href="https://doi.org/10.1145/3609027.3609405">multi-DSL problem</a> (i.e. implement all three different styles of programming in OCaml, and then examine the DSL properties/data structures). She also pointed me to <a href="https://arxiv.org/abs/2502.19538">Multi-Language Probabilistic Programming</a> which allows for differently specialised probabilistic programming languages.</li>
<li><a href="https://homepages.inf.ed.ac.uk/slindley/">Sam Lindley</a> also suggested this multi-DSL would be a good use of effects: could we write <em>one</em> OCaml program to represent all three styles, but then interpret them completely differently using effects? We could get a set of points as program traces via sampling using effects, and then we could do reproducible simulations via effects for stochastic choice, and then for causal path tests effects that check program data structure invariants regularly to build up causal hypotheses.  I need to talk to <a href="https://kcsrk.info">KC Sivaramakrishnan</a> and <a href="https://patrick.sirref.org">Patrick Ferris</a> about this more, as with any effects based idea that involves more than just a Suspend effect.</li>
<li><a href="https://www.chalmers.se/en/persons/ms/">Mary Sheeran</a> observed that this is somewhat like hardware programming, whereby we put the minimal structural constraints in and then try to discover layouts.</li>
<li><a href="https://www.chalmers.se/en/persons/rjmh/">John Hughes</a> noted that the statistical testing combined with datastructures (the synthetic computation) is quite similar to quickcheck: can we posit causal relationships and 'quickcheck' them efficiently?</li>
<li><a href="https://www.uu.nl/staff/GKKeller">Gabriele Keller</a> is working with spatial ecologists on saltmarshes using array programming to speed up their calculations, so we had a <em>very</em> productive conversation that I will follow up on! Sounds remarkably similar to the work in the Cairngorms that <a href="https://coomeslab.org">David Coomes</a> is leading in the <a href="https://www.clr.conservation.cam.ac.uk/">CLR</a>.</li>
<li><a href="https://justtesting.org/">Manuel Chakravarthy</a> gave me a lot of tips on Mac/iOS approaches and also told me about <a href="https://volteuropa.org/">Volt Europa</a> and their push for liberal sovereignty.</li>
<li><a href="https://www.cs.cornell.edu/~jnfoster/">Nate Foster</a> and <a href="https://homepages.inf.ed.ac.uk/slindley/">Sam Lindley</a> helped me simplify my thinking a lot: rather than worry about scale (millions of species), can we find the smallest possible example to work outwards from synthetically? I obviously thought of <a href="https://anil.recoil.org/ideas/hedgehog-mapping">hedgehog mapping</a> as good one here. We also thought that viewing causality as a 'triangle' wasn't right: instead, we could use a combination of synthetic models + observational samples as bidirectional lenses, and then draw causal path diagrams across them to test the lenses (sort of like natural experiments). This is somewhat like <a href="https://doi.org/10.1145/1328897.1328487">boomerang lenses</a> but for sample data instead of strings.</li>
<li><a href="https://richarde.dev/">Richard Eisenberg</a> gave me practical OxCaml advice as it's a fast-moving target: layout polymorphism is a while away yet, so keep using <code>ppx_template</code> for now, but other features like float16 (useful for TESSERA) could be done fairly easily.</li>
<li><a href="https://people.mpi-sws.org/~rossberg/">Andreas Rossberg</a> was impressed by the use of WASM for browser-based geospatial, and we discussed the difficulty of using wasm with the DOM for interactive interfaces. Machine learning workflows perform well because of the lack of DOM transitions, but hopefully <a href="https://hacks.mozilla.org/2026/02/making-webassembly-a-first-class-language-on-the-web/">Mozilla is working on improving this</a>.</li>
<li><a href="https://simon.peytonjones.org/">Simon Peyton Jones</a> looked bemused by it all and thought it was too high level a concept to latch onto. I need a worked example like the above to convince him when I'm back at Cambridge!</li>
</ul>
<p>It was a short trip to Portugal in the end, but massively energising. I do love hanging with functional programmers!</p>
<h2><a href="https://anil.recoil.org/news.xml#biodiversity-action-through-technology" class="anchor" aria-hidden="true"></a>Biodiversity action through technology</h2>
<p>Two big perspective papers on global biodiversity are out in PNAS this week, which I <a href="https://anil.recoil.org/notes/nas-rs-biodiversity-papers">wrote up separately</a> in a detailed note.
To follow up on these, <a href="https://web.eecs.umich.edu/~comar/">Cyrus Omar</a> is chairing this year's <a href="https://propl.dev">PROPL</a> (to be held at PLDI this summer), and we've been discussing doing something different this year to tie into a more 'action-oriented' workshop that combines the learnings from the last couple of years with the call to biodiversity action above.</p>
<p>Meanwhile, to followup on the <a href="https://anil.recoil.org/notes/first-tessera-hackathon">first TESSERA hackathon</a> over in India a few weeks ago, <a href="https://www.cse.iitd.ac.in/~aseth/">Aadi Seth</a> and <a href="https://svr-sk818-web.cl.cam.ac.uk/keshav/wiki/index.php/Main_Page">Srinivasan Keshav</a> have put together a <a href="https://www.linkedin.com/posts/core-stack_first-round-of-innovation-challenge-advances-activity-7436361754617552897-tyrD">call for students</a> to get involved. If you're interested then <a href="https://docs.google.com/document/d/1vcYj6D_ReWE5xG51A7-Gdt2g6pqpZfw8/edit">apply here</a> and get going with TESSERA!</p>
<p><img src="https://anil.recoil.org/images/iitm-campus-1.webp" alt="%rc" title="Life on the IIT-Madras campus is all about adorable dogs and deer roaming around">
And not to be left behind by their Delhi counterparts, <a href="https://kcsrk.info">KC Sivaramakrishnan</a> announced that applications <a href="https://fplaunchpad.org/2026/03/06/applications-open-post-bacc-fellowship.html">are now open</a> for the <a href="https://fplaunchpad.org">FP Launchpad</a> in IIT-Madras.
This should be of interest to computer scientists who want to get involved in environmental work; as I <a href="https://anil.recoil.org/notes/india-ai-summit">mentioned</a> before, one of the illustrative projects to kick off the FPL is programming TESSERA embeddings more ergonomically:</p>
<blockquote>
<p>A programmable public infrastructure for environmental planning, combining
TESSERA's satellite-derived representations with CoRE Stack data and
compositional functional models in O(x)Caml to support auditable indicators
and scenario analysis for India’s water and habitat systems.
<cite>-- <a href="https://fplaunchpad.org/charter/">FP Launchpad Charter, 2026</a> </cite></p>
</blockquote>
<p>So it's action stations at both the IITs and I'm looking forward to working with them from Cambridge! This is a nice followup to our <a href="https://www.cam.ac.uk/news/new-boost-for-historic-relationship-between-university-of-cambridge-and-india-announced">Cambridge VC visiting India</a> and kicking off a cricketing tour!</p>
<h2><a href="https://anil.recoil.org/news.xml#docker-buzz-from-the-cacm-article" class="anchor" aria-hidden="true"></a>Docker buzz from the CACM article</h2>
<p>Following the <a href="https://anil.recoil.org/notes/cacm-docker-cover">CACM Docker article</a>, there's been lots of positive online discussions about the article. <a href="https://news.ycombinator.com/item?id=47289311">Hackernews</a> leads the way with typically split opinions. Some loved it, some hated it, some thought it should be replaced with a very small shell script, and others reminisced over our use of <a href="https://en.wikipedia.org/wiki/Slirp">SLIRP</a>. Overall though a lovely discussion and vibe.</p>
<p><a href="https://news.ycombinator.com/item?id=47289311"> <img src="https://anil.recoil.org/images/hn-docker-ss-1.webp" alt="%c" title="Docker on top of HN again"> </a></p>
<p>I also read two interesting papers while <a href="https://anil.recoil.org/notes/2026w9">researching</a> more background for the <a href="https://anil.recoil.org/papers/2026-package-calculus">package calculus</a> that <a href="https://ryan.freumh.org">Ryan Gibb</a> has been working on:</p>
<ul>
<li><a href="https://arxiv.org/abs/2601.12811">Docker Does Not Guarantee Reproducibility</a> discusses some of the common pitfalls around building fully reproducible containers. While there's support at the lower levels for this in the Docker stack, I agree it's not kept up with modern needs. Luckily, <a href="https://patrick.sirref.org">Patrick Ferris</a> is hacking on a new <a href="https://patrick.sirref.org/weekly-2025-w49/">shell interface with provenance</a>.</li>
<li><a href="https://arxiv.org/abs/2501.15919v1">Does Functional Package Management Enable Reproducible Builds at Scale? Yes.</a>: This is a complementary paper, and argues for a Nix-like approach. It's good to see that Nix (despite its constrained versioning) does a good job of supporting retrospective builds.</li>
</ul>
<h2><a href="https://anil.recoil.org/news.xml#visitor-from-kth" class="anchor" aria-hidden="true"></a>Visitor from KTH</h2>
<p><img src="https://anil.recoil.org/images/wg28-26-1.webp" alt="%rc" title="The Mill does the best fish and chips in Cambridge I reckon!">
We had a delightful visit from <a href="https://www.kth.se/profile/yifang">Professor Yifang Ban</a> from KTH, who delivered this week's <a href="https://watch.eeg.cl.cam.ac.uk/w/dZNDoKiuH8sugfKLCWh8gS">EEG seminar</a> on EO-AI4GlobalChange. We went to the pub after, and discussed a rather staggering number of <a href="https://anil.recoil.org/papers/2025-tessera-tasks">downstream tasks</a> that Prof Ban works on:</p>
<blockquote>
<p>In this seminar, Professor Ban will discuss recent research at the
intersection of EO and AI, with a focus on deep learning methods for
monitoring environmental change at scale. She will present selected results
from EO-AI4GlobalChange, a collaborative research project developing novel,
globally-applicable deep learning approaches for analysing multi-sensor,
multi-modal EO data. The talk will cover examples including 2D and 3D urban
mapping, urban change detection, wildfire detection and near-real-time
monitoring, flood mapping, and multi-hazard building damage detection.</p>
<p>The seminar will also briefly introduce PANGAEA, a global benchmark for
Geospatial Foundation Models, and discuss insights from the systematic
evaluation of widely used foundation models across multiple geospatial
domains. Finally, Professor Ban will briefly outline the objectives of the
recently established AI4EO Working Group within Group on Earth Observations
(GEO), which aims to advance GEO’s vision of Earth Intelligence for All
through AI-driven Earth observation research, innovation, and collaboration.
<cite>-- <a href="https://watch.eeg.cl.cam.ac.uk/w/dZNDoKiuH8sugfKLCWh8gS">Yifang Ban, EEG Seminar</a>, March 2026</cite></p>
</blockquote>
<p>But most importantly, we had excellent fish and chips to celebrate her <a href="https://www.linkedin.com/feed/update/urn:li:activity:7434213135705694210/">first visit</a> to Cambridge!</p>
<h2><a href="https://anil.recoil.org/news.xml#fun-links" class="anchor" aria-hidden="true"></a>Fun links</h2>
<ul>
<li>Next OxCaml <a href="https://anil.recoil.org/notes/aoah-2025-13">vibespiling</a> target: "<a href="https://iev.ee/blog/resharp-how-we-built-the-fastest-regex-in-fsharp/">How we built the fastest regexp engine in F#</a>" with code <a href="https://github.com/ieviev/resharp-dotnet">here</a>.</li>
<li>The calls to <a href="https://anil.recoil.org/notes/rs-future-of-publishing">reform publishing</a> are getting <a href="https://www.experimental-history.com/p/the-one-science-reform-we-can-all">louder and louder</a>. A new ATProto service called <a href="https://chive.leaflet.pub/3mgb6k5pwsc2q">Chive</a> looks interesting here.</li>
<li>New podcast on sci-fi is a lot of fun, called <a href="https://starshipalexandria.com/">Starship Alexandria</a> with Emma Newman and Adrian Tchaikovsky. I've been <a href="https://www.jonmsterling.com/2026-W10/">reminded</a> to pick up Adrian's latest series <a href="https://www.goodreads.com/book/show/60147395-city-of-last-chances">City of Last Chances</a> which I'm enjoying so far. Great insect world building as always!</li>
<li>Extremely sad news is the passing of Prof <a href="https://royalsociety.org/people/alan-wilson-10879/">Alan Wilson</a>, who I was showing my <a href="https://www.flickr.com/photos/avsm/albums/72177720328187549/with/54709177736">Botswana leopard pictures</a> and getting flying tips from just a few months ago. He passed away in a light aircraft crash while heading into the sand dunes of Namibia. Very, very sad news.</li>
<li>More bad news of (a pretty bad) week is that <a href="https://doi.org/10.21203/rs.3.rs-6079807/v1">global warming has accelerated significantly</a>.</li>
<li><strong>But the good news</strong> is that I learnt of <a href="https://en.wikipedia.org/wiki/Lazarus_taxon">lazarus taxon</a> that come back from extinction, such as this week's <a href="https://www.theguardian.com/environment/2026/mar/05/marsupials-discovered-new-guinea">adorable marsupial thought extinct for 6000 years</a>. Hurrah!</li>
</ul><h1>References</h1><ul><li>Madhavapeddy (2026). Connecting the dots for biodiversity action from the NAS/Royal Society Forum. <a href="https://doi.org/10.59350/dy7d3-hdt43" target="_blank"><i>10.59350/dy7d3-hdt43</i></a></li>
<li>Madhavapeddy (2026). At the AI Impact Summit in Delhi: people, planet, progress. <a href="https://doi.org/10.59350/6vc5q-mbk23" target="_blank"><i>10.59350/6vc5q-mbk23</i></a></li>
<li>Feng et al (2026). Applications of the TESSERA Geospatial Foundation Model to Diverse Environmental Mapping Tasks. SSRN. <a href="https://doi.org/10.2139/ssrn.6142416" target="_blank"><i>10.2139/ssrn.6142416</i></a></li>
<li>Madhavapeddy (2025). Royal Society's Future of Scientific Publishing meeting. <a href="https://doi.org/10.59350/nmcab-py710" target="_blank"><i>10.59350/nmcab-py710</i></a></li>
<li>Madhavapeddy (2026). 1st TESSERA/CoRE hackathon at the Indian AI Summit. <a href="https://doi.org/10.59350/1na80-7ak85" target="_blank"><i>10.59350/1na80-7ak85</i></a></li>
<li>Omar et al (2025). A FAIR Case for a Live Computational Commons. Association for Computing Machinery. <a href="https://doi.org/10.1145/3759536.3763802" target="_blank"><i>10.1145/3759536.3763802</i></a></li>
<li>Gibb et al (2026). Package Managers à la Carte: A Formal Model of Dependency Resolution. arXiv. <a href="https://doi.org/10.48550/arXiv.2602.18602" target="_blank"><i>10.48550/arXiv.2602.18602</i></a></li>
<li>Patterson et al (2023). Semantic Encapsulation using Linking Types. ACM. <a href="https://doi.org/10.1145/3609027.3609405" target="_blank"><i>10.1145/3609027.3609405</i></a></li>
<li>10.1145/1328897.1328487<a href="https://doi.org/10.1145/1328897.1328487" target="_blank"><i>10.1145/1328897.1328487</i></a></li>
<li>Rahmstorf et al (2025). Global Warming has Accelerated Significantly. Research Square. <a href="https://doi.org/10.21203/rs.3.rs-6079807/v1" target="_blank"><i>10.21203/rs.3.rs-6079807/v1</i></a></li>
<li>Stites et al (2025). Multi-Language Probabilistic Programming. arXiv. <a href="https://doi.org/10.48550/arXiv.2502.19538" target="_blank"><i>10.48550/arXiv.2502.19538</i></a></li></ul>
