---
title: '.plan-26-17: Unwedging kernels, dogfood deployments, and managing beef leakage'
description: Welcoming Akshay to Cambridge, TESSERA AWS sync done, oi now self-hosts
  this site, and a new 4C forest leakage preprint appears.
url: https://anil.recoil.org/notes/2026w17
date: 2026-04-26T00:00:00-00:00
preview_image: https://anil.recoil.org/images/welcome-akshay.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>After the <a href="https://anil.recoil.org/notes/2026w16">travel marathon</a> of the past fortnight I got to catch up with hacking this week! I did pop down to London once for a Royal Society <a href="https://anil.recoil.org/notes/rs-eu-ai-science">policy meeting on AI in science with the European Commission</a> and discovered that the EU still has a (much-shrunken) <a href="https://www.eeas.europa.eu/delegations/united-kingdom_en?s=3225">delegation</a> in London; a bit of post-Brexit infrastructure that I hadn't appreciated existed but am very glad to hear about.</p>
<p>While I was down there, I caught up with <a href="https://web.eecs.umich.edu/~comar/">Cyrus Omar</a> who was over from Michigan to chat about our <a href="https://anil.recoil.org/papers/2025-fairground">Fairground</a> <a href="https://anil.recoil.org/projects/enki">planetary wiki</a> work. We're both interested in how programmable wikis can become a serious substrate for sharing structured scientific data with provenance baked in, and PROPL 2026 is coming up in PLDI where we'll do more work on this.</p>
<h2><a href="https://anil.recoil.org/news.xml#welcoming-akshay-to-cambridge" class="anchor" aria-hidden="true"></a>Welcoming Akshay to Cambridge</h2>
<p>I'm most delighted to welcome <a href="https://oppi.li/">Akshay Oppiliappan</a> to my group here in Cambridge! I've
long been a fan of his work on <a href="https://tangled.org">Tangled</a>, and indeed
consider it to be <a href="https://anil.recoil.org/notes/atproto-for-fun-and-blogging">the most useful app built over ATProto</a>.</p>
<p>We've been <a href="https://anil.recoil.org/notes/tangled-and-ci">using Tangled</a> for a lot of our <a href="https://anil.recoil.org/notes/disentangling-git-with-bluesky">code hosting</a> here in my
group, and it's a really practical way to get towards some of the things we
want to do for building federated scientific infrastructure under the <a href="https://anil.recoil.org/notes/principles-for-collective-knowledge">five principles of collective knowledge</a>.</p>
<p><img src="https://anil.recoil.org/images/welcome-akshay.webp" alt="%c" title="Akshay, Jon, Mark and me hang out in my jungle"></p>
<p>One interesting thing I learnt is that Tangled is working on a separable
'app view' (that is, a version of the <a href="https://tangled.org">https://tangled.org</a> website that can
be deployed elsewhere). I'd love to have a version that is restricted to just the
immediate group members in order to help get a focussed view on a particular
set of repositories, while still keeping the overall metadata open.</p>
<h2><a href="https://anil.recoil.org/news.xml#tessera-aws-sync-done-zarr-bindings-next" class="anchor" aria-hidden="true"></a>TESSERA: AWS sync done, Zarr bindings next</h2>
<p>The big milestone on the <a href="https://anil.recoil.org/projects/tessera">TESSERA</a> side is that the <a href="https://anil.recoil.org/notes/2026w16">AWS Open Data sync</a> has finally finished, so we now have the full half-petabyte mirrored alongside our <a href="https://anil.recoil.org/notes/2026w14">Cambridge Ceph</a> copy. With that done, I'm turning my attention to the OxCaml Zarr conversion by building on Mark's <a href="https://github.com/mtelvers/ocaml-zarr">ocaml-zarr</a> work, so that we can start consuming the cloud-native stores directly via HTTP.</p>
<p>There are also some exciting updates coming soon about a new version of the
TESSERA model that pushes the embedding quality further. The nice property of
how we've architected "embeddings as data" is that no user-facing code will need to
change when v1.1 lands. We just regenerate the map tiles under the existing
<a href="https://anil.recoil.org/notes/tessera-embeddings-convention">geo-embeddings convention</a> and downstream
tasks should pick up the improvements automatically. More on this once the
embeddings generation progresses!</p>
<h2><a href="https://anil.recoil.org/news.xml#recoil-refresh-to-linux-70" class="anchor" aria-hidden="true"></a>Recoil refresh to Linux 7.0</h2>
<p>On the home infrastructure front, I spent some quality time upgrading several of the <a href="https://anil.recoil.org/notes/decentralised-stack">Recoil self-hosting</a> machines to <strong>Ubuntu 26.04</strong>. I have <strong>not</strong> been able to recreate the pesky <a href="https://github.com/openzfs/zfs/issues/16133">io_uring/zfs wedge</a> that has been plaguing me even on 6.14 kernels recently. Fingers crossed that it really is fixed and not just hiding behind a race condition!</p>
<p>I've also been happily using <a href="https://komo.do/">Komodo</a> as the lightweight web interface for Docker across three machines, and am busy migrating to Mythic Beasts since our former Equinix hosting is <a href="https://www.datacenterdynamics.com/en/news/equinix-to-kill-off-metal-by-june-2026/">sunsetting</a> next month. The only technical complexity here is that <a href="https://docs.joinmastodon.org/admin/migrating/">Mastodon is tied to one hostname</a> and I made a mistake calling it <a href="https://amok.recoil.org">amok.recoil.org</a> (the raw hostname) instead of something more abstract. <a href="https://mynameismwd.org">Michael Dales</a> did <a href="https://digitalflapjack.com/weeknotes/2025-03-31/">manage to migrate</a> last year though, which is a good sign when I try next week...</p>
<h2><a href="https://anil.recoil.org/news.xml#oi-continues-and-now-deploys-this-very-site" class="anchor" aria-hidden="true"></a>oi continues, and now deploys this very site</h2>
<p>My sidequest on <a href="https://github.com/avsm/oi"><strong>oi</strong></a>, my <a href="https://anil.recoil.org/notes/2026w16">uv-like distributor for OCaml binaries</a>, has been steadily gaining steam. It now supports <a href="https://anil.recoil.org/projects/oxcaml">OxCaml</a> as well as multiple OCaml versions, which is tricky since OxCaml isn't relocatable yet. Still, some hacks later, I've got far enough that I'm quietly using it for myself day-to-day to see if the tool holds up under real development workloads.</p>
<p>This very website is now deployed using:</p>
<pre><code>oi run --toolchain=oxcaml @avsm/arod -- arod serve -v
</code></pre>
<p>This feels like a nice eating-my-own-dogfood moment! I'll write up technical
details properly once I've stopped rewriting the implementation.</p>
<p>I've also been working with <a href="https://github.com/samoht">Thomas Gazagnaire</a> to merge his significant changes from the
last four months into the <a href="https://anil.recoil.org/notes/aoah-2025">agentic libraries</a> I built last year, so
we can reconcile our diverging trees. He's been hacking on these in his
<a href="https://tangled.org/gazagnaire.org/monopampam">monopampam</a> tree and there's a
lot of cleanup to bring across.</p>
<h3><a href="https://anil.recoil.org/news.xml#cross-building-ocaml-windows-binaries" class="anchor" aria-hidden="true"></a>Cross-building OCaml Windows binaries</h3>
<p>Apropos to the above, I've been poking at
<a href="https://github.com/msys2/msys2-docker">msys2-docker</a> to see if I could compile
OCaml Windows binaries directly from Linux without doing full
cross-compilation. It does <em>almost</em> work but the layering of MSYS2 inside Wine
is unreliable due to fork not working very well. <a href="https://dave.recoil.org">Dave Scott</a> then mentioned to me
over a coffee that it's possible to do this more directly via Wine running
<code>cmd.exe</code>, by extracting the necessary bits out of a <code>nanoserver</code> Docker image.
That sounds way better, so I'll try that approach next week.</p>
<h2><a href="https://anil.recoil.org/news.xml#a-new-forest-leakage-preprint" class="anchor" aria-hidden="true"></a>A new forest leakage preprint</h2>
<p>A new preprint has gone up from our <a href="https://anil.recoil.org/projects/4c">4C</a> trusted carbon credits work, led by the wonderful <a href="mailto:fd370@cam.ac.uk">Francisco d'Albertas</a>.</p>
<p>This one's about <a href="https://www.researchsquare.com/article/rs-9440067/v1">"Estimating the carbon impacts of leakage from forest restoration and the costs of reducing them"</a>. The abstract:</p>
<blockquote>
<p>Ecosystem restoration is a key nature-based climate solution but risks
displacing economic activities and triggering leakage – whereby forgone
production drives habitat loss elsewhere, eroding benefits. Focusing on
reforestation opportunities Brazilian ranchland we characterized leakage risk
as the ratio of forgone beef production to carbon gained.</p>
<p>Assuming 100% of forgone production results in extensification we asked: what
is the impact of unaddressed leakage; how much can leakage be reduced by
prioritizing restoration in low-yielding, high-carbon areas; and can it be
cost-effectively mitigated by targeted intensification?</p>
<p>Taking likely leakage into account but
not tackling it increased median costs of restoration (over ignoring it
entirely) by 43-100%, to median values of 33 and 24 USD tCO₂e⁻¹ in the Atlantic
Forest and Amazon, respectively. Prioritizing low-leakage sites reduced these
costs by 21–37%; combining this with targeted intensification cut net carbon
costs further, to 67% of unmitigated levels. Our broad findings hold at 30%
(cf 100%) extensification and in other sensitivity analyses, and reveal
leakage can substantially increase carbon costs, but that careful siting and
targeted intensification can provide extremely cost-effective mitigation.
<cite>-- <a href="https://anil.recoil.org/papers/2026-forest-leakage">d'Albertas et al., 2026</a></cite></p>
</blockquote>
<p>This pushes our <a href="https://anil.recoil.org/papers/2025-forest-restoration-cc">forest restoration</a> analyses onto the all-important "leakage" question, which is something of an elephant in the room for almost any nature-based climate intervention (if we choose our interventions badly, then displacement of existing use of that land causes yet more deforestation). Congratulations to Chico and the rest of the team for getting this out!</p><h1>References</h1><ul><li>Wheeler et al (2025). The path to robust evaluation of carbon credits generated by forest restoration and REDD+ projects. <a href="https://doi.org/10.1016/j.rse.2025.115041" target="_blank"><i>10.1016/j.rse.2025.115041</i></a></li>
<li>Madhavapeddy (2026). TESSERA now supports the Zarr geo-embeddings convention proposal. <a href="https://doi.org/10.59350/c3hrq-zsx02" target="_blank"><i>10.59350/c3hrq-zsx02</i></a></li>
<li>d'Albertas et al (2026). Estimating the carbon impacts of leakage from forest restoration and the costs of reducing them. Research Square. <a href="https://doi.org/10.21203/rs.3.rs-9440067/v1" target="_blank"><i>10.21203/rs.3.rs-9440067/v1</i></a></li>
<li>Madhavapeddy (2025). Socially self-hosting source code with Tangled on Bluesky. <a href="https://doi.org/10.59350/r80vb-7b441" target="_blank"><i>10.59350/r80vb-7b441</i></a></li>
<li>Madhavapeddy (2025). Four Ps for Building Massive Collective Knowledge Systems. <a href="https://doi.org/10.59350/418q4-gng78" target="_blank"><i>10.59350/418q4-gng78</i></a></li>
<li>Omar et al (2025). A FAIR Case for a Live Computational Commons. Association for Computing Machinery. <a href="https://doi.org/10.1145/3759536.3763802" target="_blank"><i>10.1145/3759536.3763802</i></a></li>
<li>Madhavapeddy (2025). Using AT Proto for more than just Bluesky posts. <a href="https://doi.org/10.59350/32rdt-zny05" target="_blank"><i>10.59350/32rdt-zny05</i></a></li>
<li>Madhavapeddy (2025). mlgpx is the first Tangled-hosted package available on opam. <a href="https://doi.org/10.59350/7267y-nj702" target="_blank"><i>10.59350/7267y-nj702</i></a></li></ul>
