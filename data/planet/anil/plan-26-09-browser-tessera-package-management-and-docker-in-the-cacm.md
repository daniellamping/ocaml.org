---
title: '.plan-26-09: Browser TESSERA, package management and Docker in the CACM'
description: Got TESSERA working in Zarr and the browser, and a preprint of package
  management a la carte pushed out
url: https://anil.recoil.org/notes/2026w9
date: 2026-03-01T00:00:00-00:00
preview_image: https://anil.recoil.org/images/cacm-docker-cover-1.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<h2><a href="https://anil.recoil.org/news.xml#decade-of-docker-containers" class="anchor" aria-hidden="true"></a>Decade of Docker containers</h2>
<p>A busy week on the socials as the <a href="https://anil.recoil.org/papers/2026-decade-docker">paper</a> <a href="https://dave.recoil.org">Dave Scott</a>, <a href="https://github.com/justincormack">Justin Cormack</a> and I wrote
looking back at the Docker adventure made <a href="https://anil.recoil.org/notes/cacm-docker-cover">the cover of the Communications of the CACM</a>.  Lots of really positive coverage about it online,
and the print issue should be with you all this coming week!</p>
<p><a href="https://cacm.acm.org/research/a-decade-of-docker-containers/"> <img src="https://anil.recoil.org/images/cacm-docker-cover-1.webp" alt="%c" title="Cover of a Decade of Docker Containers"> </a></p>
<h2><a href="https://anil.recoil.org/news.xml#tessera-and-zarr" class="anchor" aria-hidden="true"></a>TESSERA and Zarr</h2>
<p>In TESSERA land I spent most of my week heads down on getting full Zarr streaming support. This has been extraordinarily successful; I now not only have a significant chunk of embeddings streaming over HTTP using Zarr, it also meant that I could build an entire classification and segmentation pipeline that runs entirely in my browser! I prototyped a user interface that can run <a href="https://toao.com/blog/earth-observation-budget-solar-farms-tiny-model">Sadiq's solar farm CNN</a> entirely in my browser using wasm and WebGPU. There's a nice roundup for <a href="https://medium.com/@tobias.ramalho.ferreira/zarr-in-the-browser-fast-flexible-and-surprisingly-powerful-for-big-geo-data-eeb90ddf8a3d">Zarr visualisation options</a> that greatly helped me put this together.</p>
<p><img src="https://anil.recoil.org/images/tessera-zarr-stream-1.webp" alt="%c" title="A full browser based streaming interface for TESSERA using Zarr, to be released soon!"></p>
<p>Here's a screenshot to whet your appetite; I'll write a full blog about this shortly and post a hosted URL.  The reason it's taking a bit longer is that <a href="https://www.tunbury.org/">Mark Elvers</a> and I have been busy rearranging storage in our cluster; exposing all the embeddings as Zarr means transcoding 100s of terabytes of data, which is killing our poor internal network.</p>
<p>While working on this, I also found this <a href="https://rohitbandaru.github.io/blog/JEPA-Deep-Dive/">overview of hierarchical JEPA</a> to be very good.</p>
<p>The OxCaml TESSERA pipeline continues to <a href="https://www.tunbury.org/2026/02/25/teserra-pipeline/">come online</a> and <a href="https://jon.recoil.org">Jon Ludlam</a> also got <a href="https://jon.recoil.org/blog/2026/02/weeknotes-2026-08.html#oxcaml">odoc OxCaml docs building</a> from my <a href="https://github.com/avsm/oxmono">monorepo</a> which is a HUGE help to developing all the code.</p>
<p>While preparing my <a href="https://ifip-wg28.github.io">wg2.8</a> I also needed an HTTP caching proxy that would permanently persist Zarr tiles on my laptop for demos, so I wrote a <a href="https://github.com/avsm/oxmono/tree/main/avsm/httpz-perma-proxy">HTTP perma proxy</a> over httpz/OxCaml as well which is working for me.
Also spent a bit of time reviewing <a href="https://github.com/ocaml/opam-repository/pull/29451">OCaml relocatability in opam</a> as that would be extremely useful for rapid development setup of oxmono.</p>
<p>I also researched <a href="https://github.com/huggingface/datasets/issues/4096">Huggingface support for Zarr</a> which is still not quite there but it seems to be of interest to HF.</p>
<h2><a href="https://anil.recoil.org/news.xml#package-management-a-la-carte" class="anchor" aria-hidden="true"></a>Package management a la carte</h2>
<p><a href="https://ryan.freumh.org">Ryan Gibb</a> followed up on his <a href="https://anil.recoil.org/notes/2026w7">FOSDEM talks extravaganza</a> with a preprint on the <a href="https://anil.recoil.org/papers/2026-package-calculus">package management calculus</a> we've been working on for a while. Lots of discussions and online interest in places like <a href="https://news.ycombinator.com/item?id=47136272">HN</a> and <a href="https://lobste.rs/s/fm1eln/package_managers_la_carte_formal_model">Lobsters</a>.  I thought one of the most telling things about how subtle this area is was from HN. Someone confidently pointed out that Rust has semver:</p>
<blockquote>
<p>The rust ecosystem standardised on semver. This means it is perfectly allowed to use 1.2 in place of 1.1. While you can specify upper bounds for the dependency ranges, that is extremely uncommon in practice. Instead the bounds are just "1.1 or newer semver compatible" etc.</p>
</blockquote>
<p>...but the <a href="https://news.ycombinator.com/item?id=47136272#47193369">reality</a> that Ryan points out when you look at the crates registry is:</p>
<blockquote>
<p>In https://github.com/rust-lang/crates.io-index I count just under 7000 upper bounds on dependency ranges that aren't
just semver in disguise (e.g. not "&gt;=1.0.0, &lt;2.0.0"):
$ rg --no-filename -o '"req":"[^"]<em>&lt;[^"]</em>"' . | grep -Ev '&lt; ?=? ?([0-9]+(.0){0,2}|0.[0-9]+(.0)?)"' | wc -l
6727
So it's definitely used. One person's non-breaking change is another's breaking change https://xkcd.com/1172/</p>
</blockquote>
<p>This repeats itself all over the place. Table 1 of the <a href="https://anil.recoil.org/papers/2026-package-calculus">paper</a> covers a lot of the edge cases that we had to categorise when working through all the package ecosystems.</p>
<p>There was also a <a href="https://lobste.rs/s/fm1eln/package_managers_la_carte_formal_model">brief discussion</a> on Lobsters about <a href="https://www.oilshell.org/blog/2022/03/backlog-arch.html#what-is-a-narrow-waist">narrow waist effect</a> in technology.</p>
<h2><a href="https://anil.recoil.org/news.xml#fun-links-around-the-web" class="anchor" aria-hidden="true"></a>Fun links around the web</h2>
<p>My paper of the week is <a href="https://www.usenix.org/system/files/osdi25-schuermann.pdf">"Building Bridges: Safe Interactions with Foreign Languages through Omniglot"</a> from OSDI25, which builds out a generic FFI with zero-copy. Seems very useful for OxCaml as well; I'm going to see Mae Milano at next week's WG2.8 and hopefully learn more!</p>
<p><a href="https://www.usenix.org/system/files/osdi25-schuermann.pdf"> <img src="https://anil.recoil.org/images/omniglot-fig1.webp" alt="%c" title="Building Bridges: Safe Interactions with Foreign Languages through Omniglot, OSDI 25"> </a></p>
<ul>
<li>New <a href="https://www.linkedin.com/pulse/talent-everywhere-opportunity-isnt-greater-cambridge-impact-gnvpe">Greater Cambridge Impact</a> startup in Cambridge with a nice focus on closing social inequality around here. Nearly <a href="https://www.cambridge-news.co.uk/news/local-news/cambridgeshire-areas-deprived-kids-suffering-29293363">1 in 3 children in parts of Cambridgeshire</a> live in poverty, a shocking statistic against the backdrop of University largesse. And related to the next item, <a href="https://www.theguardian.com/business/2023/nov/14/millions-of-uk-households-forced-to-unplug-fridge-to-cope-with-rising-bills">a million households don't have a fridge</a> due to energy poverty.</li>
<li>Brilliant episode of Amol Rajan's podcast on <a href="https://www.bbc.co.uk/sounds/play/curation/m001bm45/m002rrxd">the addiction to ultra processed foods</a>, which shows the other side of the coin to our own work on <a href="https://anil.recoil.org/notes/food-and-risk-to-life">food and the risk to wildlife</a> from the <a href="https://anil.recoil.org/papers/2024-food-life">LIFE metric</a>. Treating ULP as an <a href="https://www.theguardian.com/food/2023/oct/12/its-like-trying-to-quit-smoking-why-are-1-in-7-of-us-addicted-to-ultra-processed-foods">addictive substance</a> seems to be the only way out of this monocultural consumption mess we've got us into.</li>
<li>While convincing me to visit Brown (not much convincing needed to be honest), <a href="https://cs.brown.edu/~sk/">Shriram Krishnamurthi</a> pointed out that <a href="https://en.wikipedia.org/wiki/Roger_Williams">Roger Williams</a> who founded Rhode Island in 1636 was educated at... Pembroke College Cambridge!</li>
<li>Extremely cool paper upcoming at CVPR26 on <a href="https://wenjiawang0312.github.io/projects/embodmocap/">EmbodMocap: In-the-Wild 4D Human-Scene Reconstruction for Embodied Agents</a> that uses just two iPhones to do mocap reconstructions. Very relevant to our <a href="https://anil.recoil.org/ideas/digitisation-of-insects">insect reconstruction</a> project.</li>
<li>Interesting to note that Forester may also <a href="https://amok.recoil.org/@avsm/116119655840612727">benefit from odoc plugins</a> for knowledge visualisation.</li>
<li>I've started listening to back issues of <a href="https://podcasts.castplus.fm/environment-variables">Anne Currie's podcast called Environmental Variables</a></li>
<li>Been thinking about how <a href="https://nfraprado.net/post/vcard-rss-as-an-alternative-to-social-media.html">Vcard and RSS fit together for my blog</a>.</li>
<li><a href="https://loosemore.com/">Tom Loosemore</a> makes a great case for how <a href="https://loosemore.com/2026/02/25/ai-agents-will-join-up-government-before-government-does/">gov.uk should handle AI agents</a> by <em>"restricting access to only those AI Agents who sign up to a GOV.UK kitemark, with legally-mandated conditions to manage the risks above"</em>. Excellent analogy to how India <a href="https://economictimes.indiatimes.com/tech/technology/uidai-teams-up-with-google-for-display-of-authorised-aadhaar-centres-on-google-maps/articleshow/128815309.cms?from=mdr">did this while Aadhar was maturing</a> for universal ID in India.</li>
</ul>
<p>Next week I'm in Portugal at WG2.8, so less hacking than usual!</p><h1>References</h1><ul><li>Ball et al (2025). Food impacts on species extinction risks can vary by three orders of magnitude. <a href="https://doi.org/10.1038/s43016-025-01224-w" target="_blank"><i>10.1038/s43016-025-01224-w</i></a></li>
<li>Madhavapeddy et al (2026). A Decade of Docker Containers. <a href="https://doi.org/10.1145/3761803" target="_blank"><i>10.1145/3761803</i></a></li>
<li>Gibb et al (2026). Package Managers à la Carte: A Formal Model of Dependency Resolution. arXiv. <a href="https://doi.org/10.48550/arXiv.2602.18602" target="_blank"><i>10.48550/arXiv.2602.18602</i></a></li></ul>
