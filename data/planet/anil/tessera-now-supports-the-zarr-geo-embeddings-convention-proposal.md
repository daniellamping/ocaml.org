---
title: TESSERA now supports the Zarr geo-embeddings convention proposal
description: "Community feedback reshaped our Zarr store layout \u2014 years became
  a dimension, shards got bigger, and we retired the TESSERA-specific convention in
  favour of a shared geo-embeddings standard that also covers other models."
url: https://anil.recoil.org/notes/tessera-embeddings-convention
date: 2026-03-27T00:00:00-00:00
preview_image: https://anil.recoil.org/images/videos/e5b98a04-9f08-446b-9cfd-b15d8d441d5e.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>The ink's barely dry on my last <a href="https://en.wikipedia.org/wiki/Request_for_Comments">RFC</a> post about <a href="https://anil.recoil.org/notes/tessera-zarr-v3-layout">streaming millions of TESSERA tiles over HTTP</a> using Zarr v3 sharding, and it's out of date! The store layout I'd carefully craftd lasted about 48 hours before <a href="https://github.com/ucam-eo/zarr-convention-tessera/issues/1">a discussion</a> from <a href="https://github.com/geospatial-jeff">Jeff Albrecht</a> and <a href="https://github.com/ljstrnadiii">Len Strnad</a> convinced me to rethink my approach.</p>
<p>This led me to build a "MegaZarr" unified store, and retire our TESSERA-specific convention to instead integrate with a shared <a href="https://github.com/geo-embeddings/embeddings-zarr-convention">geo-embeddings Zarr convention</a> that's appeared in the last week.  Things are moving <em>really</em> quickly in the world of geospatial embeddings; I'll explain some of the gory details about the store here, and here's a video to show it in action or <strong><a href="https://tze.geotessera.org">try it for yourself</a></strong>:</p>
<p></p><div class="video-center"><iframe title="TESSERA Zarr v3 streaming (take 2)" width="100%" height="315px" src="https://crank.recoil.org/videos/embed/e5b98a04-9f08-446b-9cfd-b15d8d441d5e" frameborder="0" allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups allow-forms"></iframe></div><p></p>
<h2><a href="https://anil.recoil.org/news.xml#arise-the-megazarrrrr" class="anchor" aria-hidden="true"></a>Arise, the MegaZarrrrr</h2>
<p>The <a href="https://anil.recoil.org/notes/tessera-zarr-v3-layout">original layout</a> had a separate Zarr store per year, an HWB array layout, and 256-pixel shards with very fine tile granularity. After the discussions, instead of a Zarr store per year, we now have a <strong>single store with time as a first-class dimension</strong>, which I dubbed <a href="https://www.linkedin.com/feed/update/urn:li:activity:7442572588876439552?commentUrn=urn:li:comment:(activity:7442572588876439552%2C7442624868975804416)&amp;dashCommentUrn=urn:li:fsd_comment:(7442624868975804416%2Curn:li:activity:7442572588876439552)">the MegaZarr</a>. The unified array shape is now <code>(T, B, H, W)</code> or <a href="https://stackoverflow.com/questions/67087131/what-is-nchw-format">NCHW ordering</a> which is easier to use in ML frameworks like PyTorch. Each year's data lives in its own temporal slice, and the <code>time</code> coordinate array stores the years explicitly so it can just be indexed by the integer year directly.</p>
<p>You can see this in the video above in the new 'explorer' mode, whereby you can just go to <a href="https://tze.geotessera.org">https://tze.geotessera.org</a> and click around individual tiles and 10m² pixels and see the embeddings all the way from 2017 to 2025 where available.  We can't easily prepend years to an existing store without rewriting the whole thing, so we zero-fill the entire year range upfront and selectively replace years as they're generated. We use dual sentinels in the scales array to do this via <code>+inf</code> for "land but no data yet" and <code>NaN</code> for water areas that don't have valid embeddings.
This lets a single <code>isfinite()</code> call identify valid terrestrial pixels in a tile.</p>
<p>Switching from HWB to NCHW format also aligns with how ML frameworks expect data since all 128 bands for a pixel are contiguous in a single inner chunk, which is what downstream tasks usually need. The common operation of grabbing a single year is fine since the time dimension coming first means temporal slicing is cheap.</p>
<p>I've also tweaked the shard/chunk balancing sizes to a larger 4096×4096 shard size vs the original 256×256, and a slightly larger inner chunk of (1,128,32,32). There are still only two HTTP range requests for a single pixel, but now each shard covers a larger (around 40km²) area with far fewer files on the server. The <a href="https://tze.geotessera.org">TZE explorer</a> confirms this still works reasonably well on mobile.</p>
<p><img src="https://anil.recoil.org/images/ssds-by-the-dozen.webp" alt="%c" title="Mark Elvers is installing SSDs today by the bushel so we can keep up with all the storage requirements here. We do need cloud hosting for all this Zarr as we're over a petabyte raw now in the University and it's nervous work..."></p>
<h2><a href="https://anil.recoil.org/news.xml#new-geotessera-tooling-around-zarr" class="anchor" aria-hidden="true"></a>New geotessera tooling around Zarr</h2>
<p>All this also means that all my hard work on <a href="https://anil.recoil.org/notes/geotessera-python">geotessera</a> will also soon be obsolete! A quickstart to grab embeddings using <em>just</em> xarray/Zarr from Python looks like</p>
<pre><code># params, year (int), x, y
ds = xr.open_zarr(args.store, group=f"utm{zone:02d}",
       zarr_format=3, consolidated=True,
       chunks={"time": 1, "band": 128, "y": 4096, "x": 4096})
