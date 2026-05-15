---
title: A Decade of Docker Containers on the CACM cover!
description: Our CACM cover article reflects on a decade of Docker, from the early
  days of hacking Docker for Mac on a French farm to today's AI-driven sandboxing,
  covering the technical origins, cross-platform challenges, and the vibrant open-source
  community that made it all possible.
url: https://anil.recoil.org/notes/cacm-docker-cover
date: 2026-02-24T00:00:00-00:00
preview_image: https://anil.recoil.org/images/cacm-docker-cover-1.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>I am <em>beyond</em> excited to be on the cover of the <a href="https://cacm.org">CACM</a> March issue with "<strong><a href="https://cacm.acm.org/research/a-decade-of-docker-containers/">A Decade of Docker Containers</a></strong>", coauthored with <a href="https://dave.recoil.org">Dave Scott</a> and <a href="https://github.com/justincormack">Justin Cormack</a>:</p>
<p><a href="https://cacm.acm.org/research/a-decade-of-docker-containers/"> <img src="https://anil.recoil.org/images/cacm-docker-cover-1.webp" alt="%rc" title="Cover of the CACM March 2026!"> </a></p>
<blockquote>
<p>For the past decade, Docker has provided a robust solution for building,
shipping, and sharing applications. But behind its simple "build and run"
workflow lie many years of complex technical challenges.
<cite>-- <a href="https://cacm.acm.org/research/a-decade-of-docker-containers/">A Decade of Docker Containers</a>, Communications of the ACM, Mar 26</cite></p>
</blockquote>
<p>Docker was such a whirlwind ride that we never got to write any academic papers about some of the technical systems magic that went into it. Today's article, along with the <a href="https://anil.recoil.org/papers/2025-docker-icfp">ICFP experience report</a> from last year form a companion pair to delve into the tricks required to scale the system to millions of daily users.</p>
<p>We cover the technical origins in Linux, the library VMM layers needed to hide Linux on macOS and Windows. And then we discuss where Docker is going next, with the giant AI coding wave making it incredibly important to sandbox agents running pretty much everywhere now.</p>
<div class="video-center" style="padding:56.25% 0 0 0;position:relative;"><iframe src="https://player.vimeo.com/video/1166690675?badge=0&amp;autopause=0&amp;player_id=0&amp;app_id=58479" frameborder="0" allow="autoplay; fullscreen; picture-in-picture; clipboard-write; encrypted-media; web-share" referrerpolicy="strict-origin-when-cross-origin" style="position:absolute;top:0;left:0;width:100%;height:100%;" title="A Decade of Docker Containers"></iframe></div>
<p>The video accompanying the article was <a href="https://anil.recoil.org/notes/2026w6">recorded</a> in my office by the wonderful <a href="https://www.rosiepowellfreelance.com/">Rosie Powell</a>, with thanks to Pembroke College. And the pixel cover art of container ships that the CACM commissioned is fantastic!</p>
<h2><a href="https://anil.recoil.org/news.xml#getting-involved-in-docker" class="anchor" aria-hidden="true"></a>Getting involved in Docker</h2>
<p><img src="https://anil.recoil.org/images/cacm-docker-cover-5.webp" alt="%rc" title="The first time I booted up an embedded VM and got a console!">
Firstly, a huge thank you to Solomon Hykes, the project founder and the person who invited us <a href="https://anil.recoil.org/notes/docker-buys-unikernel-systems">to join forces</a> in the early days of Docker. We all holed up in a French farm and hacked like mad (<a href="https://x.com/solomonstre/status/1584963582235906049">photo from Solomon</a>) and came up with the first iteration of Docker for Desktop in a few days!</p>
<p>We didn't actually realise that's what we'd <a href="https://web.archive.org/web/20160504110338/https://blog.docker.com/2016/03/docker-for-mac-windows-beta/">call it back then</a>.  The project was originally codenamed Pinata and even had a <a href="https://forums.docker.com/t/pinata-missing-in-latest-mac-beta-1-11-2-beta15/15541">CLI tool</a> of the same name for quite a while! In order to get a feel for whether or not it would be popular, we took a leaf out Gmail's launch and send out limited invite codes. There was nothing to have been worried about as it took off fast (except the traditional <a href="https://news.ycombinator.com/item?id=11352389">HN disdain</a>) with <a href="https://medium.com/@nzoschke/docker-for-mac-beta-review-b91692289eb5">positive reviews</a>.</p>
<blockquote>
<p>Docker For Mac is a game changer. I’ve been able to cope with the previous tools but the experience has been rough to say the least.
<cite>-- <a href="https://medium.com/@nzoschke/docker-for-mac-beta-review-b91692289eb5">Docker For Mac Beta Review</a>, Noah Zoschke, Apr 2016</cite></p>
</blockquote>
<p>After the desktop beta came out in 2016, we also <a href="https://www.docker.com/blog/docker-unikernels-open-source/">open sourced quite a few components</a>, some of which are now features <a href="https://anil.recoil.org/notes/apple-containerisation">implemented</a> into macOS and Windows. Some <a href="https://anil.recoil.org/papers/2025-docker-icfp">tricks like VPNKit</a> are now <a href="https://github.com/containers/gvisor-tap-vsock">adopted widely</a> in other ecosystems, which is nice to see.</p>
<h2><a href="https://anil.recoil.org/news.xml#docker-is-defined-by-its-incredible-community" class="anchor" aria-hidden="true"></a>Docker is defined by its incredible community</h2>
<p></p><div class="video-vertical"><iframe title="Solomon Hykes unveiling the Docker Desktop Pinãta" src="https://crank.recoil.org/videos/embed/2c165f30-ea51-4b6e-893d-9273aba630be" frameborder="0" allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups allow-forms" style="aspect-ratio: 9/16; width: 100%; height: 100%; max-width: 325px;"></iframe></div>
While our article covers the technical aspects of Docker, we don't comment enough on how <em>fun</em> the community is! (See the massive <a href="https://cacm.acm.org/research/a-decade-of-docker-containers/#:~:text=from%20external%20networks.-,Acknowledgments,-We%20are%20grateful">acknowledgements</a> section in the article for just a small sample of the key contributors).<p></p>
<p>Container management and cloud computing are obviously worth vast amounts of money now, but the giant whale and plush toys and crazy antics at Dockercons are what I'll remember most fondly.  Throughout all the ups and downs, Docker's been (I strongly feel) a strong force for openness in preventing any single entity capturing the full workflow of how we manage software, and therefore contributing to building a vibrant and diverse ecosystem.</p>
<p>Today, it's still entirely possible for a small player to quite simply spin up their own selfhosted infrastructure and interoperate with the behemoths. That's important; heck I use <a href="https://docs.docker.com/engine/swarm/">swarm mode</a> on my own <a href="https://anil.recoil.org/news.xml">##selfhosting</a> to this day!</p>
<p>We're seeing a big change in open source community building happening this year. The vibe coding onslaught is calling into question how we'll make open source friends in the future, and it looks like we're falling back to <a href="https://mitchellh.com/writing/my-ai-adoption-journey">reputation networks for contributors</a>. I hope that we see more Docker-style communities spring up than boring corporate driven artificial ecosystems!</p>
<h2><a href="https://anil.recoil.org/news.xml#the-futures-bright-for-containerisation" class="anchor" aria-hidden="true"></a>The future's bright for containerisation</h2>
<p><img src="https://anil.recoil.org/images/cacm-docker-cover-3.webp" alt="%rc" title="Stealing a camel and joining forces with a whale at OSCon">
The other great thing to see in recent years is the new generation of maintainers hacking on adjacent technologies among my colleagues and students. <a href="https://www.tunbury.org/">Mark Elvers</a> is advancing <a href="https://www.tunbury.org/2026/02/19/obuilder-hcs/">Windows support with HCS</a>, <a href="https://ryan.freumh.org">Ryan Gibb</a> just uploaded his latest work on <a href="https://anil.recoil.org/papers/2026-package-calculus">formalising dependency management</a> and <a href="https://patrick.sirref.org">Patrick Ferris</a> has been hacking on <a href="https://patrick.sirref.org/merry/index.xml">shell integrated provenance</a>. And "old" maintainers like <a href="https://github.com/samoht">Thomas Gazagnaire</a> who I cofounded UnikS with are <a href="https://gazagnaire.org/blog/2026-02-23-asplos-unikernels.html">taking Docker and OCaml into space</a>!</p>
<p><img src="https://anil.recoil.org/images/cacm-docker-cover-2.webp" alt="%rc" title="At Pembroke College with Solomon">
Combining these advances with <a href="https://anil.recoil.org/papers/2025-ocaml-ai">agentic coding</a> results in radically different coding methodologies, but using the same lower level interfaces that Docker's built on today. Evolution is happening fast, and more accessible than ever thanks to Docker's open source roots.</p>
<p>Here's to the coming century of containerisation; enjoy <a href="https://cacm.acm.org/research/a-decade-of-docker-containers/">reading the article</a> and do let me know if you have any comments or queries!</p>
<p><img src="https://anil.recoil.org/images/cacm-docker-cover-4.webp" alt="%c" title="Remembering Gordon the turtle, sadly passed on now"></p><h1>References</h1><ul><li>Madhavapeddy et al (2026). A Decade of Docker Containers. <a href="https://doi.org/10.1145/3761803" target="_blank"><i>10.1145/3761803</i></a></li>
<li>Madhavapeddy et al (2025). Functional Networking for Millions of Docker Desktops. <a href="https://doi.org/10.1145/3747525" target="_blank"><i>10.1145/3747525</i></a></li>
<li>Madhavapeddy (2025). Under the hood with Apple's new Containerization framework. <a href="https://doi.org/10.59350/70ynk-ves20" target="_blank"><i>10.59350/70ynk-ves20</i></a></li>
<li>Gibb et al (2026). Package Managers à la Carte: A Formal Model of Dependency Resolution. arXiv. <a href="https://doi.org/10.48550/arXiv.2602.18602" target="_blank"><i>10.48550/arXiv.2602.18602</i></a></li></ul>
