---
title: Streaming millions of TESSERA tiles over HTTP with Zarr v3
description: How we restructured TESSERA's geospatial embeddings from millions of
  individual numpy files into sharded Zarr v3 stores for efficient HTTP streaming,
  enabling everything from single-pixel mobile lookups to regional-scale analysis
  with just a couple of range requests.
url: https://anil.recoil.org/notes/tessera-zarr-v3-layout
date: 2026-03-14T00:00:00-00:00
preview_image: https://anil.recoil.org/images/tessera-zarr-stream-1.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>I've been working on making <a href="https://anil.recoil.org/projects/tessera">TESSERA</a> map embeddings even easier to retrieve,
so that we can build <a href="https://anil.recoil.org/notes/2026w10">dynamic user interfaces</a> in the browser or on mobile phones.</p>
<p>When we <a href="https://anil.recoil.org/notes/geotessera-python">first released</a> GeoTessera last year, every 0.1°
tile was a pair of <a href="https://numpy.org/">numpy</a> files; one quantized embedding and one scale
array.  That worked fine for grabbing a few tiles at a time, but our
<a href="https://anil.recoil.org/notes/geotessera-python">push for global coverage from 2017-2025</a> is producing around 1.8 million
tiles per year, each weighing in at around 150MB!  Serving these over HTTP means a
million small directories on disk, and every client that wants a contiguous
region needs to discover, fetch and stitch dozens of them, and has a minimum download of 150MB.</p>
<p>To fix this, we need to rethink the structure of the storage entirely: it's quite tricky to support
both a small download (e.g. from a mobile phone) and also a large region from a cloud provider.
Luckily, there's a new cloud-native streaming format in town that's just the ticket, known
as <a href="https://zarr.dev">Zarr</a>.
Since the GeoTESSERA <a href="https://anil.recoil.org/notes/geotessera-python-0-7">0.7 release</a> where we first added basic
Zarr support, I've been working on consolidating all our tiles into a
single sharded Zarr v3 store per year.</p>
<p></p><div class="video-center"><iframe title="Tessera Zarr streaming preview" width="100%" height="315px" src="https://crank.recoil.org/videos/embed/08aafc87-9aea-48e3-8c41-a2fe1b94fea4" frameborder="0" allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups allow-forms"></iframe></div><p></p>
<p>This post explains the <strong><a href="https://github.com/ucam-eo/zarr-convention-tessera">TESSERA Zarr conventions proposal</a></strong> and why the
chunking size choices matter.  I'd also love to get feedback from experienced
geospatial gurus, so this post is also an RFC of sorts.</p>
<h2><a href="https://anil.recoil.org/news.xml#why-zarr-v3" class="anchor" aria-hidden="true"></a>Why Zarr v3?</h2>
<p><a href="https://zarr.dev/">Zarr</a> is a format for large N-dimensional typed arrays
designed for cloud object stores.  It's great because it allows multidimensional
arrays to be accessed via HTTP, meaning that normal S3 or HTTP static servers are sufficient
for hosting large datasets.</p>
<p>I built a first prototype a few <a href="https://anil.recoil.org/notes/2026w9">weeks ago</a> using Zarr v2, and mapped the existing
npy tile format we use to it. This collects up batches of 10m2 pixel embeddings into
larger tiles, which can be downloaded as a unit (of around 150MB each).
The <a href="https://zarr-specs.readthedocs.io/en/latest/v3/core/v3.0.html">v3 specification</a>
(released last year) brings a couple of important new features to improve this:</p>
<ul>
<li>
<p><strong><a href="https://zarr-specs.readthedocs.io/en/latest/v3/codecs/sharding-indexed/index.html">Sharding</a></strong> a single physical file can contain many logical chunks,
indexed by an inline index.  This means a client can issue one <a href="https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Range_requests">HTTP range request</a>
to get the shard index, then a second byte range to get exactly the
chunk it needs.  Without sharding, every logical chunk would be a separate file,
and so reducing our minimum pixel size to save on downloads for small ROIs (e.g. for mobile devices)
would be impractical and involve 100s of millions of tiny files.</p>
</li>
<li>
<p><strong><a href="https://zarr-specs.readthedocs.io/en/latest/v3/codecs/index.html">Codecs</a></strong> formalise the chain of compression, transposition
and serialisation applied to each chunk.  Sharding is one such codec, and we also use <a href="https://zarr-specs.readthedocs.io/en/latest/v3/codecs/blosc/index.html">Blosc</a>/<a href="https://github.com/facebook/zstd">Zstd</a> for all arrays, which gives us reasonable compression ratios on the int8 embeddings. We're never going to get amazing compression ratios of the TESSERA embeddings because they are high entropy (we reduce 1000s of dimensions into 128 during the training and inference process), but there's still some win to be had.</p>
</li>
</ul>
<p>The Python <a href="https://zarr.readthedocs.io/">zarr</a> library has reasonably solid v3 support now, and in my tests the wider ecosystem
such as <a href="https://xarray.dev/">xarray</a>, <a href="https://dask.org/">dask</a>, <a href="https://corteva.github.io/rioxarray/">rioxarray</a> can all read these
v3 stores without issue. So I think we're good to use v3 features now!</p>
<h2><a href="https://anil.recoil.org/news.xml#the-store-layout" class="anchor" aria-hidden="true"></a>The store layout</h2>
<p>Each embeddings year gets a single Zarr store.  Within it, each <a href="https://en.wikipedia.org/wiki/Universal_Transverse_Mercator_coordinate_system">UTM zone</a> is a <a href="https://zarr.readthedocs.io/en/latest/user-guide/groups/">group</a> that contains that particular strip of the planet's embeddings and scales. For clients to visualise what's going on, there's an optional global RGB preview group:</p>
<pre><code>2024.zarr/
    zarr.json                 # root: version, year, conventions
    utm29/                    # one group per UTM zone
        embeddings            # int8    (H, W, 128)
        scales                # float32 (H, W)
        rgb                   # uint8   (H, W, 4)     [optional]
        easting               # float64 (W,)
        northing              # float64 (H,)
        band                  # int32   (128,)
    utm30/
        ...
    global_rgb/               # EPSG:4326 preview pyramid
        0/rgb                 # uint8   (H, W, 4)     level 0
        1/rgb                 #                       level 1
        2/rgb                 #                       level 2
        ...
