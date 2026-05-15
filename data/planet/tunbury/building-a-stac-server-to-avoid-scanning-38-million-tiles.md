---
title: Building a STAC server to avoid scanning 3.8 million tiles
description: The GeoTessera project produces 128-channel geospatial embeddings from
  Sentinel satellite imagery. The dataset is tiled at 0.1-degree resolution across
  the globe, covering 9 years and comprising roughly 3.8 million tiles, each containing
  embeddings and scale-factor files.
url: https://www.tunbury.org/2026/04/17/geotessera-stac/
date: 2026-04-17T16:30:00-00:00
preview_image: https://www.tunbury.org/images/tessera-globe.png
authors:
- Mark Elvers
source:
ignore:
---

<p>The <a href="https://geotessera.org">GeoTessera</a> project produces 128-channel geospatial embeddings from Sentinel satellite imagery. The dataset is tiled at 0.1-degree resolution across the globe, covering 9 years and comprising roughly 3.8 million tiles, each containing embeddings and scale-factor files.</p>

<p>These tiles live on three storage backends: the primary source on <code class="language-plaintext highlighter-rouge">okavango</code> here in Cambridge (ZFS over spinning disks), an S3 bucket in AWS us-west-2, and a CephFS cluster in Scaleway Paris. Keeping them in sync was becoming slow due to the continual scanning of the source and target.</p>

<p>The <code class="language-plaintext highlighter-rouge">s5cmd sync</code> or <code class="language-plaintext highlighter-rouge">rsync</code>/<code class="language-plaintext highlighter-rouge">rclone</code> approach works, but they start by listing every file on both sides to compute the diff. With 3.8 million tile directories, each containing 3 files, that scan takes a very long time.</p>

<p>What I wanted was an index that tracked what each store contained so that the sync could be reduced to a set difference on metadata rather than a filesystem walk.</p>

<h1>The existing registry</h1>

<p>There is already <code class="language-plaintext highlighter-rouge">registry.parquet</code> which lists every tile on <code class="language-plaintext highlighter-rouge">okavango</code> with coordinates, year, file sizes, and hashes. For the target stores, I needed an equivalent parquet file per store that records which tiles it has.</p>

<p>Initially, the sync tool reads the content of a remote store from an <code class="language-plaintext highlighter-rouge">s5cmd ls</code> or <code class="language-plaintext highlighter-rouge">find</code> output and builds the parquet manifest. From then on, diffs are fast:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>=== GeoTessera Sync Status ===

Registry: 3831542 tiles across 9 year(s)

Stores:
  okavango        3831542 tiles
  s3              3831566 tiles
  scaleway        3822382 tiles

Pairwise diffs (missing from target):
  okavango -&gt; scaleway: 9184 missing
</code></pre></div></div>

<p>Copying the missing tiles then becomes a targeted operation where I can pipe the manifest into <code class="language-plaintext highlighter-rouge">xargs -P 32</code> with <code class="language-plaintext highlighter-rouge">s5cmd cp</code>, rather than letting sync discover what’s missing by scanning everything.</p>

<h1>Fixing the Arrow library</h1>

<p>The tool is written in OCaml using <a href="https://github.com/mtelvers/arrow">mtelvers/arrow</a> for parquet I/O. The upstream <code class="language-plaintext highlighter-rouge">registry.parquet</code> uses the <code class="language-plaintext highlighter-rouge">large_string</code> Arrow type (int64 offsets) for its hash column, which the OCaml bindings didn’t support. They only handled regular <code class="language-plaintext highlighter-rouge">utf8</code> (int32 offsets). Reading the column would silently pass the C++ type check (thanks to a special-case hack) but then crash when the OCaml code tried to interpret int64 offsets as int32.</p>

<p>The <a href="https://github.com/mtelvers/arrow/commit/c7db370">fix</a> added first-class <code class="language-plaintext highlighter-rouge">LargeUtf8</code> support across the library: new <code class="language-plaintext highlighter-rouge">read_large_utf8</code> / <code class="language-plaintext highlighter-rouge">read_large_utf8_opt</code> reader functions with int64 offset handling, <code class="language-plaintext highlighter-rouge">large_utf8</code> / <code class="language-plaintext highlighter-rouge">large_utf8_opt</code> writer functions, a <code class="language-plaintext highlighter-rouge">LargeUtf8</code> variant in the high-level <code class="language-plaintext highlighter-rouge">Table.col_type</code> GADT, and updates to <code class="language-plaintext highlighter-rouge">fast_read</code> for automatic type detection. The silent special case in the C++ layer was removed in favour of proper type dispatch. The library was also bumped from C++17 to C++20 to support Arrow 23 headers.</p>

