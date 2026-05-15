---
title: '.plan-26-08: At AI summit, Shriram''s PL opinions, Zarr hacking'
description: TESSERA paper accepted at CVPR 2026, went to the AI Impact Summit, OCaml
  Zarr hacking, Shriram's talk on human factors of formal methods, and discussions
  on teaching OxCaml to agents.
url: https://anil.recoil.org/notes/2026w8
date: 2026-02-22T00:00:00-00:00
preview_image: https://anil.recoil.org/images/week26n8-3.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>Most of the week was taken up by hopping over to New Delhi to host a <a href="https://anil.recoil.org/notes/first-tessera-hackathon">TESSERA hackathon</a> and also to <a href="https://anil.recoil.org/notes/india-ai-summit">attend the AI Impact Summit</a>.  I redeyed back to host <a href="https://cs.brown.edu/~sk/">Shriram Krishnamurthi</a> in Cambridge as he does his UK tour.</p>
<h2><a href="https://anil.recoil.org/news.xml#tessera" class="anchor" aria-hidden="true"></a>TESSERA</h2>
<p>The best news of the week was that the <a href="https://anil.recoil.org/papers/2025-tessera">TESSERA paper</a> got
accepted into <a href="https://cvpr.thecvf.com/">CVPR 2026</a>, out of a whopping 16000+
(!) submissions. This has been a giant amount of work for the whole team, but
particular props to lead author and PhD student <a href="https://www.cst.cam.ac.uk/people/zf281">Frank Feng</a> who has lead the whole
effort with perseverance and a big smile the whole time!</p>
<p>It looks like CVPR is in Denver right before PLDI in Boulder (where I have an
OxCaml tutorial to help hold) so I guess a chunk of my June will be spent in
Colorado this year.</p>
<p>I also spent some time porting <a href="https://www.tunbury.org/">Mark Elvers</a> OCaml Zarr implementation over to
<a href="https://github.com/avsm/oxmono">OxCaml</a>, and also started <a href="https://github.com/ucam-eo/geotessera/pull/194">adding Zarr zone support
to geotessera</a> so we can start
converting the registry over.</p>
<h2><a href="https://anil.recoil.org/news.xml#literature-downloader" class="anchor" aria-hidden="true"></a>Literature downloader</h2>
<p><a href="https://www.lambdacambridge.com/robin-message">Robin Message</a> and <a href="https://toao.com">Sadiq Jaffer</a> have restarted the <a href="https://anil.recoil.org/projects/ce">literature downloader</a>, and Robin has
been manually classifying DOI prefixes into a two-level tree so we can easily
dispatch download logic on a per-publisher basis (we have individual <a href="https://anil.recoil.org/papers/2025-evidence-tap">agreements</a>
via the University library with many publishers).</p>
<p>I'm surprised that DOIs are not a two-level tree to start with, as now with no
central source of detailed DOI prefix metadata if a journal is sold to another
publisher (as just happened with <a href="https://jfp.mpi-sws.org/">JFP</a>), you either
have to forward a portion of your DOI space or continue to resolve old journal
article DOIs forever.</p>
<p>I also started migrating a lot of datasets over to our new Ceph cluster,
including full syncs of GBIF, OpenAlex, and Crossref. This should set us up
nicely for <a href="https://anil.recoil.org/ideas/living-iucn-redlist">Shane's dashboard</a> using locally hosted
database for fast queries.  On the queue once the storage settles is also
<a href="https://github.com/inaturalist/inaturalist-open-data">iNaturalist open data</a>, and
to mirror the TESSERA embeddings to our Ceph so that local Cambridge users
such as <a href="https://ancazugo.github.io/">Andres Zuñiga-Gonzalez</a> can access them more easily to do global analyses directly without a full
local copy.</p>
<h3><a href="https://anil.recoil.org/news.xml#figuring-out-what-a-uri-really-is" class="anchor" aria-hidden="true"></a>Figuring out what a URI really is</h3>
<p>I also had a really fun discussion with <a href="https://jonmsterling.com">Jon Sterling</a> over High Table dinner
at Pembroke about whether it was a good idea for me to get into Lean to start
to specify the semantics of URI resolution.</p>
<p>Jon published a design for <a href="https://www.forester-notes.org/JVIT">canonical URLs in Forester</a> last year,
and as I'm getting slightly obsessed with managing Atom, RSS and JSONFeeds at the moment (the <a href="https://anil.recoil.org/network">/network</a>
view above is powered by this) this seems relevant to both that and also the literature downloader.  In return for Jon's help, I will happily
<a href="https://amok.recoil.org/@jonmsterling@mathstodon.xyz/116105419508228113">code up an OCaml monorepo script</a> for him!</p>
<h2><a href="https://anil.recoil.org/news.xml#shrirams-pl-opinions" class="anchor" aria-hidden="true"></a>Shriram's PL opinions</h2>
<p><img src="https://anil.recoil.org/images/week26n8-3.webp" alt="%rc" title="Showing Shriram a thing or two about Cambridge">
Shriram passed through Cambridge on Friday on his UK lecture tour, so I leapt at the chance to host him after leaping off the redeye from India. I last <a href="https://anil.recoil.org/notes/icfp25-what-i-learnt">chatted to Shriram at ICFP</a> over the summer, and this time we got hear him speak about <a href="https://www.youtube.com/watch?v=wBRtEQ02-HI&amp;list=PL1a1q1zrmyEwpA2PvYcM1UqE18zekujW-&amp;index=1">The Human Factors of Formal Methods</a> in the <a href="https://talks.cam.ac.uk/talk/index/244831">Logic &amp; Semantics seminar</a> here in the CL.</p>
<p>The talk was fantastic and I can't recommend watching it enough; I have so many
papers to follow up on now:</p>
<ul>
<li><a href="https://doi.org/10.1037/h0048826">Perceptual learning; differentiation or enrichment</a> (1955)</li>
<li><a href="https://doi.org/10.1037/0278-7393.13.4.640">Sexing day-old chicks: A case study and expert systems analysis of a difficult perceptual-learning task</a> (1987). The twist being it involves WWII era tanks as well.</li>
<li><a href="https://doi.org/10.1037/a0025140">Practicing versus inventing with contrasting cases: The effects of telling first on learning and transfer</a> (2011)</li>
</ul>
<h3><a href="https://anil.recoil.org/news.xml#can-llms-learn-the-stroop-effect" class="anchor" aria-hidden="true"></a>Can LLMs learn the Stroop effect?</h3>
<p>Shriram used the Stroop effect in his talk, which naturally led to <a href="https://www.cl.cam.ac.uk/~nk480/">Neel Krishnaswami</a> and me wondering if <a href="https://www.crumplab.com/blog/771_GPT_Stroop/">LLMs could learn the Stroop effect too</a>! I found <a href="https://doi.org/10.1136/bmj-2024-081948">one paper</a> on this topic:</p>
<blockquote>
<p>Moreover, as in humans, age is a key determinant of cognitive decline: “older” chatbots, like older patients, tend to perform worse on the MoCA test. These findings challenge the assumption that artificial intelligence will soon replace human doctors, as the cognitive impairment evident in leading chatbots may affect their reliability in medical diagnostics and undermine patients’ confidence.
<cite>-- <a href="https://www.bmj.com/content/387/bmj-2024-081948">Age against the machine</a>, 2024</cite></p>
</blockquote>
<p>I find the analogy between human age and 'model age' a bit incongruous, since of course models dont age -- there are improved training regimes. So the basic takeaway is that human cognitive impairment is decreasing as frontier LLMs advance.</p>
<h3><a href="https://anil.recoil.org/news.xml#adversarial-experiments-to-teach-oxcaml" class="anchor" aria-hidden="true"></a>Adversarial experiments to teach OxCaml?</h3>
<p>When chatting about how to teach our agents OxCaml better, Shriram pointed me to his 2017 paper on <a href="https://cs.brown.edu/~sk/Publications/Papers/Published/pkf-teach-pl-exp-adv-think/">Teaching Programming Languages by Experimental and Adversarial Thinking</a>:</p>
<blockquote>
<p>Its essence is to view programming language learning as a natural science
activity, where students probe languages experimentally to understand both
the normal and extreme behaviors of their features. [...] The approach is
modular (with minimal dependencies), incremental (it can be introduced slowly
into existing classes), interoperable (it does not need to push out other,
existing methods), and complementary (since it introduces a new mode of
thinking).
<cite>-- <a href="https://cs.brown.edu/~sk/Publications/Papers/Published/pkf-teach-pl-exp-adv-think/">J. Pombrio et al 2017</a></cite></p>
</blockquote>
<p>There's obvious parallels here to how the OCaml to OxCaml translation process
works, whereby we typically add in mode annotations once the OCaml version is
working. The only practical twist is that shifting to OxCaml also requires
porting code to Base/Core as well, since the stdlib doesn't have mode
annotations.</p>
<h2><a href="https://anil.recoil.org/news.xml#fun-reading" class="anchor" aria-hidden="true"></a>Fun Reading</h2>
<p><img src="https://anil.recoil.org/images/week26n8-4.webp" alt="%rc" title="Ran into Neil Lawrence randomly at the AI Summit!"></p>
<ul>
<li>I discovered that <a href="https://github.com/saurabhs92/logic-and-functional-programming-iit-delhi?tab=readme-ov-file">Saurabh Sharma taught OCaml</a> as the first year course in IIT-Delhi for quite some time!</li>
<li>I enjoyed the <a href="https://music.youtube.com/podcast/iIJjl6I4GNU">Full Disclosure episode with Rutger Bregman</a> as a followup to <a href="https://anil.recoil.org/notes/hny2026">reading his book</a>.</li>
<li>Nice episode of MCJ covering <a href="https://music.youtube.com/podcast/iIJjl6I4GNU">Turning Wasted Renewable Power into AI Compute with Rune</a>. Lots of geeking about the physics of using all that power. <a href="https://dave.recoil.org">Dave Scott</a> also pointed out to me the reason we can't just build AI datacenters up north in Scotland where the renewable power is cheap and plentiful is because there's a <a href="https://www.theregister.com/2025/11/18/uk_ai_growth_zones/">requirement for a constant national electricity price</a>.</li>
<li>Welcome <a href="https://github.com/samoht">Thomas Gazagnaire</a> back to the blogosphere with a banging post about <a href="https://gazagnaire.org/blog/2026-02-19-nasa-fprime.html">porting NASA's reusable flight software framework to OCaml</a>.</li>
<li><a href="https://news.mongabay.com/2026/02/scientists-cant-agree-on-where-the-worlds-forests-are/">Scientists cant agree on where the world's forests are</a>: would be fun to cross-check the datasets mentioned here against TESSERA.</li>
</ul>
<p><img src="https://anil.recoil.org/images/week26n8-1.webp" alt="%rc" title="Most of my week was spent in Delhi traffic, here with Amanda Brock!">
Extremely random feature: I added <a href="https://amok.recoil.org/@avsm/116081078944922670">finger support</a> to my website, so you can just do <code>finger @anil.recoil.org</code> (it is installed by default on macOS) to see my latest weekly.</p>
<h2><a href="https://anil.recoil.org/news.xml#next-week" class="anchor" aria-hidden="true"></a>Next Week</h2>
<p>I need to get TESSERA Zarr in shape. This will fix so many infrastructure
issues with using the embeddings!  I'm also going to vibe code up a cool
website for the project, using the feed aggregation logic from my own website
and these <a href="https://github.com/CloudAI-X/threejs-skills">Threejs Claude skills</a>
I just stumbled across.</p>
<p>I'm also off to <a href="https://ifip-wg28.github.io/">WG2.8</a> the week after, so I need
to figure out what functional programming goodness I will present there!</p>
<p><img src="https://anil.recoil.org/images/week26n8-2.webp" alt="%c" title="Thanks Jon Sterling for letting me look around Clare College and see the restored buildings; the scaffolding just came down!"></p><h1>References</h1><ul><li>Madhavapeddy (2026). At the AI Impact Summit in Delhi: people, planet, progress. <a href="https://doi.org/10.59350/6vc5q-mbk23" target="_blank"><i>10.59350/6vc5q-mbk23</i></a></li>
<li>Feng et al (2025). TESSERA: Temporal Embeddings of Surface Spectra for Earth Representation and Analysis. arXiv. <a href="https://doi.org/10.48550/arXiv.2506.20380" target="_blank"><i>10.48550/arXiv.2506.20380</i></a></li>
<li>Madhavapeddy (2025). What I learnt at ICFP/SPLASH 2025 about OCaml, Hazel and FP. <a href="https://doi.org/10.59350/w1jvt-8qc58" target="_blank"><i>10.59350/w1jvt-8qc58</i></a></li>
<li>Jaffer et al (2025). AI-assisted Living Evidence Databases for Conservation Science. Cambridge Open Engage. <a href="https://doi.org/10.33774/coe-2025-rmsqf" target="_blank"><i>10.33774/coe-2025-rmsqf</i></a></li>
<li>Madhavapeddy (2026). Happy new year and my fave readings of the year. <a href="https://doi.org/10.59350/y9f0e-raa45" target="_blank"><i>10.59350/y9f0e-raa45</i></a></li>
<li>Madhavapeddy (2026). 1st TESSERA/CoRE hackathon at the Indian AI Summit. <a href="https://doi.org/10.59350/1na80-7ak85" target="_blank"><i>10.59350/1na80-7ak85</i></a></li>
<li>Gibson et al (1955). Perceptual learning: Differentiation or enrichment?. <a href="https://doi.org/10.1037/h0048826" target="_blank"><i>10.1037/h0048826</i></a></li>
<li>Biederman et al (1987). Sexing day-old chicks: A case study and expert systems analysis of a difficult perceptual-learning task.. <a href="https://doi.org/10.1037/0278-7393.13.4.640" target="_blank"><i>10.1037/0278-7393.13.4.640</i></a></li>
<li>Schwartz et al (2011). Practicing versus inventing with contrasting cases: The effects of telling first on learning and transfer.. <a href="https://doi.org/10.1037/a0025140" target="_blank"><i>10.1037/a0025140</i></a></li>
<li>Dayan et al (2024). Age against the machine—susceptibility of large language models to cognitive impairment: cross sectional analysis. British Medical Journal Publishing Group. <a href="https://doi.org/10.1136/bmj-2024-081948" target="_blank"><i>10.1136/bmj-2024-081948</i></a></li></ul>