pixel = ds.sel(time=year, x, y, method="nearest")
emb_int8 = pixel["embeddings"].values
scale = float(pixel["scales"].values)
embedding = emb_int8.astype(np.float32) * scale
for i in range(min(16, len(embedding))):
  print(f"{i:4d}  {int(emb_int8[i]):5d}  {embedding[i]:10.6f}")
</code></pre>
<p>A few things of note here are that the use of coordinate arrays means that you can index into the Zarr array using the natural year integer and the UTM x/y coordinates.  After that you dequantise using the scales array, and the dimensions are in an array for you.</p>
<p>Now, there is still some useful housekeeping whereby Geotessera will continue to be useful for Zarr. Firstly, it can provide coordinate lookups and also a name resolver to find a copy of the embeddings (we are getting offers to mirror and version from several other people, which is great). Secondly, the biggest downside to Zarr is that it doesn't have a quick index of sparsity, which means that coverage tests can be expensive in terms of HTTP HEAD requests. So I suspect I'll keep the existing GeoParquet tile registry around to provide a quick lookup for clients that want to test large-scale tile existence. However, downloading this index will be optional so clients can get started immediately without an expensive database check.</p>
<p>Using the new wrapper (which I'm tidying up for a release) means you just need to:</p>
<pre><code class="language-python">from geotessera.store import GeoTesseraZarr

gt = GeoTesseraZarr()  # defaults to public store
emb = gt.sample_at(-2.97, 53.44, year=2025)          # (128,) float32
mosaic, transform = gt.read_region(bbox, year=2025)  # (H, W, 128)
</code></pre>
<h2><a href="https://anil.recoil.org/news.xml#from-a-tessera-convention-to-a-community-one" class="anchor" aria-hidden="true"></a>From a TESSERA convention to a community one</h2>
<p>While I was iterating on the store layout, a <a href="https://anil.recoil.org/notes/2026w12">Clark University hackathon</a> on geo-embeddings created an <a href="https://github.com/geo-embeddings/embeddings-stac-specification">embeddings-stac-specification</a> repository to connect STAC with vector embeddings. We had a <a href="https://github.com/geo-embeddings/embeddings-stac-specification/issues/9">productive discussion</a> about how TESSERA's QAT-compressed embeddings differ from other models.</p>
<p>There's also another <a href="https://github.com/geo-embeddings/embeddings-zarr-convention">geo-embeddings Zarr convention</a>, which is a model-agnostic standard for describing geospatial embeddings in Zarr stores. It currently has examples for <a href="https://clay.earth">Clay</a> (768-dim chip embeddings on a regular grid) and <a href="https://developers.google.com/earth-engine">AEF</a> (64-dim patch embeddings with global quantisation).</p>
<p>To make TESSERA fit into this, I <a href="https://github.com/geo-embeddings/embeddings-zarr-convention/compare/add-tessera">proposed several additions</a> to the convention:</p>
<ol>
<li>
<p>Per-pixel scale quantization (<code>"method": "per_pixel_scale"</code>). Unlike the other models, TESSERA uses <a href="https://anil.recoil.org/papers/2025-tessera">quantization-aware training</a> with a per-pixel scale array, and not a single global scale. The convention therefore now supports a <code>scale_array</code> field pointing to the array name and a <code>nodata</code> field for sentinel values.</p>
</li>
<li>
<p>UTM zone spatial layout (<code>"spatial_layout": "utm_zones"</code>). In order to minimise distortion due to varying pixel sizes at different points of the earth, TESSERA stores each UTM zone as a separate group rather than reprojecting to a global grid. This is a bit less seamless than the simpler <code>global</code> layout, but prioritises accuracy until we can measure the effect of the distortion. We use <a href="https://zarr.readthedocs.io/en/latest/user-guide/groups/">Zarr groups</a> for this, so the <a href="https://dl2.geotessera.org/zarr/v2/store.zarr/">root store</a> has subgroups with utmXY that the client opens as needed.</p>
</li>
<li>
<p>Multiple source datasets. TESSERA fuses Sentinel-1 (radar) and Sentinel-2 (optical), so <code>geoemb:source_data</code> now accepts an array of URLs rather than just one.</p>
</li>
<li>
<p>Build version tracking. We're going to be releasing multiple variations and models of our embeddings as the technology advances, so <code>geoemb:build_version</code> also records which version of the pipeline produced the store. I'm hoping to use this to record <a href="https://anil.recoil.org/notes/principles-for-collective-knowledge">end-to-end provenance</a> for every single embedding.</p>
</li>
</ol>
<p>See the <a href="https://github.com/avsm/embeddings-zarr-convention/blob/beb2da66d68eebf42a2bbef928bb7cde181570ba/examples/tessera_example.json">TESSERA example</a> JSON for all this in one place.
With the <code>geoemb:</code> convention covering everything we need, the <a href="https://github.com/ucam-eo/zarr-convention-tessera">zarr-convention-tessera</a> repo can now be retired after a glorious two weeks of life. The <a href="https://inspect.geozarr.org/?url=https://dl2.geotessera.org/zarr/v2/store.zarr/">Zarr web inspector</a> confirms our live store is compliant, except for those in-flight proposals that still need to be ratified.</p>
<h2><a href="https://anil.recoil.org/news.xml#whats-next" class="anchor" aria-hidden="true"></a>What's next</h2>
<p>Thanks to everyone in the Zarr community for their really useful input. I wouldn't be surprised if we need to tweak my (newbie) layout a bit more yet, but comments are welcome <a href="https://github.com/geo-embeddings/embeddings-zarr-convention/pull/2">on my PR</a>. Once things settle, I'm back to working on the <a href="https://anil.recoil.org/notes/2026w9">OxCaml Zarr library</a> to add the sharding support I need.</p>
<p>Also to throw a bit of a clanker into my delicate Zarr work, we're working on a revision of the TESSERA model that supports <a href="https://arxiv.org/abs/2205.13147">Matryoshka representations</a>, which means I need to figure out how to support slicing by dimension as well. After all this careful work in v1.0 to make the embeddings contiguous in memory, we will soon need to be able to get just the first few dimensions out of 128 to do quick sample analyses. Code doesn't survive very long any more with all the progress in training!</p>
<p>The live Zarr v3 TESSERA store is browsable now at the <a href="https://tze.geotessera.org">TZE explorer</a>, and the embeddings convention is open for feedback at <a href="https://github.com/geo-embeddings/embeddings-zarr-convention">geo-embeddings/embeddings-zarr-convention</a>.</p><h1>References</h1><ul><li>Feng et al (2025). TESSERA: Temporal Embeddings of Surface Spectra for Earth Representation and Analysis. arXiv. <a href="https://doi.org/10.48550/arXiv.2506.20380" target="_blank"><i>10.48550/arXiv.2506.20380</i></a></li>
<li>Madhavapeddy (2026). Streaming millions of TESSERA tiles over HTTP with Zarr v3. <a href="https://doi.org/10.59350/tk0er-ycs46" target="_blank"><i>10.59350/tk0er-ycs46</i></a></li>
<li>Madhavapeddy (2025). Four Ps for Building Massive Collective Knowledge Systems. <a href="https://doi.org/10.59350/418q4-gng78" target="_blank"><i>10.59350/418q4-gng78</i></a></li>
<li>Madhavapeddy (2025). GeoTessera Python library released for geospatial embeddings. <a href="https://doi.org/10.59350/7hy6m-1rq76" target="_blank"><i>10.59350/7hy6m-1rq76</i></a></li>
<li>Kusupati et al (2024). Matryoshka Representation Learning. arXiv. <a href="https://doi.org/10.48550/arXiv.2205.13147" target="_blank"><i>10.48550/arXiv.2205.13147</i></a></li></ul>
