---
title: OCI image export from day10
description: mtelvers/day10 can now export build results as multi-layer OCI images,
  where each opam package becomes its own layer.
url: https://www.tunbury.org/2026/04/09/oci-export/
date: 2026-04-09T15:15:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p><a href="https://github.com/mtelvers/day10">mtelvers/day10</a> can now export build results as multi-layer OCI images, where each opam package becomes its own layer.</p>

<h1>Background</h1>

<p>Previously, <a href="https://github.com/mtelvers/day10">mtelvers/day10</a> could export builds into Docker using <code class="language-plaintext highlighter-rouge">--tag</code>, which assembled all layers into a single flat filesystem and piped it through <code class="language-plaintext highlighter-rouge">docker import</code>. This produced working images but threw away the layer structure that makes container distribution efficient. Every image was a single layer, regardless of how much it shared with other builds. Last year, I played with <a href="https://www.tunbury.org/2025/08/18/buildkit-bake/">BuildKit Bake</a>, attempting to create a Dockerfile for each package in opam.</p>

<p>The new <code class="language-plaintext highlighter-rouge">--oci</code> flag generates an OCI image layout directory, which I think of as a Docker registry on the file system. Each opam package in the dependency tree becomes a separate layer, and images built into the same directory naturally deduplicate shared layers through content-addressed storage.</p>

<h1>Usage</h1>

<p>Build a package and generate an OCI image:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 health-check \
  --cache-dir /var/cache/day10 \
  --opam-repository /path/to/opam-repository \
  --oci /tmp/oci-output \
  0install.2.18
</code></pre></div></div>

<p>This creates an OCI image layout at <code class="language-plaintext highlighter-rouge">/tmp/oci-output</code> with one layer per package: a base Debian/Ubuntu layer, then <code class="language-plaintext highlighter-rouge">ocaml-base-compiler</code>, <code class="language-plaintext highlighter-rouge">dune</code>, <code class="language-plaintext highlighter-rouge">ocamlfind</code>, <code class="language-plaintext highlighter-rouge">lwt</code>, and so on up to <code class="language-plaintext highlighter-rouge">0install</code> itself.</p>

<p>Build a second package into the same directory:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 health-check \
  --cache-dir /var/cache/day10 \
  --opam-repository /path/to/opam-repository \
  --oci /tmp/oci-output \
  0install-gtk.2.18
</code></pre></div></div>

<p>The shared dependencies including the base system, the compiler, dune, lwt, and everything else in common are not re-created. The OCI blobs directory already contains those layers, and the new image’s manifest simply references them.</p>

<h1>Batch builds with –fork</h1>

<p>For building many packages in parallel, pass a JSON package list and <code class="language-plaintext highlighter-rouge">--fork</code>:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 health-check \
  --cache-dir /var/cache/day10 \
  --opam-repository /path/to/opam-repository \
  --oci /tmp/oci-output \
  --fork 10 \
  @packages.json
</code></pre></div></div>

<p>All forked processes write into the same OCI directory. Layer creation is protected by file locking so concurrent builds of different packages that share dependencies don’t conflict. The shared <code class="language-plaintext highlighter-rouge">index.json</code> accumulates a manifest entry for each package as it completes.</p>

<h1>Pushing to a registry</h1>

<p>I used <a href="https://github.com/containers/skopeo">skopeo</a> to push images to a container registry:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>skopeo copy \
  oci:/tmp/oci-output:0install.2.18 \
  docker://docker.io/ocurrent/ocaml-packages:0install-2.18
</code></pre></div></div>

<p>If you push multiple images that share layers, the registry deduplicates them:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Copying blob sha256:1b7427dc... already exists
Skipping blob sha256:1b7427dc... (already present)
Copying blob sha256:f3d6a324... already exists
Skipping blob sha256:f3d6a324... (already present)
...
Writing manifest to image destination
</code></pre></div></div>

<h1>Running images with Docker</h1>

<p>Install skopeo and load an OCI image into the Docker daemon:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>skopeo copy \
  oci:/tmp/oci-output:0install.2.18 \
  docker-daemon:0install:2.18
</code></pre></div></div>

<p>Then run it:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>$ docker run --rm 0install:2.18 opam exec -- 0install --version
0install (zero-install) 2.18
</code></pre></div></div>

<p>Docker shares layers between images at the storage driver level.</p>

<h1>The 128-layer limit</h1>

