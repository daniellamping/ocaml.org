---
title: '.plan-26-16: Chennai, Cambridge, Belfast: a week on the wing'
description: A week of hops between Chennai, Cambridge and Belfast for the FP Launchpad
  takeoff at IIT Madras, a surprise Publication of the Year at the Cambridge Ring
  Hall of Fame, meeting the VC on the upcoming Rokos School of Governance, mirroring
  half a petabyte of TESSERA tiles and hacking on oi
url: https://anil.recoil.org/notes/2026w16
date: 2026-04-19T00:00:00-00:00
preview_image: https://anil.recoil.org/images/26w16-1.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p><img src="https://anil.recoil.org/images/26w16-1.webp" alt="%rc" title="The water service at Sashwatha cafe in Chennai was striking">
I spent most of the week in the air with hops to Chennai, Cambridge and Belfast. The reason was the <strong><a href="https://anil.recoil.org/notes/fpl-launch">FP Launchpad 'takeoff'</a> at IIT Madras</strong> where I spent half the week, in a campus teeming with banyan trees and monkeys! I've sketched out two project ideas inspired by the visit: an <a href="https://anil.recoil.org/ideas/lean-io-uring-backend">io_uring backend for Lean</a> and an <a href="https://anil.recoil.org/ideas/spytial-ocaml-port">OCaml port of sPyTial</a>, with more to come this week as I catch up.</p>
<h2><a href="https://anil.recoil.org/news.xml#paper-of-the-year-at-the-cambridge-ring-hall-of-fame" class="anchor" aria-hidden="true"></a>Paper of the Year at the Cambridge Ring Hall of Fame</h2>
<p><img src="https://anil.recoil.org/images/26w16-5.webp" alt="%rc" title="Celebrating with Dave on the Queens' bridge!">
Back on home turf in Cambridge, the <a href="https://www.cambridgering.org.uk/">Cambridge Ring</a> held its annual <a href="https://www.cst.cam.ac.uk/news/celebrating-culture-innovation-our-hall-fame-awards">Hall of Fame Awards</a> at Queens' College. In previous years they tell you if you've won, but this year it switched to an "Oscar style" nomination mechanism. After a tense set of announcements of the runners up (well, cheering on <a href="https://jonmsterling.com">Jon Sterling</a> actually!) I was delighted that our ICFP 2025 paper <em><a href="https://anil.recoil.org/papers/2025-docker-icfp">"Functional Networking for Millions of Docker Desktops"</a></em> won Publication of the Year! (see <a href="https://www.linkedin.com/feed/update/urn:li:activity:7450463144923058177/">LI</a> or <a href="https://bsky.app/profile/anil.recoil.org/post/3mjlzvcn7yk2w">Bluesky</a>)</p>
<p>My huge thanks to my coauthors <a href="https://dave.recoil.org">Dave Scott</a>, <a href="https://patrick.sirref.org">Patrick Ferris</a>, <a href="https://ryan.freumh.org">Ryan Gibb</a> and <a href="https://github.com/samoht">Thomas Gazagnaire</a>, and to the cast of hundreds at Docker, Cambridge, California and beyond who made the decade of work behind it possible. There's a companion <a href="https://anil.recoil.org/papers/2026-decade-docker">CACM piece</a> with <a href="https://dave.recoil.org">Dave Scott</a> and <a href="https://github.com/justincormack">Justin Cormack</a> that covers the broader arc, but it's the ICFP experience report that has the OCaml networking nitty gritty details that I sunk a lot of time into over the years.</p>
<h2><a href="https://anil.recoil.org/news.xml#visiting-the-old-combination-room" class="anchor" aria-hidden="true"></a>Visiting the Old Combination Room</h2>
<p><img src="https://anil.recoil.org/images/26w16-3.webp" alt="%rc" title="The VC addressing the Cambridge academics and staff in the OCR">
On getting back to Cambridge on a redeye flight, I got invited to the <a href="https://www.admin.cam.ac.uk/offices/em/old-schools/">Old Schools Combination Room</a> with the Vice-Chancellor <a href="https://en.wikipedia.org/wiki/Deborah_Prentice">Deborah Prentice</a> to celebrate and to discuss the upcoming <a href="https://www.cam.ac.uk/stories/rokos-school-of-government">Rokos School of Governance</a>. It was lovely to catch up with <a href="https://www.cfse.cam.ac.uk/directory/marla_fuchs">Marla Fuchs</a>, who has done an enormous amount of work behind the scenes to make all of this happen, and also to hear her toil get properly acknowledged in the room full of very senior people!</p>
<p><img src="https://anil.recoil.org/images/26w16-4.webp" alt="%rc" title="The assembled Cambridge staff and academics">
The conversations about what a 21st-century governance school should look like were
very animated. More on my thoughts as they form in the coming months, but this
feels like a real opportunity in Cambridge.</p>
<h2><a href="https://anil.recoil.org/news.xml#tessera-tiles-on-aws-and-a-multi-registry-client" class="anchor" aria-hidden="true"></a>TESSERA: tiles on AWS and a multi-registry client</h2>
<p><img src="https://anil.recoil.org/images/26w16-2.webp" alt="%rc" title="Marla and me kick back in the Old Combination Room!">
<a href="https://www.tunbury.org/">Mark Elvers</a> and I have been grinding through the migration of <a href="https://anil.recoil.org/projects/tessera">TESSERA</a> tiles to AWS Open Data. Mark has written up the nuts and bolts in two posts:</p>
<ul>
<li><a href="https://www.tunbury.org/2026/04/17/geotessera-stac/">GeoTessera STAC</a> on exposing the embeddings via a <a href="https://stacspec.org/">STAC</a> catalogue so clients can discover tiles using standard spatiotemporal queries. Zarr v3 doesn't have an indexing mechanism that's standard, which is an unusual (but I think deliberate) omission.</li>
<li><a href="https://www.tunbury.org/2026/04/17/cephfs-to-s3/">CephFS to S3</a> on the actual bulk transfer from our Cambridge Ceph cluster to AWS, including the tuning of parallelism and retry behaviour to make a half-a-petabyte transfer go faster. The first iteration without the tweaks would have taken two months of transfer time, but we got it down to a week...</li>
</ul>
<p>With the precious TESSERA data now in two sites at last, I've been building the multi-registry support into <code>geotessera</code> itself so the client can discover and fetch tiles transparently from Cambridge, AWS or other future mirrors. This builds on the <a href="https://anil.recoil.org/notes/tessera-zarr-v3-layout">Zarr v3 layout</a> and <a href="https://anil.recoil.org/notes/tessera-embeddings-convention">geo-embeddings convention</a> and gets us closer to a proper federated story for TESSERA data distribution. I'm also meeting up with <a href="https://web.eecs.umich.edu/~comar/">Cyrus Omar</a> and his group in London next week as they are visiting, as this is related to our <a href="https://anil.recoil.org/projects/enki">planetary wiki</a> that we <a href="https://anil.recoil.org/papers/2025-fairground">wrote up</a> last year.</p>
<h2><a href="https://anil.recoil.org/news.xml#oi-a-uv-like-distributor-for-ocaml-binaries" class="anchor" aria-hidden="true"></a>oi: a uv-like distributor for OCaml binaries</h2>
<p>Now that more people outside our immediate circle are <a href="https://anil.recoil.org/projects/oxcaml">using OCaml in production</a> across the group, the issue of <em>"how to run this OCaml CLI tool without CLI gymnastics"</em> has started to bite. opam has always been great once set up, but it's a lot of machinery for someone who just wants to run the TESSERA CLI or some utility.</p>
<p>I've been spending a lot of time with <a href="https://github.com/astral-sh/uv">uv</a> in recent months while working on the Python machine learning end of TESSERA, and it has become my default choice to <a href="https://anil.recoil.org/notes/geotessera-python">ship Python tooling</a>. I spotted an opportunity to get this Python goodness over to my statically typed world and have been hacking on <a href="https://github.com/avsm/oi"><strong>oi</strong></a>: a fast, stateless client that fetches and manages binary releases of OCaml tooling with a single invocation.</p>
<p>The idea is much older than my prototype code. Back in 2023 I sketched out an <a href="https://github.com/avsm/opam-repo-roadmap-thoughts">opam-repo roadmap</a> around a merge-queue-driven overlay repository: rather than each user resolving the whole universe on their laptop, a central CI would continuously solve and build the overlay, and clients would just pull pre-resolved, pre-built artefacts. What I was missing was a clean way to actually <em>execute</em> the builds reproducibly.</p>
<p>Two members of my group came up with the answers. Firstly, <a href="https://www.dra27.uk">David Allsopp</a> got his <a href="https://www.dra27.uk/blog/platform/2025/12/17/its-merged.html">relocatable OCaml compiler patches merged</a> after a year of hard work. Then <a href="https://www.tunbury.org/">Mark Elvers</a> came up with the <a href="https://www.tunbury.org/2026/04/17/day10-build/">day10 build</a> tool that builds opam packages inside OCI containers with layer caching, and also an <a href="https://www.tunbury.org/2026/04/02/opam-overlay-ci/">opam overlay CI</a> that wires up the GitHub merge queue so regressions can be caught before a PR lands.</p>
<p>Some cool things you can do with oi today:</p>
<ul>
<li><code>oi run utop</code> gets you the toplevel quickly.</li>
<li><code>oi run --with=async utop</code> gets you the toplevel with Async loaded.</li>
<li><code>oi run --with=https://tangled.org/patrick.sirref.org/merry msh</code> gets you running with the <code>msh</code> binary from <a href="https://patrick.sirref.org/weekly-2026-w12/index.xml">Merry</a></li>
<li><code>oi run https://www.cl.cam.ac.uk/~avsm2/foo.ml</code> runs a remote script without a dune file being needed!</li>
</ul>
<p>It does this with the simple trick of adding OCaml attributes to the toplevel, just as <a href="https://packaging.python.org/en/latest/specifications/inline-script-metadata/#inline-script-metadata">Python inline script metadata</a> does. oi then synthesises a dune file and adds ppx preprocessors in via heuristics. You can see my version of the package attributes in the snippet of OCaml below:</p>
<pre><code class="language-ocaml">[@@@opam base stdio ppx_jane]

