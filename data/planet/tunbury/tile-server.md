---
title: Tile Server
description: My throw-away comment at the end of my earlier post shows my scepticism
  that the JSON file approach was really viable.
url: https://www.tunbury.org/2025/12/02/tessera-stac/
date: 2025-12-02T20:00:00-00:00
preview_image: https://www.tunbury.org/images/meighen-island.png
authors:
- Mark Elvers
source:
ignore:
---

<p>My throw-away comment at the end of my earlier <a href="https://www.tunbury.org/2025/11/30/tessera-zarr/">post</a> shows my scepticism that the JSON file approach was really viable.</p>

<p>A quick <code class="language-plaintext highlighter-rouge">ls | wc -l</code> shows nearly one million tiles in 2024 alone. We need a different approach. There are already parquet files available, and checking <code class="language-plaintext highlighter-rouge">register.parquet</code>, I can see it has everything we need!</p>

<p>As an alternative, more scalable solution, we could have a server that loads the Parquet files using <a href="https://github.com/mtelvers/arrow">mtelvers/arrow</a>, derived from <a href="https://github.com/LaurentMazare/ocaml-arrow">LaurentMazare/ocaml-arrow</a>, which can respond to queries raised by callbacks from Leaflet, allowing it to draw the required bounding boxes. Ultimately this could provide links to the Zarr data in stored in S3.</p>

<p>It’s a pretty simple API:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">GET /years</code> - Available years</li>
  <li><code class="language-plaintext highlighter-rouge">GET /stats?year=YYYY</code> - Coverage statistics</li>
  <li><code class="language-plaintext highlighter-rouge">GET /tiles?minx=&amp;miny=&amp;maxx=&amp;maxy=&amp;year=&amp;limit=</code> - Tiles in bounding box</li>
  <li><code class="language-plaintext highlighter-rouge">GET /density?year=&amp;resolution=</code> - Tile density grid</li>
</ul>

<p>The code is available at <a href="https://github.com/mtelvers/tile-server">mtelvers/title-server</a> and currently deployed at <a href="https://stac.mint.caelum.ci.dev">stac.mint.caelum.ci.dev</a>.</p>
