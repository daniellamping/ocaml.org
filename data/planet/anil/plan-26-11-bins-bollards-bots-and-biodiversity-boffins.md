---
title: '.plan-26-11: Bins, bollards, bots and biodiversity boffins'
description: Evidence synthesis at the DEFRA science conference, TESSERA transcoding
  and building a new SPA, OpenStreetMap/DuckDB bindings in OxCaml, and early thoughts
  on vibecoding etiquette.
url: https://anil.recoil.org/notes/2026w11
date: 2026-03-15T00:00:00-00:00
preview_image: https://anil.recoil.org/images/defra-etap-2.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<h2><a href="https://anil.recoil.org/news.xml#evidence-synthesis-at-the-defra-science-conference" class="anchor" aria-hidden="true"></a>Evidence Synthesis at the DEFRA science conference</h2>
<p><img src="https://anil.recoil.org/images/defra-etap-2.webp" alt="%c" title="Completely crammed event; lots of interest in AI driven evidence gathering!"></p>
<p>Starting with <a href="https://anil.recoil.org/projects/ce">evidence synthesis</a>, we got invited after our <a href="https://anil.recoil.org/notes/red-pill-conservation">previous meetings</a> with DEFRA to run a session about our <a href="https://anil.recoil.org/papers/2025-evidence-tap">evidence TAP</a> at their annual <em>"DEFRA Science, Analysis and Data Professions Conference"</em> in London. I couldn't make this one in person, but <a href="https://toao.com">Sadiq Jaffer</a> and <a href="https://samreynolds.org">Sam Reynolds</a> did their usual brilliant double act with help from the rest of the CE team.</p>
<p><img src="https://anil.recoil.org/images/defra-etap-3.webp" alt="%rc">
One little shocking fact I learnt is that DEFRA aren't covered by same sorts of publishing agreements we enjoy via <a href="https://www.jisc.ac.uk/">JISC</a>, which means they need to separately negotiate and pay up to all the publishers. While I'm all for <a href="https://anil.recoil.org/notes/rs-future-of-publishing">sustainable publishing</a>, it's incredibly inefficient to have public funds support a bunch of research which then needs to be repurchased by the government agency seeking to take nature positive decisions. I'm feeling the call of <a href="https://anil.recoil.org/notes/coar-prc">COAR and open publishing</a> more and more...</p>
<p></p><div class="video-center"><iframe title="Cambridge Evidence TAP OpenGL interactive visualiser" width="100%" height="315px" src="https://watch.eeg.cl.cam.ac.uk/videos/embed/32884cdf-cdb1-487e-b413-9c12d0d0ad09" frameborder="0" allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups allow-forms"></iframe></div><p></p>
<p>Also congratulations to 15-year-old Jens Kromdijk, who interned with us in a school placement last summer, and had his OpenGL knowledge graph visualiser showcased in front of DEFRA in last week's talk by Sam! The video of this in action is above, and it's a very cool piece of game engine programming repurposed for interactive visualisation of the academic literature. Nice work Jens!</p>
<p><img src="https://anil.recoil.org/images/defra-etap-4.webp" alt="%c" title="Sam shows off Jens' OpenGL viewer on stage"></p>
<h2><a href="https://anil.recoil.org/news.xml#tessera-hacking" class="anchor" aria-hidden="true"></a>TESSERA hacking</h2>
<p>I spent a bunch of time on <a href="https://anil.recoil.org/notes/tessera-zarr-v3-layout">figuring out TESSERA and Zarr v3</a> layouts, and working on getting adequate parallel performance out of it. The embeddings for 2024 and 2025 are now transcoding and pyramiding, so it'll be a few more days and can test these out properly.</p>
<h3><a href="https://anil.recoil.org/news.xml#learning-spa-and-typescript" class="anchor" aria-hidden="true"></a>Learning SPA and Typescript</h3>
<p>I figure I need to learn 'modern' web programming while building the <a href="https://tze.geotessera.org">TZE explorer</a> so I knocked up a website to aggregate information about <a href="https://anil.recoil.org/projects/tessera">TESSERA</a>. I used the latest <a href="https://vite.dev/">Vite v8</a> and <a href="https://rolldown.rs/">Rolldown</a> along with <a href="https://svelte.dev/">Svelte</a>, and resisted the urge to do this in OCaml so I could learn about another language ecosystem!</p>
<p>The experience of <a href="https://github.com/ucam-eo/geotessera.org">building the SPA with Claude</a> was straightforward, except for the usual problem of versioning going wrong. I had to manually intervene to do the fairly complex npm version upgrade (the agent picked Vite6 and Rollup, and I needed Vite8 and Rolldown). Deploying the SPA to GitHub Pages was a bit complicated as all unknown routes need to be redirected to a single index file, so this has to be customised to the static page provider. Since <a href="https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-custom-404-page-for-your-github-pages-site">GitHub Pages serves a 404.html</a>, I had to patch the site to generate stub 404.htmls in all the subdirectories so that they could be navigated to.</p>
<p><a href="https://geotessera.org"> <img src="https://anil.recoil.org/images/tessera-web-ss-1.webp" alt="%c" title="The geotessera.org website, as a single page application"> </a></p>
<p>Anyway, after all this we ended up with a nice <a href="https://geotessera.org">geotessera.org</a> site. I'd like to switch to using the shiny new <a href="https://docs.tangled.org/hosting-websites-on-tangled.html">Tangled Pages</a> but am just waiting on custom domains support first. Congratulations to Akshay and Anirudh on shipping the complicated feature in Tangled! This is a convenient place where we can post news such as <a href="https://geotessera.org/blog/2026-03-10-tessera-v1-weights">the v1 model weights being released</a>.</p>
<p>I've added a TESSERA Atom feed <a href="https://geotessera.org/blog/feed.xml">with all posts</a> and <a href="https://geotessera.org/blog/feed-original.xml">only original posts</a> (i.e. those not federated from elsewhere), so get your feed readers subscribed!</p>
<h3><a href="https://anil.recoil.org/news.xml#relevant-geospatial-papers" class="anchor" aria-hidden="true"></a>Relevant geospatial papers</h3>
<p>I ran across a nice whitepaper from <a href="https://github.com/Element84/vector-embeddings-catalog-whitepaper/blob/main/VectorEmbeddings_WhitePaper_June2025.pdf">Element 84 on a vector embeddings marketplace</a> from last summer.
Then "<a href="https://arxiv.org/abs/2603.02080">From Pixels to Patches: Pooling Strategies for Earth Embeddings</a>" is a nice systematic view on how
to aggregate embeddings:</p>
<blockquote>
<p>As geospatial foundation models shift from patch-level to pixel-level
embeddings, practitioners must aggregate thousands of pixel vectors into
patch representations that preserve class-discriminative signal while
matching downstream label resolution.</p>
<p>Tessera (512-d) shows the largest mean-pooling gap (12%), suggesting
higher-dimensional embeddings benefit most from summary statistics.
<cite>-- <a href="https://arxiv.org/abs/2603.02080">From Pixels to Patches: Pooling Strategies for Earth Embeddings</a>, Corley et al, 2026</cite></p>
</blockquote>
<p>This is particularly timely as we're working on <a href="https://huggingface.co/blog/matryoshka">Matryoshka embeddings</a> which should bring a fresh (and more information rich) twist to this when they're ready.</p>
<p><a href="https://patball1.github.io">James G. C. Ball</a> also pushed out an excellent preprint of doing <a href="https://anil.recoil.org/papers/2026-tessera-trentino">data-efficient tree species mapping in temperate mountain forests</a>, which is essential reading if you're building classifiers or segmenters using models like TESSERA!</p>
<h2><a href="https://anil.recoil.org/news.xml#working-on-oxcaml-in-oxmono" class="anchor" aria-hidden="true"></a>Working on OxCaml in OxMono</h2>
<p>I've continued working on a bunch of side OCaml libraries in <a href="https://github.com/avsm/oxmono">OxMono</a>.   Firstly thanks to <a href="https://jon.recoil.org">Jon Ludlam</a> for <a href="https://github.com/avsm/oxmono/pull/3">getting me oxdoc HTML working</a> in my repo!</p>
<h3><a href="https://anil.recoil.org/news.xml#openstreetmap-and-duckdb-in-oxcaml" class="anchor" aria-hidden="true"></a>OpenStreetMap and DuckDB in OxCaml</h3>
<p>Both <a href="https://tze.geotessera.org">TZE</a> and the <a href="https://anil.recoil.org/projects/enki">Enki</a> dashboards that <a href="https://shaneweisz.com">Shane Weisz</a> is working on could also use vector tiles for human activities, ideally via local services. I built an OxCaml <a href="https://openstreetmap.org">OpenStreetMap</a> converter from their <a href="https://wiki.openstreetmap.org/wiki/PBF_Format">compressed protobuf format</a> so that I do queries via an in-process DuckDB from OCaml. This allows me to do rapid queries for various vector tags, such as finding all the bollards or horse-friendly gates in the world:</p>
<pre><code>&gt; select * from nodes where map_contains(nodes.tags, 'barrier');
┌──────────────────────────────┬──────────────────────┬───────────────────────────────────────────────
│  id    │        lat          │         lon          │                     tags                     
│ i64    │       double        │        double        │             map(varchar, varchar)             
├────────┼─────────────────────┼──────────────────────┼───────────────────────────────────────────────
│ 291281 │          51.8116825 │  -0.8396488000000001 │ {access=private, barrier=gate}                 
│ 291711 │          51.8161508 │           -0.8351806 │ {barrier=bollard, motor_vehicle=no}            
│ 155806 │   60.65463260000001 │    7.881729600000001 │ {barrier=cattle_grid}                          
│ 402757 │  50.917684400000006 │           -1.4037637 │ {barrier=bollard, bollard=fixed}               
│ 198617 │          59.3074218 │           17.9634278 │ {barrier=yes, bicycle=yes, foot=yes, horse=yes} 
│ 392357 │  45.082863100000004 │   2.7073557000000004 │ {barrier=gate, bicycle=yes, foot=yes}        
│ 424724 │   51.60528540000001 │            -0.178978 │ {barrier=cycle_barrier, foot=yes}
</code></pre>
<p>I'll write more about this next week when it's more fleshed out, but the basic
bindings and CLI tools for OSM are in my
<a href="https://github.com/avsm/oxmono/tree/duckdb">oxmono#duckdb</a> branch for the very
curious. I'm still running performance matrices to figure out the best
parallelisation strategy for importing the millions of records involved, but
the local/stack/unboxing support is already a significant performance boost
and very usable (about an hour to import the full 250GB database, which is very
usable after that for queries).</p>
<h3><a href="https://anil.recoil.org/news.xml#cpu-and-gpu-inference" class="anchor" aria-hidden="true"></a>CPU and GPU inference</h3>
<p><a href="https://www.tunbury.org/">Mark Elvers</a> meanwhile has been figuring out how to do <a href="https://www.tunbury.org/2026/03/13/oxcaml-inference/">efficient CPU inference</a> using <a href="https://anil.recoil.org/notes/icfp25-oxcaml">OxCaml</a> SIMD and also <a href="https://www.tunbury.org/2026/03/11/gpu-vs-cpu/">GPU vs CPU NUMA</a> (which threw me back 12 years to <a href="https://www.youtube.com/watch?v=Ss4pUbq09Lw">my FOSDEM 2013 talk on NUMA</a>).</p>
<p>We have a lot of spare CPU compared to GPU, so this is a direction we'll likely go down to <a href="https://www.tunbury.org/2026/03/09/ocluster/">soak spare CPU</a> for relatively slow and steady inference of TESSERA tiles.</p>
<h3><a href="https://anil.recoil.org/news.xml#llm-protocols-for-sharing-code" class="anchor" aria-hidden="true"></a>LLM protocols for sharing code</h3>
<p>My monorepo is once again vastly diverging from colleagues'; for example
<a href="https://github.com/samoht">Thomas Gazagnaire</a> has his agentic <a href="https://tangled.org/gazagnaire.org/monopampam">monopampam</a> monorepo chocabloc
full of exciting new developments. None of the tooling we are building can
quite manage to keep things in sync due to the unbelievable throughput of a
well-prompted LLM.</p>
<p>But on the other hand, none of the code we're building is
fit for third party release without extensive code review (which I've yet to
do!). We're risking getting stuck in an uncanny valley of 'almost there' code,
which feels a bit like a return to the land of untyped Javascript!</p>
<p>While I've been punting having a firm opinion on this down the road as I'm not sharing much code yet, <a href="https://patrick.sirref.org">Patrick Ferris</a> has put together an excellent <a href="https://patrick.sirref.org/vibecoding-etiquette/index.xml">piece on vibecoding etiquette</a> which I agree with. <a href="https://jon.recoil.org">Jon Ludlam</a> has also adopted a slightly different (but compatible) policy of having a <a href="https://jon.recoil.org/blog/2026/01/weeknotes-2026-04-05.html#containers-vs-accounts">separate commit email for his agent</a> and depending on a rebase to 'own' the code:</p>
<blockquote>
<p>It's always slightly alarming to see my own name on the output of the bots,
assigning me (or sometimes someone else (!!)) copyright over code I've never
seen. This is, of course, a whole other pandora's box that I really don't
want to open right now - but I think the point is that I'll feel a lot more
comfortable if the commits are all by <code>Jon's Agent &lt;jon+claude@recoil.org&gt;</code>
rather than by me!
<cite>-- <a href="https://jon.recoil.org/blog/2026/01/weeknotes-2026-04-05.html#containers-vs-accounts">Containers vs accounts</a>, 2026-04, Jon Ludlam</cite></p>
</blockquote>
<p>Both of these seem right to me. When I'm back in Cambridge in a few weeks, I'm going to take a serious look at locally hosted models as well. Luke Marsden has been doing brilliant work on <a href="https://blog.helix.ml/">open agent infrastructure</a> that I've not had the bandwidth to try out yet.</p>
<h2><a href="https://anil.recoil.org/news.xml#fun-links" class="anchor" aria-hidden="true"></a>Fun Links</h2>
<ul>
<li>I've been watching <a href="https://loosemore.com/">Tom Loosemore</a> doing epic <a href="https://loosemore.com/2026/03/05/450/">hacking</a> with UK <a href="https://loosemore.com/2026/02/25/ai-agents-will-join-up-government-before-government-does/">government websites</a> in the past few months. After seeing his latest post <a href="https://loosemore.com/2026/03/13/beaten-by-the-bins/">about bin collection days</a> I couldn't resist pitching in <a href="https://bsky.app/profile/did:plc:nhyitepp3u4u6fcfboegzcjw/post/3mh3qbe4gak2d">to build a script</a> for him. There was a push for this sort of thing back in 2010 (via <a href="https://news.ycombinator.com/item?id=1584597">ScraperWiki or Dapper</a>), and perhaps resurrecting these in an agentic world would be a nice way for collaboration on providing programmatic access to otherwise fragmented local government interfaces.</li>
<li>"<a href="https://www.foodandwine.com/university-of-illinois-new-popcorn-11922114">The University of Illinois Just Released a Popcorn So Good It Doesn’t Need Butter</a>" is my science story of the week.</li>
<li><a href="https://wedistribute.org/">We Distribute</a> bridged onto <a href="https://anil.recoil.org/notes/atproto-for-fun-and-blogging">ATProto</a> <a href="https://bsky.app/profile/wedistribute.org/post/3mgojpfdnmk2w">last week</a> and is a nice collection news about the fediverse, ATproto and Matrix ecosystems.</li>
<li>It turns out that our UK spend on netzero is <a href="https://www.theguardian.com/environment/2026/mar/11/reaching-net-zero-by-2050-cheaper-for-uk-than-one-fossil-fuel-crisis">in total less than a single oil crisis</a>. Energy independence through renewables is just sound fiscal policy.</li>
<li>Perplexity is <a href="https://www.perplexity.ai/personal-computer-waitlist">building a Databox</a> equivalent to meet all the Claws, and <a href="https://www.docker.com/blog/run-nanoclaw-in-docker-shell-sandboxes/">Docker and Nanoclaw joined up</a> with sandboxes too. Noone's quite built the kind of temporal and spatially aware personal database we proposed <a href="https://anil.recoil.org/papers/2015-aarhus-databox">back in 2015</a> or <a href="https://anil.recoil.org/projects/perscon">earlier</a>. I guess I'm glad that my <a href="https://github.com/avsm/lifedb-plugins">OCaml LifeDB prototype</a> is now 18 years old and finally relevant.</li>
<li>OpenUK has issued its <a href="https://openuk.uk/press-releases-posts/openuk-provides-recommendations-for-the-uk-foundation-of-open-source/">recommendation to UKRI for a UK Foundation for Open Source</a> with a comprehensive report. I remain enthusiastic about the prospect of a variant of the <a href="https://anil.recoil.org/notes/uk-national-data-lib">UK National Data Library</a> and this is a good report!</li>
</ul><h1>References</h1><ul><li>Madhavapeddy (2026). Discussing effective conservation with all the UK Chief Scientists. <a href="https://doi.org/10.59350/qjrmv-38130" target="_blank"><i>10.59350/qjrmv-38130</i></a></li>
<li>Madhavapeddy (2025). Royal Society's Future of Scientific Publishing meeting. <a href="https://doi.org/10.59350/nmcab-py710" target="_blank"><i>10.59350/nmcab-py710</i></a></li>
<li>Madhavapeddy (2025). Holding an OxCaml tutorial at ICFP/SPLASH 2025. <a href="https://doi.org/10.59350/55bc5-x4p75" target="_blank"><i>10.59350/55bc5-x4p75</i></a></li>
<li>Madhavapeddy (2025). Thoughts on the National Data Library and private research data. <a href="https://doi.org/10.59350/fk6vy-5q841" target="_blank"><i>10.59350/fk6vy-5q841</i></a></li>
<li>Jaffer et al (2025). AI-assisted Living Evidence Databases for Conservation Science. Cambridge Open Engage. <a href="https://doi.org/10.33774/coe-2025-rmsqf" target="_blank"><i>10.33774/coe-2025-rmsqf</i></a></li>
<li>Madhavapeddy (2025). Publish, Review, Curate to upend scholarly publishing. <a href="https://doi.org/10.59350/fpc9w-ccj82" target="_blank"><i>10.59350/fpc9w-ccj82</i></a></li>
<li>Chaudhry et al (2015). Personal Data: Thinking Inside the Box. <a href="https://doi.org/10.7146/aahcc.v1i1.21312" target="_blank"><i>10.7146/aahcc.v1i1.21312</i></a></li>
<li>Ball et al (2026). Geospatial foundation models enable data-efficient tree species mapping in temperate mountain forests. bioRxiv. <a href="https://doi.org/10.64898/2026.02.23.707022" target="_blank"><i>10.64898/2026.02.23.707022</i></a></li>
<li>Madhavapeddy (2026). Streaming millions of TESSERA tiles over HTTP with Zarr v3. <a href="https://doi.org/10.59350/tk0er-ycs46" target="_blank"><i>10.59350/tk0er-ycs46</i></a></li>
<li>Madhavapeddy (2025). Using AT Proto for more than just Bluesky posts. <a href="https://doi.org/10.59350/32rdt-zny05" target="_blank"><i>10.59350/32rdt-zny05</i></a></li>
<li>Corley et al (2026). From Pixels to Patches: Pooling Strategies for Earth Embeddings. arXiv. <a href="https://doi.org/10.48550/arXiv.2603.02080" target="_blank"><i>10.48550/arXiv.2603.02080</i></a></li></ul>
