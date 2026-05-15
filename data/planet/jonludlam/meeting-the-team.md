---
title: Meeting the Team
description:
url: https://jon.recoil.org/blog/2025/04/meeting-the-team.html
date: 2025-04-08T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#meeting-the-team" class="anchor"></a>Meeting the Team</h1>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2025-04-08</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/meta.html" title="meta">meta</a></p></li></ul>
<p>It's tremendously exciting to be back in the <a href="https://www.cst.cam.ac.uk/">Computer Laboratory</a>, as the last time I worked here was just before the pandemic. I'm now a member of the <a href="https://www.cst.cam.ac.uk/research/eeg">Energy and Environment Group</a> whose goal is "to have a measurable impact on tools and techniques for de-risking the future".</p>
<h2><a href="https://jon.recoil.org/atom.xml#what's-going-on?" class="anchor"></a>What's going on?</h2>
<p>With such a broad goal, it's hard to know where to start and how I'll fit in, so my first few weeks have been spent getting to know the other members of the group and what they're up to. It's an incredibly inspiring group of individuals who are all doing amazing work, and it's really humbling and daunting to be a part of it.</p>
<p>There's some really interesting work going on in our group on LLMs, principally led by the fantastic <a href="https://toao.com/">Sadiq Jaffer</a>. We had a chat a few weeks ago and have started to explore some ideas around seeing how well LLMs can program in OCaml already before we start to do some RL training on them. Having not done any LLM stuff before, it's a steep learning curve for me, but we're already seeing some interesting results. We should have some more to say about this in the coming weeks.</p>
<p>Last week I met with <a href="https://digitalflapjack.com/">Michael Dales</a>, and he talked about the project <a href="https://github.com/quantifyearth/shark">shark</a> that he and <a href="https://jon.recoil.org/patrick.sirref.org">Patrick Ferris</a> have been working on. It's kind of a mix between a shell and a jupyter-style notebook, with a strong focus on reproducibility. The traditional pain of notebooks is, of course, the execution model, whereby cells might be executed in any order you like. This means that the state you find the notebook in might not be even reachable again, let alone consistently reproducible. Shark is trying to address this by using file-system snapshots and clever analysis of the inputs and outputs of each cell to both ensure reproducibility, but also to allow a fast editing cycle, rerunning of only the bits that need to be rerun, even in the presence of slow data processing steps. It's a fascinating project, and I can't wait to see it in action when Michael gives us a demo!</p>
<p>I also met up with <a href="https://ryan.freumh.org">Ryan Gibb</a> with <a href="https://www.dra27.uk/blog/">David Allsopp</a> and we had a good chat about his project <a href="https://github.com/RyanGibb/babel">Babel</a>, which is using the <a href="https://nex3.medium.com/pubgrub-2fb6470504f">PubGrub</a> algorithm to do package resolution for multiple package domains at once. We've got a number of avenues to explore here, from building a PubGrub implementation in OxCaml, to using Babel to construct Docker images for opam packages entirely from scratch, without using a base image.</p>
<p>With my other hat on as a member of the CTO office at <a href="https://tarides.com/">Tarides</a>, I'm very much looking forward to using OCaml and OxCaml to solve some real-world problems that are in an entirely different domain than I've been used to over the last few years.</p>