<h1>I didn’t need a STAC server</h1>

<p><a href="https://stacspec.org">STAC</a> (SpatioTemporal Asset Catalogue) is a standard for describing geospatial data. The sync tool doesn’t use it. Previously, I created <a href="https://github.com/mtelvers/tile-server">mtelvers/tile-server</a>, which served as the basis for this project. It works directly with parquet files. But since we had all the tile metadata loaded anyway, wrapping it in a STAC API was straightforward and gives us:</p>

<ul>
  <li>A standard API that tools like <a href="https://pystac.readthedocs.io/">pystac</a>, QGIS, and STAC browsers can query</li>
  <li>Per-tile asset links showing which stores have each tile and where to download it</li>
  <li>Spatial search by bounding box</li>
</ul>

<p>The server loads the parquet files at startup, builds an in-memory index, and serves STAC-compliant JSON. The first store listed is the primary (its tiles form the catalogue); others are cross-referenced to populate asset links.</p>

<div class="language-json highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">{</span><span class="w">
  </span><span class="nl">"id"</span><span class="p">:</span><span class="w"> </span><span class="s2">"2024_grid_0.85_49.95"</span><span class="p">,</span><span class="w">
  </span><span class="nl">"assets"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w">
    </span><span class="nl">"okavango"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w">
      </span><span class="nl">"href"</span><span class="p">:</span><span class="w"> </span><span class="s2">"https://dl2.geotessera.org/.../2024/grid_0.85_49.95"</span><span class="p">,</span><span class="w">
      </span><span class="nl">"file:size"</span><span class="p">:</span><span class="w"> </span><span class="mi">108527232</span><span class="p">,</span><span class="w">
      </span><span class="nl">"file:checksum"</span><span class="p">:</span><span class="w"> </span><span class="s2">"sha256:..."</span><span class="w">
    </span><span class="p">},</span><span class="w">
    </span><span class="nl">"s3"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w">
      </span><span class="nl">"href"</span><span class="p">:</span><span class="w"> </span><span class="s2">"https://tessera-embeddings.s3.us-west-2.amazonaws.com/.../2024/grid_0.85_49.95"</span><span class="w">
    </span><span class="p">},</span><span class="w">
    </span><span class="nl">"scaleway"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w">
      </span><span class="nl">"href"</span><span class="p">:</span><span class="w"> </span><span class="s2">"https://dl1.scw.geotessera.org/.../2024/grid_0.85_49.95"</span><span class="w">
    </span><span class="p">}</span><span class="w">
  </span><span class="p">}</span><span class="w">
</span><span class="p">}</span><span class="w">
</span></code></pre></div></div>

<h1>Map envy</h1>

<p>The real motivation for the frontend was seeing the <a href="https://geotessera.org/coverage">GeoTessera coverage map</a>. It’s a beautiful visualisation of global tile coverage, and I felt a bit left out with plain data tables. Using the <a href="https://maplibre.org/">MapLibre GL</a> frontend on top of the STAC API, with a Sentinel-2 satellite basemap, you can browse the tile inventory spatially, inspect per-tile metadata and store locations, and more.</p>

<p>It’s live at <a href="https://stac.mint.caelum.ci.dev">stac.mint.caelum.ci.dev</a>.</p>

<h1>The stack</h1>

<p>The project is two OCaml binaries. Firstly, <code class="language-plaintext highlighter-rouge">stac-server</code>, which handles the STAC API using the parquet files, and secondly, <code class="language-plaintext highlighter-rouge">stac-sync</code> for CLI scanning stores, diffing manifests, generating copy lists, and recording synced tiles</p>

<p>Caddy sits in front as a reverse proxy, serving the static frontend at <code class="language-plaintext highlighter-rouge">/</code> and proxying <code class="language-plaintext highlighter-rouge">/api/*</code> to the OCaml server.</p>

<p>The source is at <a href="https://github.com/mtelvers/stac-server">mtelvers/stac-server</a></p>
