---
title: '.plan-26-15: Banyan trees, (anti)botnets and Bose-Einstein bases'
description: Travelling from Ireland to IIT Madras for the FP Launchpad launch, mirroring
  half a petabyte of TESSERA embeddings to AWS Open Data, antibotty discussions, and
  Tangled trust boundaries for AI code review.
url: https://anil.recoil.org/notes/2026w15
date: 2026-04-12T00:00:00-00:00
preview_image: https://anil.recoil.org/images/iitmcamp-2026-1.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>I spent this week shuttling from Ireland to Cambridge to India for the <a href="https://fplaunchpad.org/2026/03/30/fp-launchpad-kickoff.html">FP Launchpad</a> takeoff at IIT Madras next week. This new centre is possibly even more exciting than a <a href="https://www.bbc.co.uk/news/articles/cr51z54d5rpo">roundtrip to the moon</a> for the PL geeks among you!</p>
<p><img src="https://anil.recoil.org/images/iitmcamp-2026-1.webp" alt="%c" title="Certainly the nicest management school campus I've ever been to"></p>
<p>I'm hosted by <a href="https://kcsrk.info">KC Sivaramakrishnan</a> and staying in the amazing Bose-Einstein guest house on campus. There are banyan trees, monkeys, huge bats and deer everywhere! Ilya Sergey and I wandered up to the local student coffee shop and ran into <a href="https://durwasa-chakraborty.github.io/">Durwasa Chakraborty</a> and <a href="https://orcid.org/0000-0001-9987-7491">Vimala Soundarapandian</a> where we learnt more about PhD life out here. Durwasa has a fun blog post on <a href="https://durwasa-chakraborty.github.io/posts/how-agentic-llms-almost-destroyed-my-academic-career/">"How Agentic LLMs Almost Destroyed My Academic Career"</a> which came up as we chatted about how easy access was to frontier coding LLMs in India (answer: not very) and also <a href="https://anil.recoil.org/notes/opam-ai-disclosure">my recent musings</a> about AI provenance.</p>
<p><img src="https://anil.recoil.org/images/iitmcamp-2026-2.webp" alt="%rc" title="There is some monkeying around going on, but not from the students who were all working very hard!">
This is my second trip to India this year after the <a href="https://anil.recoil.org/notes/india-ai-summit">AI Impact Summit</a> and the <a href="https://anil.recoil.org/notes/first-tessera-hackathon">TESSERA hackathon</a> in Delhi, but my first time back to Tamil Nadu in a while. I went to school here in <a href="https://www.psbbschools.ac.in/">PSBB</a> back in 1990 and it's fantastic touring my old haunts again!</p>
<h2><a href="https://anil.recoil.org/news.xml#antibotties-and-mythos-fallout" class="anchor" aria-hidden="true"></a>Antibotties and Mythos fallout</h2>
<p>My <a href="https://anil.recoil.org/notes/internet-immune-system">antibotty immune system post</a> got some <a href="https://lobste.rs/s/aw2jr4/assessing_claude_mythos_preview_s">discussion on Lobsters</a> after the launch of Mythos. I thought that <a href="https://lcamtuf.coredump.cx/">lcamtuf</a> (of afl-fuzz fame) offered the most nuanced take:</p>
<blockquote>
<p>I'm less convinced that this is a fundamental inflection point for software
security. The bulk of vulnerability research - I'd not hesitate to say 80%+ -
was already automated with fuzzers and related tools. Fuzzers discovered tens
of thousands of bugs in critical software and put some visible strain on the
OSS ecosystem. We now have a new tool that will uncover even more bugs, but I
don't think this is how the world ends. When I came up with afl-fuzz, I
could've put out a press release saying that it's too dangerous to share. We
survived. So, there's a combination of impressive results and marketing at
play.</p>
<p>As a tangent, I worry a lot more about enterprise security. We now have a
tool that can be deployed on the cheap to pull on every door handle at any
large organization; solving this attack surface is less tractable than
finding software bugs, in part because, to stick to my tortured metaphor, the
"handles" the bots can pull on include every human working at the company.
Worse, with vuln discovery, there's this inherent symmetry: you can use the
same tools to beat the bad guys to the punch. Using LLMs to secure an
enterprise is considerably harder, in part because you need much higher
decision fidelity to automatically stop bad things without getting in the way
of normal work.
<cite>-- <a href="https://lobste.rs/s/aw2jr4/assessing_claude_mythos_preview_s#c_8qj9op">lcamtuf, Lobste.rs</a>, 2026</cite></p>
</blockquote>
<p>In connected news, I've also been triaging bugs in various projects that have
come out of Mythos-related disclosures, but more on that in a few weeks when
it's public...</p>
<h3><a href="https://anil.recoil.org/news.xml#fp-launchpad-preview" class="anchor" aria-hidden="true"></a>FP Launchpad preview</h3>
<p>The <a href="https://fplaunchpad.org/2026/03/30/fp-launchpad-kickoff.html">FP Launchpad kickoff</a> is on Monday and headed by <a href="https://kcsrk.info">KC Sivaramakrishnan</a>, with a goal of bringing more assurance to systems construction. The speaking lineup is cracking: we've got Rishiyur Nikhil on <a href="https://fplaunchpad.org/2026/04/09/rishiyur-nikhil-property-based-testing-pipelined-cpu.html">FP for hardware design</a> at Bluespec (which our Cambridge CHERI CPU uses), Chester Rebeiro on trusted hardware on RISC-V Shakti, Krishnan Raghavan on verifiable governance with LLMs and Lean, Shriram Krishnamurthi on lightweight diagramming languages, Ilya Sergey on mechanizing a massive real world type checker with AI-assisted metatheory, and I'll be talking about <a href="https://anil.recoil.org/projects/tessera">TESSERA</a> and FP.</p>
<p>Others have been passing through Chennai as well; I missed out on last week when <a href="https://fplaunchpad.org/2026/04/03/nik-swamy-agentic-proof-oriented-programming.html">Nik Swamy's talk on agentic proof-oriented programming</a> happened. He's produced 200KLOC+ of verified F*/Pulse code in a few weeks using Copilot CLI (10x harder than a good coding CLI, if I'm being snarky), including specifications of concurrent data structures through to proofs of production C code. <a href="https://kcsrk.info">KC Sivaramakrishnan</a> <a href="https://anil.recoil.org/notes/internet-immune-system">conjectured</a> to me during the week that the same LLM coding agents driving remote vulnerability discovery could also drive proof generation. Nik's results are the first evidence I've seen that this might actually scale, and Ilya has also been speeding along with his Lean code on this topic.</p>
<h2><a href="https://anil.recoil.org/news.xml#tessera-half-a-petabyte-to-the-cloud" class="anchor" aria-hidden="true"></a>TESSERA: half a petabyte to the cloud</h2>
<p>This week in TESSERA was mainly spent rationalising the storage setup. We've now got access to <a href="https://aws.amazon.com/opendata/">AWS Open Data</a> (yay!) to mirror the TESSERA embeddings just as we run out of space in the University cluster again, and so we've started a giant transfer of everything to the cloud. This will take around 7 days (half a petabyte is a fair bit of data!), but I can satisfyingly watch this stream using S3:</p>
<pre><code>&gt; s3cmd ls s3://tessera-embeddings/v1/global_0.1_degree_representation/
    DIR  s3://tessera-embeddings/v1/global_0.1_degree_representation/2017/
    DIR  s3://tessera-embeddings/v1/global_0.1_degree_representation/2018/
    DIR  s3://tessera-embeddings/v1/global_0.1_degree_representation/2019/
    DIR  s3://tessera-embeddings/v1/global_0.1_degree_representation/2020/
    DIR  s3://tessera-embeddings/v1/global_0.1_degree_representation/2024/
    DIR  s3://tessera-embeddings/v1/global_0.1_degree_representation/2025/
</code></pre>
<p>Last week's <a href="https://anil.recoil.org/notes/2026w14">Ceph expansion</a> to 1.4PB raw was well timed as without that headroom the Cambridge side of this transfer would have been pretty squeezed, but we're all good now. Since we're also short of GPUs for inference as always, <a href="https://www.tunbury.org/">Mark Elvers</a> has also been <a href="https://www.tunbury.org/2026/04/08/intel-amx/">investigating using Intel AMX for inference</a> which turns out to be much cheaper on the cloud spot market.</p>
<p>Next week I'm adding geotessera support for multiple registries so clients can
discover tiles from both Cambridge and AWS. This builds on the <a href="https://anil.recoil.org/notes/tessera-zarr-v3-layout">Zarr v3 restructuring</a> and <a href="https://anil.recoil.org/notes/tessera-embeddings-convention">geo-embeddings convention</a> work from the past few weeks. Once
the sync finishes, I'll kick off the Zarr conversions directly on AWS as well
to actually make this usable with <a href="https://anil.recoil.org/notes/2026w14">geotessera 0.8</a>.</p>
<h2><a href="https://anil.recoil.org/news.xml#fun-links" class="anchor" aria-hidden="true"></a>Fun links</h2>
<h3><a href="https://anil.recoil.org/news.xml#tangled-trust-boundaries-for-ai-code-review" class="anchor" aria-hidden="true"></a>Tangled trust boundaries for AI code review</h3>
<p>I've been thinking about reputation systems again and how to manage our <a href="https://anil.recoil.org/notes/tangled-and-ci">Tangled</a> PDSes, which have the nice property that our own <a href="https://anil.recoil.org/notes/disentangling-git-with-bluesky">git repos are hosted by us</a> with the social metadata living elsewhere on ATProto. I'd like to extend our group's use of Tangled for better tracking of what's AI code or not, following my <a href="https://anil.recoil.org/notes/opam-ai-disclosure">OCaml AI Disclosure proposal</a>.</p>
<p>What I want are trust boundaries (within our group, or between individuals) where we can trade AI-generated code and help each other review it iteratively while remaining informed about where the code came from. I want to avoid what Avery Pennarun calls <a href="https://apenwarr.ca/log/20260316">the sloth of too many review layers</a> (every layer of approval makes you 10x slower in wall-clock time) by engineering quality <em>in</em>, taking advantage of OCaml's module interfaces to enforce boundaries. This connects to the <a href="https://anil.recoil.org/notes/principles-for-collective-knowledge">provenance</a> thinking I've been doing of late: if you can see at a glance what's AI-generated and who has reviewed what, we can direct limited human attention (and energy) much more efficiently without burning out.</p>
<p>This <a href="https://meri.leaflet.pub/3mj4qwvypq22a">post by Meri</a> also evaluates ATProto's proposed approach to permissioned data and figures out ownership transferability. What happens when a personal project (e.g. anil.recoil.org) grows into a community resource and the original creator wants to hand off control (say to eeg.cl.cam.ac.uk)? Lots of pesky details to get right here.</p>
<h3><a href="https://anil.recoil.org/news.xml#chinese-open-source-and-india" class="anchor" aria-hidden="true"></a>Chinese open source and India</h3>
<p>Kevin Xu's <a href="https://interconnect.substack.com/p/chinese-open-source-a-definitive">definitive history of Chinese open source</a> is a good read.</p>
<blockquote>
<p>DeepSeek changed everything. Along with Qwen, Kimi, GLM, MiniMax, Stepfun,
and even Unitree, Chinese open weight AI models took the world by storm. The
traction even earned “open source” a shoutout in the Chinese Premier’s
all-important address at the Two Sessions, with an explicit commitment to
continue the growth of open source communities and ecosystems as part of the
country’s next five-year plan.
<cite>-- <a href="https://interconnect.substack.com/p/chinese-open-source-a-definitive">Kevin Xu</a>, 2026</cite></p>
</blockquote>
<p>I'd love to see someone write an equivalent history for Indian open source;
given my <a href="https://anil.recoil.org/notes/india-ai-summit">increasing time here</a> it feels like there's a
story to tell here as well, even though there hasn't been (to my knowledge) a breakout
project from here that's had impact as big as Deepseek in recent years.</p>
<h3><a href="https://anil.recoil.org/news.xml#computings-environmental-responsibility" class="anchor" aria-hidden="true"></a>Computing's environmental responsibility</h3>
<p>I'm going to be proposing a Dagstuhl seminar with Manuel Rigger and Sara Beery and ran across this <a href="https://drops.dagstuhl.de/storage/03dagstuhl-manifestos/volume011/issue01/DagMan.11.1.1/DagMan.11.1.1.pdf">previous Dagstuhl manifesto on "What is Computing's Responsibility?"</a> from a 2025 workshop. It argues that computing contributes to planetary boundary overshoot rather than solving it, as ICT accounts for ~3% of global CO2 (comparable to aviation) and energy demand is accelerating. They do invoke Jevons' Paradox (that efficiency gains from computing lead to rebound effects) rather than net reductions.</p>
<p>I'm not sure I entirely agree with their conclusions though; our <a href="https://anil.recoil.org/projects/plancomp">planetary computing</a> work and the <a href="https://anil.recoil.org/papers/2024-planetary-computing">arxiv paper</a> argue for pragmatic data-driven environmental policy-making which doesn't strike me as digital exceptionalism. The Dagstuhl summary is right that computing does have an overall responsibility problem, but I don't think the answer is to retreat from computational approaches to environmental activism. The upcoming third <a href="https://anil.recoil.org/notes/icfp25-propl">'action PROPL'</a> at PLDI in Boulder and the <a href="https://anil.recoil.org/notes/nas-rs-biodiversity-papers">Rewilding the Web</a> workshop in Edinburgh next month are both attempts at this.</p><h1>References</h1><ul><li>Madhavapeddy (2026). Connecting the dots for biodiversity action from the NAS/Royal Society Forum. <a href="https://doi.org/10.59350/dy7d3-hdt43" target="_blank"><i>10.59350/dy7d3-hdt43</i></a></li>
<li>Madhavapeddy (2026). A Proposal for Voluntary AI Disclosure in OCaml Code. <a href="https://doi.org/10.59350/cxypn-ysv27" target="_blank"><i>10.59350/cxypn-ysv27</i></a></li>
<li>Madhavapeddy (2026). At the AI Impact Summit in Delhi: people, planet, progress. <a href="https://doi.org/10.59350/6vc5q-mbk23" target="_blank"><i>10.59350/6vc5q-mbk23</i></a></li>
<li>Madhavapeddy (2026). TESSERA now supports the Zarr geo-embeddings convention proposal. <a href="https://doi.org/10.59350/c3hrq-zsx02" target="_blank"><i>10.59350/c3hrq-zsx02</i></a></li>
<li>Madhavapeddy (2026). Streaming millions of TESSERA tiles over HTTP with Zarr v3. <a href="https://doi.org/10.59350/tk0er-ycs46" target="_blank"><i>10.59350/tk0er-ycs46</i></a></li>
<li>Madhavapeddy (2025). Socially self-hosting source code with Tangled on Bluesky. <a href="https://doi.org/10.59350/r80vb-7b441" target="_blank"><i>10.59350/r80vb-7b441</i></a></li>
<li>Madhavapeddy (2025). Programming for the Planet at ICFP/SPLASH 2025. <a href="https://doi.org/10.59350/hasmq-vj807" target="_blank"><i>10.59350/hasmq-vj807</i></a></li>
<li>Madhavapeddy (2026). 1st TESSERA/CoRE hackathon at the Indian AI Summit. <a href="https://doi.org/10.59350/1na80-7ak85" target="_blank"><i>10.59350/1na80-7ak85</i></a></li>
<li>Madhavapeddy (2025). Four Ps for Building Massive Collective Knowledge Systems. <a href="https://doi.org/10.59350/418q4-gng78" target="_blank"><i>10.59350/418q4-gng78</i></a></li>
<li>Ferris et al (2024). Planetary computing for data-driven environmental policy-making. arXiv. <a href="https://doi.org/10.48550/arXiv.2303.04501" target="_blank"><i>10.48550/arXiv.2303.04501</i></a></li>
<li>Madhavapeddy (2026). The Internet needs an antibotty immune system, stat. <a href="https://doi.org/10.59350/snnnf-asc02" target="_blank"><i>10.59350/snnnf-asc02</i></a></li>
<li>Madhavapeddy (2025). mlgpx is the first Tangled-hosted package available on opam. <a href="https://doi.org/10.59350/7267y-nj702" target="_blank"><i>10.59350/7267y-nj702</i></a></li></ul>
