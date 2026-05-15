---
title: '.plan-26-14: Tracking AI screen time and escaping to pen and paper'
description: Mythos Preview and the urgent need for internet immune systems, cognitive
  DDoS and AI screen time for code, a proposal for voluntary disclosure in OCaml,
  desktop focus and printed papers, iOS misery, GeoTessera 0.8, Ceph at 1.4PB, OCaml
  CI migration, hardware perf counters for OxCaml, and the FP Launchpad launch at
  IIT Madras.
url: https://anil.recoil.org/notes/2026w14
date: 2026-04-05T00:00:00-00:00
preview_image: https://anil.recoil.org/images/26n14-ss-2.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<h2><a href="https://anil.recoil.org/news.xml#integrating-ai-screen-time-and-tracking-into-my-life" class="anchor" aria-hidden="true"></a>Integrating AI 'screen time' and tracking into my life</h2>
<p><img src="https://anil.recoil.org/images/26n14-ss-2.webp" alt="%rc" title="Top reviews for Sinamon coffee near my old haunt the Ashby at Queens!">
I've been in Belfast this week again and had a nice coffee chat with <a href="https://patrick.sirref.org">Patrick Ferris</a> about
the value of building low-level systems code by hand, since it's the discovery
of accidental/hidden interfaces and hacks that makes this whole area so much fun.
Later that day I ran across the most <a href="https://bsky.app/profile/minaskar.bsky.social/post/3mibst6ojfc2s">thoughtful post</a> on learning and AI I've
read yet:</p>
<blockquote>
<p>I'm arguing that the way we use them matters more than whether we use them,
and that the distinction between tool use and cognitive outsourcing is the
single most important line in this entire conversation, and that almost
nobody is drawing it clearly.</p>
<p>Schwartz can use Claude to write a paper because Schwartz already knows the
physics. His decades of experience are the immune system that catches
Claude's hallucinations. <strong>A first-year student using the same tool, on the
same problem, with the same supervisor giving the same feedback, produces
the same output with none of the understanding</strong>.  The paper looks identical.
The scientist doesn't.</p>
<p><cite>-- <a href="https://ergosphere.blog/posts/the-machines-are-fine/">The machines are fine. I'm worried about us</a>, Minas Karamanis, 2026</cite></p>
</blockquote>
<p>I have been feeling the effects of nonstop <a href="https://anil.recoil.org/notes/aoah-2025">agentic coding</a> in December myself! The stuff I waved through a Claude session does have some utility, but I fear the second-order effects of using it on my own cognition are far more sinister and damaging. Marvin Hagemeister called this <a href="https://marvinh.dev/blog/ddosing-the-human-brain/">DDoSing the human brain</a> as (human) thinking context is spent just trying to figure out and steer the flood of generated code rather than building steady understanding.
However, it doesn't make sense to wish coding agents away; instead I want to figure out how to firewall its use and adapt teaching techniques as quickly as possible to the new normal.</p>
<p>The first step is simply to <em>track</em> my usage, so so I put together a proposal for <a href="https://anil.recoil.org/notes/opam-ai-disclosure">AI provenance disclosure</a> for OCaml code. This is a voluntary, low-level mechanism that I think is an important discipline to adopt to give us the equivalent of "screen time" for our coding cognition. Early feedback on the <a href="https://github.com/avsm/ocaml-ai-disclosure">proposal</a> and the <a href="https://discuss.ocaml.org/t/a-proposal-for-voluntary-ai-disclosure-in-ocaml-code/17950">OCaml Discuss thread</a> has largely been around why I don't do something bigger (for all languages), but I'm very deliberately keeping it focused on the main tools I use and not broadening it so the point of being uselessly general. I'll round up the feedback later this week as more comes in after the Easter break.</p>
<h3><a href="https://anil.recoil.org/news.xml#desktop-environments-need-to-support-flow-states-not-destroy-it" class="anchor" aria-hidden="true"></a>Desktop environments need to support flow states, not destroy it</h3>
<p><img src="https://anil.recoil.org/images/26n14-ss-1.webp" alt="%rc" title="Bangor marina this week after Storm Dave whipped through NI was a peaceful place to relax!">
I've also been noticing that my desktop environments do an increasingly terrible job
of preserving my focus flow state.  What I want from a desktop environment is
to be able to take a batch of tools like my browser, my terminal, and email
client, and then <em>narrow all of them around</em> a specific task.</p>
<p>For example, if I'm doing paper reviews for a conference then I want browser
tabs around my HotCRP, a terminal to take notes in my vim, and an email client
with searches setup around that conference so I can find instructions from the
chairs or other matters. But right now every single application seems to
become general enough to the point where it's impossible to avoid a reminders
app telling me about weekly shopping or some pointless university admin.</p>
<p><img src="https://anil.recoil.org/images/pandemic-flow-1.webp" alt="%rc" title="This was my pandemic-era flow state">
As a result, when I do something that needs deep focus I <em>still</em> print them out
onto physical paper and review them the old-fashioned way with a pen. That
kind of sucks in 2026!</p>
<p>It's not all doom and gloom though, as Adrian Sampson built <a href="https://al.radbox.org/">Analog Library</a> as a distraction-free way of reading papers without the <a href="https://anil.recoil.org/notes/acm-ai-recs">AI gunk that pervades the ACM</a> nowadays. <a href="https://kagi.com">Kagi</a> is a search engine I pay for, and it supports a good degree of personalisation and filtering. For example I now use this tip from Adrian for all searches to the ACM DL to redirect to his site in the search results in instead:</p>
<blockquote>
<p>If you use Kagi as your search engine, add this rule to your redirects to
rewrite result URLs to point to Analog Library:</p>
<p><code>^https://dl.acm.org/doi/(abs/)?(.*)|https://al.radbox.org/doi/$2</code></p>
</blockquote>
<h3><a href="https://anil.recoil.org/news.xml#ios-and-how-terrible-it-is-for-elderly-relatives" class="anchor" aria-hidden="true"></a>iOS and how terrible it is for elderly relatives</h3>
<p>While I'm ranting about desktop interfaces that fight you, I've also been
struggling with the terrible iOS interface while helping my elderly
parents recently. There are lots of issues around font sizing (try increasing
the default and see how badly the UI works on an iPhone).</p>
<p>But one iOS-ism in particular took the mickey this week. In the latest iOS 26.4 the autocorrect dictionary is allegedly improved, and my parents really need this as they often text in different languages and are getting increasingly confused by the poor old autocorrect. To take advantage of the new improvements, you need <a href="https://www.cultofmac.com/how-to/reset-iphone-ipad-keyboard-dictionary">manually reset the existing dictionaries first</a>. How hard could this be, right? In order to do this, you need to terrifyingly navigate into:</p>
<ol>
<li>Settings (<em>Q: what are we setting here? But ok lets go</em>)</li>
<li>General (<em>Q: what does 'general' mean? Let's check all the more specific menus first to figure it out!</em>)</li>
<li>Transfer or Reset (<em>Q: wait, I don't want to transfer my phone to anyone!</em>)</li>
<li>Reset (<em>Q: oh jesus what's all this about preparing for a new iphone are we deleting all my data??</em>)</li>
<li>Reset Keyboard Dictionary (<em>Q: yay i avoided deleting all my data but there's no feedback this dictionary thing worked at all</em>)</li>
</ol>
<p><img src="https://anil.recoil.org/images/reset-iphone.webp" alt="%c" title="(image credit: Killian Bell / Cult of Mac)"></p>
<p>While doing this reset, with a bigger font size it's almost impossible to not accidentally hit the 'reset entire iPhone and lose all your data' button instead.
<a href="https://github.com/https://roscidus.com">Thomas Leonard</a> has been steadily <a href="https://roscidus.com/blog/blog/2026/03/28/input-devices/">building OCaml libraries for Wayland and input</a> so I'm going to attempt a switch to a Linux desktop (again) as a good excuse to use his tools! It's interesting how good a Linux desktop is at font sizing in comparison to Apple these days as well. There are many, many weird things about 2026 so far, but I never expected it to be the year of 'Linux on the desktop'.</p>
<h2><a href="https://anil.recoil.org/news.xml#tessera-hacking-on-geotessera-08-and-registries" class="anchor" aria-hidden="true"></a>TESSERA hacking on geotessera 0.8 and registries</h2>
<p>I've rounded up a bunch of feedback/PRs and released <a href="https://github.com/ucam-eo/geotessera/releases/tag/v0.8.0">GeoTessera 0.8.0</a>, which pulls together the recent work on <a href="https://anil.recoil.org/notes/tessera-zarr-v3-layout">Zarr v3 cloud-native stores</a> and the <a href="https://anil.recoil.org/notes/tessera-embeddings-convention">geo-embeddings convention</a>. <a href="https://github.com/geospatial-jeff">Jeff Albrecht</a> merged my <a href="https://github.com/geo-embeddings/embeddings-zarr-convention/pull/2">PR to geoembeddings-conventions</a> as well, so we're now compliant with other foundation models in this space.
There's also been more <a href="https://github.com/wietzesuijker/aef-multiscales">multiscales activity</a> in the community around generating overview layers for the AEF mosaic Zarr store, which handles the tricky business of downsampling int8-encoded embeddings correctly.</p>
<p>Next week, I'm looking at spatial indexing for the TESSERA tiles since we now have multiple producers generating embeddings, and so the client library needs a shared spatial index to keep everything consistent. I'll almost certainly stick to a UTM based indexing, but I did find <a href="https://a5geo.org">A5</a> to be quite cool as the first pentagonal equal-area indexing system for the globe that I've seen.</p>
<h3><a href="https://anil.recoil.org/news.xml#ceph-expansion" class="anchor" aria-hidden="true"></a>Ceph expansion</h3>
<p><a href="https://www.tunbury.org/">Mark Elvers</a> has also <a href="https://www.tunbury.org/2026/03/27/ceph-expansion/">expanded our local Ceph storage</a> which was getting
very squeezed indeed with all the <a href="https://geotessera.org/blog/2026-03-30-training-and-inference-at-scale">tile generation we've been
doing</a>. We're
up to around 1.4PB raw now in total across our various hosts (not all on Ceph
yet), and the next step is to shift my home directories onto CephFS for
networked use to see how well it works interactively. The main challenge with
doing this storage migration has been making sure that we never only have one
copy of some data while migrating a large filesystem, which is always
nerve-wracking... but we've had no disasters yet!</p>
<h3><a href="https://anil.recoil.org/news.xml#moving-ocaml-infra-from-scaleway-to-cambridge" class="anchor" aria-hidden="true"></a>Moving OCaml infra from Scaleway to Cambridge</h3>
<p>Mark and I have also spent some time refreshing the OCaml CI pipeline and completing the <a href="https://www.tunbury.org/2026/04/01/from-scaleway-to-cambridge/">migration from Scaleway to Cambridge</a>. Our Cambridge VMs run on Xen with xapi, which feels pleasingly <a href="https://anil.recoil.org/papers/2010-icfp-xen">full-circle</a>.
Relatedly, I've also proposed <a href="https://discuss.ocaml.org/t/dropping-some-intermediate-ocaml-versions-from-ci/17947">dropping some intermediate OCaml versions from CI</a> to reduce the testing matrix. This builds on the <a href="https://anil.recoil.org/notes/deprecating-ocaml-408">earlier conversation</a> about deprecating OCaml 4.08 support.</p>
<h2><a href="https://anil.recoil.org/news.xml#fun-links" class="anchor" aria-hidden="true"></a>Fun links</h2>
<h3><a href="https://anil.recoil.org/news.xml#recoil-email-self-hosting" class="anchor" aria-hidden="true"></a>Recoil email self-hosting</h3>
<p>I've been sorting out the <a href="https://anil.recoil.org/notes/decentralised-stack">recoil.org self-hosting stack</a> again during the Easter break, as complaints about spam have become a family tradition to present to me during this time. I've been evaluating using the Stalwart Mail Server, as it now has <a href="https://bulwarkmail.org/">Bulwark</a>, an open-source webmail client that covers email, calendar, contacts, and also files over JMAP. There's also Plume, a <a href="https://www.reddit.com/r/stalwartlabs/comments/1s1f5yw/plume_a_swift_native_jmap_email_client_now_in/">Swift-native JMAP email client</a> for macOS.</p>
<p>This is all making me think it's to dust off my <a href="https://anil.recoil.org/notes/aoah-2025-17">JMAP library</a> and migrate properly from Dovecot/Postfix. The only thing making me nervous is Stalwart's "all in one" approach to mail storage as it locks it all up into a database. I've been running Postfix with maildir for a very, very long time and I'm not excited about giving up the simplicity of flat files on disk that I can rsync...</p>
<h3><a href="https://anil.recoil.org/news.xml#hardware-performance-counters-for-oxcaml" class="anchor" aria-hidden="true"></a>Hardware performance counters for OxCaml</h3>
<p><a href="https://toao.com">Sadiq Jaffer</a> has been hacking on integrating <a href="https://toao.com/blog/free-performance-counters-runtime-events">hardware performance counters into OxCaml's Runtime Events</a> (<a href="https://bsky.app/profile/did:plc:3lovwu4e3pkhxffeer3prugb/post/3miovqimqzk2a">bsky</a>). His prototype uses <code>rdpmc</code> on Linux which is a tiny 20-40 cycles per sample to capture perf counters at the start and end of every runtime events span. This means you can get per-GC-phase information in quite some detail and reveals fun phase information about the effectiveness of tricks like <a href="https://github.com/ocaml/ocaml/pull/10195">prefetching</a>.</p>
<p>I'm interested in having this run on Apple Silicon too (notwithstanding my intention to switch to Linux above!). @tmcgilchrist built <a href="https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html">mperf</a> as a <code>perf</code>-like CLI for macOS. It uses Apple's <a href="https://gist.github.com/ibireme/173517c208c7dc333ba962c1f0d67d12">private kperf frameworks</a>. With Sadiq's runtime integration into we're getting close to hardware-level profiling of OCaml programs all the time in production, rather than doing special 'benchmarking runs'.</p>
<h3><a href="https://anil.recoil.org/news.xml#paper-of-the-week" class="anchor" aria-hidden="true"></a>Paper of the week</h3>
<p>I had the pleasure of finally meeting <a href="https://en.wikipedia.org/wiki/Laure_Zanna">Laure Zanna</a> this week to co-organise something together. Her paper on "<a href="https://www.nature.com/articles/s41586-020-2591-3">The causes of sea-level rise since 1900</a>" is therefore my paper of the week as I learn more about weather forecasting and sea ice predictions to connect <a href="https://anil.recoil.org/projects/rsn">our remote sensing work</a> to this important part of climate change and biodiversity research.</p>
<p>The paper tries to define a "budget" for global sea-level rise so that the sum of all contributing processes (ice-mass loss, thermal expansion, reservoir impoundment) reconcile with observational data. Ice-mass loss from glaciers has caused twice as much sea-level rise as thermal expansion since 1900. I'm very curious if this could be added to TESSERA <a href="https://anil.recoil.org/papers/2025-tessera-tasks">downstream tasks</a> as we hit almost a decade of embeddings now.</p>
<h3><a href="https://anil.recoil.org/news.xml#coming-up" class="anchor" aria-hidden="true"></a>Coming up</h3>
<p>I'm back to Cambridge next week for a few days, and then I'm off to IIT Madras to see my friend <a href="https://kcsrk.info">KC Sivaramakrishnan</a> and attend the launch of the <strong><a href="https://fplaunchpad.org/2026/03/30/fp-launchpad-kickoff.html">FP Launchpad</a></strong>. It's a cracking lineup of speakers who I am really excited about seeing after a few years, and of course do some tourism in Chennai (where I used to live in the early 90s and go to school in [PSBB](noticeable mismatch with the AOHs and range maps due to the different projections)!</p><h1>References</h1><ul><li>Madhavapeddy (2025). Are you still using OCaml 4.08 or earlier? If so, we need to know. <a href="https://doi.org/10.59350/1jxq1-7e147" target="_blank"><i>10.59350/1jxq1-7e147</i></a></li>
<li>Madhavapeddy (2026). A Proposal for Voluntary AI Disclosure in OCaml Code. <a href="https://doi.org/10.59350/cxypn-ysv27" target="_blank"><i>10.59350/cxypn-ysv27</i></a></li>
<li>Madhavapeddy (2025). Dear ACM, you're doing AI wrong but you can still get it right. <a href="https://doi.org/10.59350/c84g4-5zt58" target="_blank"><i>10.59350/c84g4-5zt58</i></a></li>
<li>Feng et al (2026). Applications of the TESSERA Geospatial Foundation Model to Diverse Environmental Mapping Tasks. SSRN. <a href="https://doi.org/10.2139/ssrn.6142416" target="_blank"><i>10.2139/ssrn.6142416</i></a></li>
<li>Scott et al (2010). Using functional programming within an industrial product group: perspectives and perceptions. ACM. <a href="https://doi.org/10.1145/1863543.1863557" target="_blank"><i>10.1145/1863543.1863557</i></a></li>
<li>Madhavapeddy (2026). TESSERA now supports the Zarr geo-embeddings convention proposal. <a href="https://doi.org/10.59350/c3hrq-zsx02" target="_blank"><i>10.59350/c3hrq-zsx02</i></a></li>
<li>Madhavapeddy (2026). Streaming millions of TESSERA tiles over HTTP with Zarr v3. <a href="https://doi.org/10.59350/tk0er-ycs46" target="_blank"><i>10.59350/tk0er-ycs46</i></a></li></ul>
