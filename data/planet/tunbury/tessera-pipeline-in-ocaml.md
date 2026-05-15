---
title: Tessera pipeline in OCaml
description: The Tessera pipeline is written in Python. What would it take to have
  an OCaml version?
url: https://www.tunbury.org/2026/02/15/ocaml-tessera/
date: 2026-02-15T19:30:00-00:00
preview_image: https://www.tunbury.org/images/manchester.png
authors:
- Mark Elvers
source:
ignore:
---

<p>The Tessera pipeline is written in Python. What would it take to have an OCaml version?</p>

<p>Looking at the Python code, these are the key libraries which are used:</p>

<table>
  <thead>
    <tr>
      <th>Python Library</th>
      <th>Used for</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>numpy</strong></td>
      <td>N-dim arrays, math, <code class="language-plaintext highlighter-rouge">.npy</code> I/O</td>
    </tr>
    <tr>
      <td><strong>torch</strong></td>
      <td>Model inference</td>
    </tr>
    <tr>
      <td><strong>rasterio</strong></td>
      <td>Read GeoTIFF (ROI mask), CRS/bounds, <code class="language-plaintext highlighter-rouge">transform_bounds</code></td>
    </tr>
    <tr>
      <td><strong>pystac-client</strong></td>
      <td>STAC API search (Planetary Computer catalog)</td>
    </tr>
    <tr>
      <td><strong>planetary-computer</strong></td>
      <td>Sign STAC URLs (Azure SAS tokens)</td>
    </tr>
    <tr>
      <td><strong>stackstac</strong></td>
      <td>Load COGs into arrays, reproject, mosaic</td>
    </tr>
  </tbody>
</table>

<h1>numpy</h1>

<p>Last year, when I first looked at the Tessera titles, I wrote <a href="https://github.com/mtelvers/npy-pca">mtelvers/npy-pca</a> as a basic visualisation tool that included an npy reader. Now, I have spun that off into its own library <a href="https://github.com/mtelvers/ocaml-npy">mtelvers/ocaml-npy</a>. I subsequently noticed that there already was <a href="https://github.com/LaurentMazare/npy-ocaml">LaurentMazare/npy-ocaml</a> which may have saved me some time!</p>

<h1>pystac-client and planetary-computer</h1>

<p>For these, a new library was needed as I couldn’t see an OCaml equivalent. However, OCaml already has <a href="https://github.com/ocaml-multicore/eio">Eio</a>, <a href="https://github.com/mirage/ocaml-cohttp">cohttp-eio</a> and <a href="https://github.com/ocaml-community/yojson">yojson</a>, so it was relatively easy to produce <a href="https://github.com/mtelvers/stac-client">mtelvers/stac-client</a>, which implemented the <a href="https://stacspec.org/">STAC</a> (SpatioTemporal Asset Catalogue) API, with built-in support for <a href="https://planetarycomputer.microsoft.com/">Microsoft Planetary Computer</a> SAS token signing. This was easy to validate against the results from Python.</p>

<h1>rasterio</h1>

<p><a href="https://github.com/geocaml/ocaml-tiff">geocaml/ocaml-tiff</a> already exists, but it does not handle tiled tiff files, which are used in the land masks. Rather than reinventing the entire library, I added tiled tiff support.</p>

<h1>stackstac</h1>

<p><a href="https://github.com/geocaml/ocaml-gdal">geocaml/ocaml-gdal</a> already existed, but it lacked some required features and was a little outdated. More bindings were added for GDAL’s C API using OCaml’s ctypes-foreign adding:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">GDALOpenEx</code> with <code class="language-plaintext highlighter-rouge">/vsicurl/</code> for reading remote COGs</li>
  <li><code class="language-plaintext highlighter-rouge">GDALWarp</code> for reprojection and resampling</li>
  <li><code class="language-plaintext highlighter-rouge">GDALRasterIO</code> for reading band data</li>
  <li><code class="language-plaintext highlighter-rouge">OSRNewSpatialReference</code> / <code class="language-plaintext highlighter-rouge">OCTTransformBounds</code> for coordinate transformations</li>
</ul>

<h1>torch</h1>

<p><a href="https://github.com/LaurentMazare/ocaml-torch">LaurentMazare/ocaml-torch</a> already existed with the latest version published on opam <a href="https://github.com/janestreet/torch">janestreet/torch</a>. This uses the Jane Street standard library but it seemed pointless to reimplement this using the OCaml Standard Library, so instead, I went with implementing the OCaml bindings for the ONNX runtime <a href="https://github.com/mtelvers/ocaml-onnxruntime">mtelvers/ocaml-onnxruntime</a> as I only need the inference stage. The PyTorch model can be easily exported to ONNX format.</p>

<p>ONNX Runtime’s C API uses a function-table pattern (a struct with 500+ function pointers) which doesn’t easily map to ctypes. This needed a thin C shim (<code class="language-plaintext highlighter-rouge">libert_shim.so</code>) that exposed the needed functions as regular C symbols, which could be bound from OCaml.</p>

<h1>CPU Testing</h1>

<p>The initial OCaml pipeline was tested on my local machine without a GPU. It stored satellite data as nested OCaml arrays (<code class="language-plaintext highlighter-rouge">float array array array array</code> for 4D data), which performed poorly. This was replaced with flat <code class="language-plaintext highlighter-rouge">Bigarray.Array1.t</code> using a stride-based index arithmetic, matching NumPy’s contiguous memory layout, which performed much better. However, the real test was on a GPU.</p>

<h2>Benchmark results</h2>

<p>All benchmarks on the same machine (AMD EPYC 9965 2 x 192-Core, NVIDIA L4 24GB), same dataset (269,908 pixels), same parameters (<code class="language-plaintext highlighter-rouge">batch_size=1024</code>, <code class="language-plaintext highlighter-rouge">num_threads=20</code>, <code class="language-plaintext highlighter-rouge">repeat_times=1</code>):</p>

<table>
  <thead>
    <tr>
      <th>Rank</th>
      <th>Configuration</th>
      <th>Inference Time</th>
      <th>vs Python CPU</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td><strong>OCaml + ONNX Runtime + CUDA</strong></td>
      <td><strong>2 min 10s</strong></td>
      <td><strong>9.5x faster</strong></td>
    </tr>
    <tr>
      <td>2</td>
      <td>Python + PyTorch + CUDA</td>
      <td>2 min 41s</td>
      <td>7.7x faster</td>
    </tr>
    <tr>
      <td>3</td>
      <td>Python + PyTorch (CPU)</td>
      <td>20 min 32s</td>
      <td>1x (baseline)</td>
    </tr>
    <tr>
      <td>4</td>
      <td>OCaml + ONNX Runtime (CPU)</td>
      <td>24 min 56s</td>
      <td>0.82x</td>
    </tr>
  </tbody>
</table>

<p>The OCaml + GPU configuration is the fastest overall. I put this difference down less data marshalling in OCaml before passing it to the ONNX runtime. I’ve also read that the ONNX Runtime might edge out ahead of PyTorch as it was purpose-built as an inference-only engine.</p>

<h1>Checks</h1>

<p>The OCaml pipeline produces results that are effectively identical to Python’s, differing only due to floating-point rounding.</p>

<ul>
  <li>OCaml CPU vs Python CPU: max embedding difference of 1 in only 1,028 out of 155 million int8 elements (rounding at the quantisation boundary). Scale factors match exactly.</li>
  <li>GPU vs CPU (either language): max embedding difference of 1 in ~0.3% of elements, with negligible scale differences — expected floating-point rounding differences from GPU arithmetic.</li>
</ul>