open Base
open Stdio

type t = { bar: float } [@@deriving sexp]

let rec read_and_accumulate accum =
  let line = In_channel.input_line In_channel.stdin in
  match line with
  | None -&gt; accum
  | Some x -&gt; read_and_accumulate (accum +. Float.of_string x)

let () =
  let t = { bar=read_and_accumulate 0. } in
  printf "Total: %s\n" (Sexp.to_string_hum (sexp_of_t t))
</code></pre>
<p>So to recap, with day10 supplying a reproducible build substrate and OCaml now
being relocatable, that's all I needed to glue together this <code>oi</code> tool that
apes uv!  It's early and rough and only really intended for local use, but
issues/opinions are very welcome especially around signing and platform
compatibility. I'll be blogging about the technical details more this week,
and switching to using it day to day to make sure it's good enough before sharing more widely.
Having said that, it <a href="https://patrick.sirref.org/self-host-music/index.xml">already seems to have escaped into the wild</a>.</p>
<h2><a href="https://anil.recoil.org/news.xml#fun-links" class="anchor" aria-hidden="true"></a>Fun links</h2>
<h3><a href="https://anil.recoil.org/news.xml#rcts-as-a-survival-skill" class="anchor" aria-hidden="true"></a>RCTs as a survival skill</h3>
<p>A great FT piece this week on <a href="https://www.ft.com/content/9b8eebc6-ed76-41cf-bc46-a1ef84613218">"The trials that quietly changed our lives"</a> (h/t <a href="https://bsky.app/profile/hetanshah.bsky.social">Hetan Shah</a>) on how randomised controlled trials and the steady accumulation of evidence have underpinned most of the quiet improvements in modern life:</p>
<blockquote>
<p>Arming our children — and ourselves — with the ability to spot bunk and think
critically about claims has become an essential survival skill.
<cite>-- <a href="https://www.ft.com/content/9b8eebc6-ed76-41cf-bc46-a1ef84613218">The trials that quietly changed our lives</a>, 2026</cite></p>
</blockquote>
<p>This hits on what I've been working on with the <a href="https://anil.recoil.org/projects/ce">Conservation Evidence</a> and the <a href="https://anil.recoil.org/papers/2025-evidence-tap">evidence TAP</a> teams in recent months. The world's awash with confidently incorrect LLM-generated assertions, and the ability to trace a claim back to an actual evidentiary test isn't just a niche skill for academics any more.</p>
<h3><a href="https://anil.recoil.org/news.xml#hamed-haddadi-visits-me-and-mort" class="anchor" aria-hidden="true"></a>Hamed Haddadi visits me and Mort</h3>
<p>There have been persistent rumours that <a href="https://haddadi.github.io/">Hamed Haddadi</a> and I are the same person, and I hope that this evidence from Christ's College Cambridge at a delightful high table hosted by <a href="https://github.com/mor1">Richard Mortier</a> will resolve this situation. Thank you for your attention to this matter.</p>
<p><img src="https://anil.recoil.org/images/26w16-6.webp" alt="%c" title="Hamed? Or is it?">
<img src="https://anil.recoil.org/images/26w16-7.webp" alt="%c" title="Anil? Or is it?"></p><h1>References</h1><ul><li>Madhavapeddy et al (2025). Functional Networking for Millions of Docker Desktops. <a href="https://doi.org/10.1145/3747525" target="_blank"><i>10.1145/3747525</i></a></li>
<li>Madhavapeddy (2026). The FP Launchpad takes off at IIT Madras. <a href="https://doi.org/10.59350/4bsr3-h6735" target="_blank"><i>10.59350/4bsr3-h6735</i></a></li>
<li>Jaffer et al (2025). AI-assisted Living Evidence Databases for Conservation Science. Cambridge Open Engage. <a href="https://doi.org/10.33774/coe-2025-rmsqf" target="_blank"><i>10.33774/coe-2025-rmsqf</i></a></li>
<li>Madhavapeddy et al (2026). A Decade of Docker Containers. <a href="https://doi.org/10.1145/3761803" target="_blank"><i>10.1145/3761803</i></a></li>
<li>Madhavapeddy (2026). TESSERA now supports the Zarr geo-embeddings convention proposal. <a href="https://doi.org/10.59350/c3hrq-zsx02" target="_blank"><i>10.59350/c3hrq-zsx02</i></a></li>
<li>Madhavapeddy (2026). Streaming millions of TESSERA tiles over HTTP with Zarr v3. <a href="https://doi.org/10.59350/tk0er-ycs46" target="_blank"><i>10.59350/tk0er-ycs46</i></a></li>
<li>Omar et al (2025). A FAIR Case for a Live Computational Commons. Association for Computing Machinery. <a href="https://doi.org/10.1145/3759536.3763802" target="_blank"><i>10.1145/3759536.3763802</i></a></li>
<li>Madhavapeddy (2025). GeoTessera Python library released for geospatial embeddings. <a href="https://doi.org/10.59350/7hy6m-1rq76" target="_blank"><i>10.59350/7hy6m-1rq76</i></a></li></ul>
