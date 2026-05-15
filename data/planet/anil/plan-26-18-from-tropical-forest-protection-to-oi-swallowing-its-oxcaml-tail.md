---
title: '.plan-26-18: From tropical forest protection to oi swallowing its oxcaml tail'
description: Our REDD+ over-crediting paper hits Nature Communications just as Microsoft
  retreats from removals, we talk responsible evidence synthesis while LLMs appear
  in UK planning, and oi grows a self-update bootstrap.
url: https://anil.recoil.org/notes/2026w18
date: 2026-05-03T00:00:00-00:00
preview_image: https://anil.recoil.org/images/cyrus-visit-1.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<h2><a href="https://anil.recoil.org/news.xml#redd-over-crediting-in-nature-communications" class="anchor" aria-hidden="true"></a>REDD+ over-crediting in Nature Communications</h2>
<p>Our <a href="https://anil.recoil.org/papers/2025-redd-evals">paper</a> on learning lessons from over-crediting in
REDD+ projects came out this week in Nature Communications, led by
<a href="https://www.conservation.cam.ac.uk/directory/dr-tom-swinfield">Thomas Swinfield</a>. The reception to the paper has generally been encouragingly
positive, especially with the framing that "bad credits are not the same as bad
projects".</p>
<p>The timing of the paper hits the carbon market at a particular low time, since it's reeling from <a href="https://www.nytimes.com/2026/04/16/climate/microsoft-carbon-removal.html">Microsoft's abrupt retreat from carbon removals</a> meaning that the single largest buyer of removals is scaling back its <a href="https://news.microsoft.com/source/features/sustainability/from-farms-to-oceans-how-microsoft-is-working-to-scale-carbon-dioxide-removal/">commitments</a> from just a few months ago. While this hits the whole stack of direct-air-capture pipelines through to nature-based projects, at least it can hopefully rebuild now with nature and technological removals/avoidance working in harmony rather than bickering about which specific method is 'the best'.</p>
<p><a href="https://www.nature.com/articles/s41467-026-71552-3"> <img src="https://anil.recoil.org/papers/2025-redd-evals" alt="%c"> </a></p>
<p>We need them all, and I think the necessary response should be to raise the floor on both at once, and not try to pick a winner. The <a href="https://www.cam.ac.uk/research/news/carbon-credits-have-enabled-vital-protection-of-tropical-forests-despite-being-oversold-tenfold">Cambridge University</a> and <a href="https://www.cst.cam.ac.uk/news/carbon-credits-have-enabled-vital-protection-tropical-forests-despite-being-oversold">Computer Lab</a> writers have done a great job carrying that point through, and I've written up my own <a href="https://anil.recoil.org/notes/redd-overcrediting">full argument</a> for those who want to dig in.</p>
<h2><a href="https://anil.recoil.org/news.xml#the-inevitable-rise-of-llms-in-government-decision-making" class="anchor" aria-hidden="true"></a>The inevitable rise of LLMs in government decision-making</h2>
<p><a href="https://toao.com">Sadiq Jaffer</a> and <a href="https://samreynolds.org">Sam Reynolds</a> also gave a great talk at the <a href="https://digitalstatecraft.academy/">Digital Statecraft Academy</a> on our <a href="https://anil.recoil.org/projects/ce">evidence synthesis work</a>, on <em>"The inevitable rise of Large Language Models in government decision making"</em>. Civil servants and policy folk in the room were asking practical questions about how to do the right thing. There were questions about best practices for <a href="https://anil.recoil.org/papers/2024-ce-llm">reducing hallucinations</a> with responses discussing retrieval grounding, structured outputs, human-in-the-loop checkpoints, and maintaining proper evaluation harnesses.</p>
<p><img src="https://anil.recoil.org/images/digital-statecraft-2.webp" alt="%c" title="The inevitable rise of Large Language Models in government decision making (Sadiq and Sam)"></p>
<p>Then there were concerns about the computational and power crunch to keep all
of this affordable as adoption scales across government. We discussed the use
of smaller specialised models, on-prem inference for sensitive workloads, and
the open question of whether the UK has the data-centre capacity to host
serious sovereign deployments. The third was on whether quantum computing
changes the picture (quick answer: no).</p>
<p>Just as this was all happening, the government announced Google had won a
tender for <a href="https://www.ft.com/content/91ce4475-d325-4d65-babb-4214996bc0f6?syn-25a6b1a6=1">planning-decision automation</a>.
English councils are trialling a Google AI tool to speed up planning, which is
precisely the kind of black-box deployment my <a href="https://anil.recoil.org/notes/red-pill-conservation">red-pill/blue-pill argument</a>
was cautioning against. Decisions affecting people's homes are now being filtered
through opaque models with no public scrutiny of the reasoning chain.</p>
<p><img src="https://anil.recoil.org/images/digital-statecraft-1.webp" alt="%rc" title="Sam and Sadiq in full flow">
Sadiq and I took a closer look at the <a href="https://www.find-tender.service.gov.uk/Notice/040491-2026?origin=SearchResults&amp;p=1">tender
notice</a>,
and spotted that bidders were required to integrate with the incumbent planning
systems, which effectively freezes out smaller UK players. Last year about this
time, I was looking at how the UK might benefit from an open-data substrate via
a <a href="https://anil.recoil.org/notes/uk-national-data-lib">national data library</a>. Without a credible open
layer, every public-sector AI tender will keep collapsing onto the same handful
of incumbent vendors.</p>
<p>If there's a bright spot, it's that the questions from the Statecraft audience
suggest civil servants increasingly understand this, and the government itself
is <a href="https://ukdefencejournal.org.uk/uk-firms-win-89-percent-of-defence-contracts-since-july-2024/">dispatching more contracts to UK firms</a> in
other sectors. We'll get there with the open AI story...</p>
<h2><a href="https://anil.recoil.org/news.xml#hacking-updates" class="anchor" aria-hidden="true"></a>Hacking updates</h2>
<p><img src="https://anil.recoil.org/images/cyrus-visit-1.webp" alt="%c"></p>
<h3><a href="https://anil.recoil.org/news.xml#cyrus-visits-to-talk-hazel" class="anchor" aria-hidden="true"></a>Cyrus visits to talk Hazel</h3>
<p>The week started with a fun visit from <a href="https://kcsrk.info">KC Sivaramakrishnan</a> (over for a <a href="https://kcsrk.info/verification/rdts/lean/2026/04/28/from-convergence-to-confidence/">PaPOC keynote</a>), <a href="https://web.eecs.umich.edu/~comar/">Cyrus Omar</a>, Andrew Blinn and Matthew
Keenan (formerly an undergrad here at Cambridge, now doing a PhD over with
Cyrus in UMich).</p>
<p>We all sat down with <a href="https://ryan.freumh.org">Ryan Gibb</a> to brainstorm over ideas for how
to combine recent advances in <a href="https://hazel.org">Hazel</a> with the
<a href="https://anil.recoil.org/projects/oxcaml">OxCaml</a> work going on around here.</p>
<p><img src="https://anil.recoil.org/images/cyrus-visit-2.webp" alt="%rc">
The two most exciting things were the emergence of a Hazel CLI (so we can
not only integrate it more easily into MDX workflows but also agentic coding),
and also Ryan's <a href="https://anil.recoil.org/papers/2026-package-calculus">package calculus</a> as the basis for a
brand new approach to how we express dependencies (in a language that doesn't
have any backwards compatibility baggage to worry about). More on this as
we convene next at <a href="https://pldi26.sigplan.org/home/propl-2026">PROPL3</a> in June!</p>
<h2><a href="https://anil.recoil.org/news.xml#oi-gains-self-update" class="anchor" aria-hidden="true"></a>oi gains self-update</h2>
<p>I've been making steady progress on <a href="https://github.com/avsm/oi"><code>oi</code></a>, the
uv-style binary distributor I started working on a <a href="https://anil.recoil.org/notes/2026w16">few weeks ago</a>,
and have been <a href="https://anil.recoil.org/notes/2026w17">dogfooding</a> it with a <a href="https://jon.recoil.org/blog/2026/04/weeknotes-2026-16-17.html">few</a> <a href="https://github.com/RyanGibb/eon/commit/572538424b21a7c124baf3d5127f2d47b35ce20d">others</a>. <a href="https://www.tunbury.org/">Mark Elvers</a> and I have been <a href="https://www.tunbury.org/2026/04/30/day10-oxcaml/">cross checking</a>
that our tools are compatible while <a href="https://www.tunbury.org/2026/04/29/ocaml-ci-update/">doing OCaml maintainance</a>.</p>
<p>The big new feature this week is <code>oi self update</code>. Having distributed binaries via oi for a few weeks, the next obvious
step was for <code>oi</code> itself to become one of the binaries it updates. This makes
pushing fixes much less painful, and brings it closer to self-hosting itself.</p>
<p>The two features I really want <code>oi</code> to nail at this point are:</p>
<ul>
<li>how to <em>quickly</em> run a binary in the uv style where you don't install anything. I've added <code>oix</code> for this now, so that <code>oix utop</code> just works — backed by source tracking and a local cache.</li>
<li><em>easy</em> static-binary builds so you can ship a single binary that runs anywhere without thinking about which libc/arch the target is on. <code>oi</code> handles this by shelling out to Docker for the static-build pipeline. I'm still working on wiring the binary builds through so that it just works (and I need to investigate fat binaries to see if they're worth it, but I'm guessing not).</li>
<li>updates as a <em>library</em>, so that binaries can <a href="https://anil.recoil.org/notes/aoah-2025-24">evolve</a></li>
</ul>
<p>I've also been hacking with <a href="https://github.com/samoht">Thomas Gazagnaire</a> to merge his <a href="https://tangled.org/gazagnaire.org/monopampam">monopampam</a> tree back into the <a href="https://anil.recoil.org/notes/aoah-2025">agentic-libraries</a> trees from last year. Thomas has been doing an enormous amount of new coding <a href="https://gazagnaire.org/blog/2026-04-15-ccsds-protocol-stack.html">for space protocols</a> and has built a lovely CCSDS protocol stack. I've merged almost all of his changes back into my OCaml trees, and will look at OxCaml merges next. More on this mega monorepo as it stabilises in the next few weeks!</p>
<h2><a href="https://anil.recoil.org/news.xml#fun-links" class="anchor" aria-hidden="true"></a>Fun Links</h2>
<ul>
<li>From the <a href="https://slate.com/podcasts/political-gabfest">Slate Political Gabfest</a>, I learnt that <a href="https://time.com/article/2026/04/28/people-say-fewer-words-per-day/">the number of words we are speaking per day</a> is dramatically shrinking. Read the <a href="https://doi.org/10.1177/17456916261425131">full study</a>.</li>
<li>As Github <a href="https://oppi.li/posts/we_need_a_federation_of_forges/">crumbles</a> the Tangled team <a href="https://blog.tangled.org/vouching/">shipped</a> the reputation system I've always wanted since <a href="https://web.archive.org/web/20170715120119/http://advogato.org/person/Stab">Advogato</a>. Check out my <a href="https://pdsls.dev/at://did:plc:nhyitepp3u4u6fcfboegzcjw/sh.tangled.graph.vouch">vouch ATProto records</a> here!</li>
</ul><h1>References</h1><ul><li>Madhavapeddy (2026). Discussing effective conservation with all the UK Chief Scientists. <a href="https://doi.org/10.59350/qjrmv-38130" target="_blank"><i>10.59350/qjrmv-38130</i></a></li>
<li>Madhavapeddy (2025). Thoughts on the National Data Library and private research data. <a href="https://doi.org/10.59350/fk6vy-5q841" target="_blank"><i>10.59350/fk6vy-5q841</i></a></li>
<li>Swinfield et al (2026). Learning lessons from over-crediting to ensure additionality in forest carbon credits. Nature Publishing Group. <a href="https://doi.org/10.1038/s41467-026-71552-3" target="_blank"><i>10.1038/s41467-026-71552-3</i></a></li>
<li>Iyer et al (2025). Careful design of Large Language Model pipelines enables expert-level retrieval of evidence-based information from syntheses and databases. <a href="https://doi.org/10.1371/journal.pone.0323563" target="_blank"><i>10.1371/journal.pone.0323563</i></a></li>
<li>Gibb et al (2026). Package Managers à la Carte: A Formal Model of Dependency Resolution. arXiv. <a href="https://doi.org/10.48550/arXiv.2602.18602" target="_blank"><i>10.48550/arXiv.2602.18602</i></a></li>
<li>Pfeifer et al (2026). Sliding Into Silence? We Are Speaking 300 Daily Words Fewer Every Year. <a href="https://doi.org/10.1177/17456916261425131" target="_blank"><i>10.1177/17456916261425131</i></a></li></ul>