</code></pre>
<p>Using <strong>one store per year</strong> rather than per zone allows us to use an experimental
<a href="https://zarr.readthedocs.io/en/latest/user-guide/consolidated_metadata/">consolidated metadata</a> feature.
A single <code>zarr.consolidate_metadata()</code> call gives a client the full catalog of
zones, their spatial extents, and which arrays exist. This (I think) eliminates
the need for the <a href="https://geotessera.readthedocs.io/en/latest/geotessera.html#geotessera-registry-registry-management">Parquet registry</a>
we currently maintain for TESSERA.</p>
<p>Like the current npy embeddings, each <strong>zone group carries their own CRS</strong> to minimise
coordinate skew.  Each zone has a <code>proj:code</code> attribute
(e.g. <code>EPSG:32630</code>) and a <code>spatial:transform</code> giving the affine matrix.
Southern hemisphere zones use the canonical northern-hemisphere EPSG code
with the 10,000,000m false northing subtracted, so the northing axis is
continuous.</p>
<p><strong>Coordinate arrays</strong> (<code>easting</code>, <code>northing</code>, <code>band</code>) are small 1-D arrays
stored alongside the data, so an xarray <code>open_zarr</code> just works with labeled
axes. These are labeled as <a href="https://docs.xarray.dev/en/stable/internals/zarr-encoding-spec.html#dimension-encoding-in-zarr-formats">dimension names</a>
so that xarray or other clients can pick them up automatically.</p>
<h2><a href="https://anil.recoil.org/news.xml#sharding-and-chunking" class="anchor" aria-hidden="true"></a>Sharding and chunking</h2>
<p>The TESSERA embeddings are quite large in aggregate, and that is where most of
the design time went.  TESSERA clients have three very different access patterns:</p>
<ol>
<li>A single-pixel lookup or a small region-of-interest means that a user has a lon/lat and wants the 128-d
embedding vector at that point.  This should be ~2KB over HTTP. This might be a mobile device aiming to do active learning, for example.</li>
<li>A regional subset means that a user wants a spatial rectangle (say, 100km2)
of all 128 bands.  This should stream efficiently without reading the
whole zone and mosaicing it in memory (a source of <a href="https://github.com/ucam-eo/geotessera/issues/117">memory problems</a> currently). This might be a desktop analysis, or even a <a href="https://gazagnaire.org/blog/2026-02-25-satellite-software.html">satellite scanning a region</a>.</li>
<li>A scan of entire countries to do global analyses, which requires terabytes of downloads to retrieve the full set of embeddings.</li>
</ol>
<p>We will come back to solve the third 'entire countries' problem later, via a new variant of the model we are training that uses <a href="https://huggingface.co/blog/matryoshka">Matryoshka embeddings</a>. However, the first two also pull in opposite directions but are needed for mobile clients vs chunkier desktop analysis tools.
Zarr v3 sharding resolves both by letting us create shards:</p>
<ul>
<li>Shard: 256 × 256 pixels  (aligned to tile boundaries)</li>
<li>Chunk: 4 × 4 pixels  (inner chunk within each shard)</li>
</ul>
<p>Each shard is a single file on disk or object in S3, containing a grid of
64×64 inner chunks plus a ~32KB shard index at the end.  To read a single
pixel the client has to:</p>
<ol>
<li>Fetch the shard index via one HTTP range request of ~32KB, which is cacheable.</li>
<li>Compute which 4×4 inner chunk contains the pixel.</li>
<li>Fetch that chunk with another HTTP range request that's ~2KB for int8×128, and can reuse the previous HTTP connection via pipelining.</li>
</ol>
<p>For a single pixel read, there's a bit of extra overhead from the index, but tolerable.
For a regional read, the client fetches whole shards and gets contiguous
256×256 blocks, which is efficient for downstream processing.</p>
<p>Another neat thing about Zarr is that we can have multiple data type arrays.
TESSERA uses a quantisation trick to compress the embeddings with a 'scale'
array, which is a float32 held alongside the 128-dimension int8 values. For this array we also use the same 256/4 sharding,
and also signal that there's no data via a NaN scale. This lets us skip the need for the <a href="https://dl2.geotessera.org/v1/global_0.1_degree_tiff_all/">landmask TIFFs</a> we currently maintain.</p>
<p>After this the global RGB preview is plain sailing as its more like a
conventional visual map tile, and uses plain 512×512 chunks with no sharding
since the pyramid levels get small quickly and the access pattern is always
tile-aligned for map rendering. These previews can also be reprojected by the
client for dynamic maps.</p>
<h2><a href="https://anil.recoil.org/news.xml#geozarr-conventions" class="anchor" aria-hidden="true"></a>GeoZarr conventions</h2>
<p>The <a href="https://github.com/zarr-developers/geozarr-spec">GeoZarr spec</a> is still under active development, and has a <a href="https://github.com/zarr-developers/zarr-specs/blob/main/docs/v3/conventions.rst">conventions mechanism</a> where stores declare which metadata schemas they follow.  We use three:</p>
<ul>
<li>
<p><strong><a href="https://github.com/zarr-conventions/geo-proj">proj:</a></strong> is the CRS
information formerly held in our landmask TIFFs.  Each zone group carries a <code>proj:code</code> (e.g. <code>"EPSG:32630"</code>)
and <code>proj:wkt2</code> for the full WKT2 string.</p>
</li>
<li>
<p><strong><a href="https://github.com/zarr-conventions/spatial">spatial:</a></strong> is the affine spatial
coordinate transforms.  Each zone group has <code>spatial:transform</code> (the 6-element
affine), <code>spatial:dimensions</code>, <code>spatial:shape</code>, <code>spatial:bbox</code> and
<code>spatial:registration</code>. This can be calculated fairly easily from the CRS, but included here
so that clients that know the spatial Zarr convention can just query this and use it directly.</p>
</li>
<li>
<p><strong><a href="https://github.com/zarr-conventions/multiscales">multiscales</a></strong> is the
pyramid layout for the global preview, compatible with the approach used by
<a href="https://github.com/developmentseed/topozarr">topozarr</a>.</p>
</li>
</ul>
<p>These conventions are registered in the root <a href="https://dl2.geotessera.org/zarr/v1/2024.zarr/zarr.json">zarr.json</a> attributes as an
array, following the <a href="https://zarr.dev/zeps/draft/ZEP0009.html">ZEP for conventions</a>.  This makes
the stores more self-describing as any Zarr-aware tool can read the conventions
list and know what metadata keys to expect.</p>
<p>In order to join the Zarr specification party, I've created <strong><a href="https://github.com/ucam-eo/zarr-convention-tessera">zarr-convention-tessera</a></strong>
to crystallise the conventions I've used in TESSERA, such as the utm zone splitting and quantisation bands. Once we're happy with this
format, existing libraries like <a href="https://geotessera.readthedocs.io">geotessera</a> and also the upcoming <a href="https://jon.recoil.org/notebooks/interactive_map.html">OCaml geotessera</a>
can all switch to Zarr streaming instead.</p>
<p><a href="https://tze.geotessera.org"> <img src="https://anil.recoil.org/images/tessera-zarr-stream-1.webp" alt="%c" title="The first TESSERA zarr prototype showing the multiscale pyramid. The vertical lines in the map are debug markers to delimit UTM zones. Conveniently Cambridge is split right down the middle of two!"> </a></p>
<h2><a href="https://anil.recoil.org/news.xml#building-the-tessera-zarr-stores" class="anchor" aria-hidden="true"></a>Building the TESSERA Zarr stores</h2>
<p>The <code>geotessera-registry</code> CLI now has <a href="https://github.com/ucam-eo/geotessera/pull/211">commands in development</a> for this pipeline.
It's unfortunately a very computationally heavy job, since the input npy tiles have to be rearranged and rewritten one by one into the
Zarr format, and then the RGB pyramids calculated. This is reasonable to parallelise, but we're a little stuck on our university
storage cluster due to relatively slow network interconnects at the moment.</p>
<p>Figuring out this conversion bottleneck is top of my list next week; in particular, if you have any leads on a cloud storage
provider that may like to sponsor a petabyte or two of S3 storage, I'm all ears!</p>
<p>One other useful thing is that we generate a STAC catalog that provides a standards-compliant discovery layer: one
<a href="https://stacspec.org/">STAC</a> collection per year. This lets us use <a href="https://www.tunbury.org/2025/12/02/tessera-stac/">tile servers</a> and
hopefully eventually <a href="https://doi.org/10.1145/3759536.3763803">STAC-D pipelines</a> to integrate these into <a href="https://anil.recoil.org/projects/plancomp">planetary computing</a> pipelines.</p>
<h2><a href="https://anil.recoil.org/news.xml#whats-next" class="anchor" aria-hidden="true"></a>What's next</h2>
<p>We won't stop serving the npy files for some time, since we have a number of users already committed to those, and that workflow
is fine for regional analysis. However, I'm keen to unlock mobile workflows as there's a lot of demand for this (especially after the <a href="https://anil.recoil.org/notes/first-tessera-hackathon">TESSERA hackathon in Delhi</a>), so we'll push
forward with Zarr. In particular thank you to <a href="https://cherian.net/">Deepak Cherian</a> for giving me lots of Zarr advice on our <a href="https://eeg.zulipchat.com/#narrow/channel/527258-Tessera/topic/zarr.20file.20format/with/578418959">Zulip channel</a>.</p>
<p>While this spec is out for review, here's a sneak peek of <a href="https://tze.geotessera.org">a TESSERA Zarr web viewer</a> that reads directly from the Zarr stores into the browser, with no server required. I'm also working on an access library in OxCaml using <a href="https://www.tunbury.org/">Mark Elvers</a> <a href="https://github.com/mtelvers/ocaml-zarr">OCaml Zarr</a> library so that we can use these from our native pipeline too. This would also make it much easier to integrate TESSERA into the <a href="https://anil.recoil.org/notes/nas-rs-biodiversity-papers">biodiversity monitoring standards framework</a> that we've been working on.</p>
<p>I also discovered that there had been a very relevant <a href="https://www.clarku.edu/news/2026/03/12/sprinting-to-space-goddard-nasa-and-clarks-pathbreaking-work-in-geospatial-analytics/">vector embeddings hackathon</a> held a few days ago at Clark University. They <a href="https://www.linkedin.com/feed/update/urn:li:ugcPost:7438619750936539136">came up</a> with a <a href="https://github.com/geo-embeddings/embeddings-stac-specification">STAC for embeddings proposal</a> that I've left an <a href="https://github.com/geo-embeddings/embeddings-stac-specification/issues/9">query on</a> as well, to make sure our work is compatible.</p><h1>References</h1><ul><li>Madhavapeddy (2026). Connecting the dots for biodiversity action from the NAS/Royal Society Forum. <a href="https://doi.org/10.59350/dy7d3-hdt43" target="_blank"><i>10.59350/dy7d3-hdt43</i></a></li>
<li>Madhavapeddy (2026). .plan-26-10: Streaming TESSERA working, biodiversity action papers, and FPL takes off. <a href="https://doi.org/10.59350/re0zy-3rt26" target="_blank"><i>10.59350/re0zy-3rt26</i></a></li>
<li>Madhavapeddy (2026). 1st TESSERA/CoRE hackathon at the Indian AI Summit. <a href="https://doi.org/10.59350/1na80-7ak85" target="_blank"><i>10.59350/1na80-7ak85</i></a></li>
<li>Madhavapeddy (2025). GeoTessera 0.7 out with efficient sampling and Zarr support. <a href="https://doi.org/10.59350/nagwp-tnw89" target="_blank"><i>10.59350/nagwp-tnw89</i></a></li>
<li>Madhavapeddy (2025). GeoTessera Python library released for geospatial embeddings. <a href="https://doi.org/10.59350/7hy6m-1rq76" target="_blank"><i>10.59350/7hy6m-1rq76</i></a></li>
<li>Laud et al (2025). STACD: STAC Extension with DAGs for Geospatial Data and Algorithm Management. <a href="https://doi.org/10.1145/3759536.3763803" target="_blank"><i>10.1145/3759536.3763803</i></a></li></ul>
