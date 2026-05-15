---
title: Windows Docker Images
description: "In my previous post on the base image builder, I included a footnote
  that we now had Windows 2025 workers, but I didn\u2019t mention that the base images
  weren\u2019t building."
url: https://www.tunbury.org/2026/02/09/base-image-builder/
date: 2026-02-09T09:30:00-00:00
preview_image: https://www.tunbury.org/images/docker-base-images.png
authors:
- Mark Elvers
source:
ignore:
---

<p>In my previous post on the <a href="https://www.tunbury.org/2026/01/16/base-image-builder/">base image builder</a>, I included a footnote that we now had Windows 2025 workers, but I didn’t mention that the base images weren’t building.</p>

<p>Docker on Windows is very slow, so I have had a background task nudging these builds forward a little bit each day, and I’m pleased to now report that over the weekend, the images all built, and the entire dashboard is green!</p>

<p>The most significant change was moving away from fdopen’s opam to native opam. This has unlocked OCaml 5 builds for the first time but has removed images for OCaml &lt; 4.13. MSVC 5.0-5.2 are not available as the MSVC port was broken until OCaml 5.3 <a href="https://github.com/ocaml/ocaml/pull/12954">ocaml/ocaml#12954</a>. Each version is built on Windows Server LTSC 2019, LTSC 2022, and LTSC 2025.</p>

<table>
  <thead>
    <tr>
      <th>OCaml Version</th>
      <th style="text-align: center">MinGW</th>
      <th style="text-align: center">MSVC</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>4.13.1</td>
      <td style="text-align: center">✓</td>
      <td style="text-align: center">✓</td>
    </tr>
    <tr>
      <td>4.14.2</td>
      <td style="text-align: center">✓</td>
      <td style="text-align: center">✓</td>
    </tr>
    <tr>
      <td>5.0.0</td>
      <td style="text-align: center">✓</td>
      <td style="text-align: center">✗</td>
    </tr>
    <tr>
      <td>5.1.1</td>
      <td style="text-align: center">✓</td>
      <td style="text-align: center">✗</td>
    </tr>
    <tr>
      <td>5.2.1</td>
      <td style="text-align: center">✓</td>
      <td style="text-align: center">✗</td>
    </tr>
    <tr>
      <td>5.3.0</td>
      <td style="text-align: center">✓</td>
      <td style="text-align: center">✓</td>
    </tr>
    <tr>
      <td>5.4.0</td>
      <td style="text-align: center">✓</td>
      <td style="text-align: center">✓</td>
    </tr>
  </tbody>
</table>

<p>Below are the detailed changes.</p>

<h1><a href="https://github.com/ocurrent/ocaml-dockerfile/pull/257">PR 257 ocaml-dockerfile</a></h1>

<p><code class="language-plaintext highlighter-rouge">src-opam/distro.ml</code>:</p>

<ul>
  <li>Changed <code class="language-plaintext highlighter-rouge">opam_repository</code> to use standard <code class="language-plaintext highlighter-rouge">ocaml/opam-repository.git</code> for Windows instead of <code class="language-plaintext highlighter-rouge">ocaml-opam/opam-repository-mingw.git#sunset</code></li>
  <li>Added version filter: Windows builds now require OCaml &gt;= 4.13 (native opam 2.2+ requires official packages)</li>
  <li>MSVC filter: OCaml 5.0-5.2 excluded (MSVC support restored in 5.3)</li>
</ul>

<p><code class="language-plaintext highlighter-rouge">src-opam/windows.ml</code>:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">ocaml_for_windows_package_exn</code> now returns <code class="language-plaintext highlighter-rouge">Ocaml_version.Opam.V2.package</code> directly, using official package names (<code class="language-plaintext highlighter-rouge">ocaml-base-compiler/ocaml-variants+options</code>) instead of fdopen’s <code class="language-plaintext highlighter-rouge">+mingw64</code>/<code class="language-plaintext highlighter-rouge">+msvc64</code> naming</li>
</ul>

<p><code class="language-plaintext highlighter-rouge">src-opam/opam.ml</code>:</p>

<ul>
  <li>Reduce parallelism on Windows to avoid OOM on unbound <code class="language-plaintext highlighter-rouge">make -j</code></li>
  <li>Update Visual Studio to Windows 11 SDK</li>
  <li>create_switch adds <code class="language-plaintext highlighter-rouge">system-mingw/system-msvc</code> for all Windows versions (not just 5.x)</li>
  <li><code class="language-plaintext highlighter-rouge">setup_default_opam_windows_msvc</code> persists MSVC environment (<code class="language-plaintext highlighter-rouge">PATH</code>, <code class="language-plaintext highlighter-rouge">INCLUDE</code>, <code class="language-plaintext highlighter-rouge">LIB</code>, <code class="language-plaintext highlighter-rouge">LIBPATH</code>) with correct <code class="language-plaintext highlighter-rouge">PATH</code> ordering: MSVC → Cygwin → Windows</li>
</ul>

<h1><a href="https://github.com/ocurrent/docker-base-images/pull/339">PR 339 docker-base-images</a></h1>

<p><code class="language-plaintext highlighter-rouge">src/pipeline.ml</code>:</p>

<ul>
  <li>Port package (<code class="language-plaintext highlighter-rouge">system-mingw</code>/<code class="language-plaintext highlighter-rouge">system-msvc</code>) added for all Windows versions</li>
  <li>Removed fdopen overlay addition (<code class="language-plaintext highlighter-rouge">maybe_add_overlay</code> no longer called for Windows)</li>
  <li>Removed <code class="language-plaintext highlighter-rouge">opam repo remove ocurrent-overlay</code> step</li>
  <li>Changed <code class="language-plaintext highlighter-rouge">depext</code> to <code class="language-plaintext highlighter-rouge">Option</code> type - returns <code class="language-plaintext highlighter-rouge">None</code> for Windows (opam 2.2+ has depext built-in)</li>
  <li>Uses <code class="language-plaintext highlighter-rouge">opam_repository_master</code> for Windows instead of <code class="language-plaintext highlighter-rouge">opam_repository_mingw_sunset</code></li>
</ul>

<p><code class="language-plaintext highlighter-rouge">src/git_repositories.ml</code> (implied by pipeline changes):</p>

<ul>
  <li>Removed references to <code class="language-plaintext highlighter-rouge">opam_repository_mingw_sunset</code> and <code class="language-plaintext highlighter-rouge">opam_overlays</code>.</li>
</ul>
