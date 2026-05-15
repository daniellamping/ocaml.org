---
title: Using `day10` to build an OxCaml project
description: Today, looking at my OxCaml inference engine, I wanted to see whether
  day10 build . could build an OxCaml project.
url: https://www.tunbury.org/2026/04/30/day10-oxcaml/
date: 2026-04-30T20:00:00-00:00
preview_image: https://www.tunbury.org/images/oxcaml.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Today, looking at my <a href="https://www.tunbury.org/2026/03/13/oxcaml-inference/">OxCaml inference engine</a>, I wanted to see whether <a href="https://www.tunbury.org/2026/04/17/day10-build/"><code class="language-plaintext highlighter-rouge">day10 build .</code></a> could build an <a href="https://oxcaml.org">OxCaml</a> project.</p>

<p>As I already had <a href="https://github.com/oxcaml/opam-repository">oxcaml/opam-repository</a>, clone locally, along with <a href="https://github.com/ocaml/opam-repository">ocaml/opam-repository</a>. <code class="language-plaintext highlighter-rouge">day10</code> supports multiple repositories on the command line and in <code class="language-plaintext highlighter-rouge">.day10</code>, so it should work.</p>

<p>The project had no <code class="language-plaintext highlighter-rouge">.opam</code> file, so I wrote a minimal one based on <code class="language-plaintext highlighter-rouge">dune-project</code>:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>opam-version: "2.0"
synopsis: "Tessera ONNX inference engine in OxCaml with AVX2 SIMD"
maintainer: "mark@tarides.com"
authors: "Mark Elvers"
license: "ISC"
depends: [
  "ocaml-variants" {= "5.2.0+ox"}
  "dune" {&gt;= "3.21"}
  "ocaml-protoc"
  "pbrt"
  "ocaml_simd"
  "base_bigstring"
]
build: [
  ["dune" "build" "-p" name "-j" jobs]
]
</code></pre></div></div>

<p>The OxCaml compiler is <code class="language-plaintext highlighter-rouge">ocaml-variants.5.2.0+ox</code>, and <code class="language-plaintext highlighter-rouge">day10</code> parses <code class="language-plaintext highlighter-rouge">OCAML_VERSION</code> straight through <code class="language-plaintext highlighter-rouge">OpamPackage.of_string</code>, so the literal name has to match the package as it appears in the repository.</p>

<p>A <code class="language-plaintext highlighter-rouge">.day10</code> in the project root pins the compiler and lists the two repositories. The order matters: the overlay comes first, so its <code class="language-plaintext highlighter-rouge">+ox</code> forks shadow the upstream versions:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>OCAML_VERSION=ocaml-variants.5.2.0+ox
OPAM_REPOSITORY=/home/mtelvers/opam-repository-oxcaml
OPAM_REPOSITORY=/home/mtelvers/opam-repository
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">day10 build .</code> solved the constraints, built the dependency layers, and produced a working <code class="language-plaintext highlighter-rouge">_build/default/bin/main.exe</code>.</p>

<p>The project code is available at <a href="https://github.com/mtelvers/day10">mtelvers/day10</a> and the inference engine is at <a href="https://github.com/mtelvers/oxcaml-infer">mtelvers/oxcaml-infer</a>.</p>