<p>Docker’s overlay2 storage driver limits each image to 128 layers, which makes it much less useful than I had hoped. Packages with deep dependency trees can exceed this, for example, <code class="language-plaintext highlighter-rouge">ocluster.0.3.0</code> has 140 layers. You can load this into a registry without complaint, but it fails with a local Docker.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>$ skopeo copy oci:/tmp/oci-output:ocluster.0.3.0 docker-daemon:ocluster:0.3.0
FATAL: max depth exceeded
</code></pre></div></div>

<p>This is a Docker daemon infact kernel overlayfs constraint, not a registry or OCI spec limitation. The image is valid and can be stored, pushed, and pulled from any OCI-compliant registry.</p>

<p>The original <code class="language-plaintext highlighter-rouge">--tag</code> flag is unaffected by this change and continues to produce a single-layer image via <code class="language-plaintext highlighter-rouge">docker import</code>, which has no depth limit.</p>

<h1>Storage savings</h1>

<p>Building two packages (<code class="language-plaintext highlighter-rouge">0install.2.18</code> and <code class="language-plaintext highlighter-rouge">ocluster.0.3.0</code>) into the same OCI directory:</p>

<table>
  <thead>
    <tr>
      <th>&nbsp;</th>
      <th>Layers</th>
      <th>Compressed size</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0install.2.18</td>
      <td>42</td>
      <td>491 MB</td>
    </tr>
    <tr>
      <td>ocluster.0.3.0</td>
      <td>140</td>
      <td>658 MB</td>
    </tr>
    <tr>
      <td>Total (no sharing)</td>
      <td>&nbsp;</td>
      <td>1,149 MB</td>
    </tr>
    <tr>
      <td>Actual on disk (deduplicated)</td>
      <td>158 unique blobs</td>
      <td>749 MB</td>
    </tr>
    <tr>
      <td>Savings</td>
      <td>24 shared layers</td>
      <td>400 MB (35%)</td>
    </tr>
  </tbody>
</table>

<p>The savings grow with more packages. The base system (~214 MB compressed), the OCaml compiler (~138 MB), and dune (~14 MB) are shared by virtually every OCaml package. Building the full opam repository into a single OCI directory would amortise those costs across thousands of images.</p>

<h1>Layer caching</h1>

<p>Layer tarballs are cached in the build cache directory alongside each package’s filesystem. On subsequent runs, the layers are populated via hardlinks from the cache rather than re-tarring and re-compressing. Regenerating the full OCI layout for <code class="language-plaintext highlighter-rouge">0install.2.18</code> from a warm cache takes under a second.</p>

<h1>Why not Docker build?</h1>

<p>This kind of image cannot be produced by <code class="language-plaintext highlighter-rouge">docker build</code>. A Dockerfile creates layers corresponding to <code class="language-plaintext highlighter-rouge">RUN</code> instructions, so you could write a separate <code class="language-plaintext highlighter-rouge">RUN opam install &lt;pkg&gt;</code> for each dependency (see <a href="https://www.tunbury.org/2025/08/18/buildkit-bake/">BuildKit Bake-off</a>), but Docker provides no way to merge layers after the fact. If two images share the same base packages but install them in a different order, or with a single different package earlier in the chain, every subsequent layer differs.</p>

<p><a href="https://github.com/mtelvers/day10">mtelvers/day10</a> sidesteps this entirely. Each opam package is built in its own overlay filesystem, producing a diff directory that captures exactly what that package installed. These diffs are directly turned into OCI layers. Two images that happen to share <code class="language-plaintext highlighter-rouge">dune.3.22.0</code> share the same blob regardless of where it appears in their respective dependency trees.</p>

<h1>How it works</h1>

<p>Each opam package is already built in an isolated overlay filesystem, producing a diff directory (<code class="language-plaintext highlighter-rouge">fs/</code>) containing only the files installed by that package. The OCI export tars each diff, converts any overlay whiteout markers to the OCI whiteout format, gzips the result, and computes SHA256 digests for both the compressed and uncompressed forms.</p>

<p>The OCI image layout is then assembled:</p>

<ul>
  <li>Each layer tarball becomes a blob in <code class="language-plaintext highlighter-rouge">blobs/sha256/&lt;digest&gt;</code></li>
  <li>An image config records the architecture, environment, and layer diff IDs</li>
  <li>A manifest ties the config to its layers</li>
  <li>An <code class="language-plaintext highlighter-rouge">index.json</code> maps image tags to manifests</li>
</ul>

<p>The result is a spec-compliant OCI image layout that any OCI tool can consume.</p>
