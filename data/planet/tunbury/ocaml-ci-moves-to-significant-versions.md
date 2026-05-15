---
title: ocaml-ci moves to significant versions
description: The same OCaml build matrix updates which where deployed in opam-repo-ci
  have now been applied to ocaml-ci.
url: https://www.tunbury.org/2026/04/29/ocaml-ci-update/
date: 2026-04-29T08:00:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>The same OCaml build matrix updates which where deployed in <a href="https://www.tunbury.org/2026/04/27/opam-repo-ci-update/">opam-repo-ci</a> have now been applied to <a href="https://github.com/ocurrent/ocaml-ci">ocaml-ci</a>.</p>

<p>The changes add 5.5.0~beta1 testing, switch to the new <code class="language-plaintext highlighter-rouge">Ocaml_version.Releases.significant</code> list, and drop 32-bit architectures have now been applied to <a href="https://github.com/ocurrent/ocaml-ci">ocaml-ci</a>; this post covers the OCaml-CI specific changes.</p>

<h1>Switching to <code class="language-plaintext highlighter-rouge">Releases.significant</code></h1>

<p>OCaml-CI builds the test matrix in <code class="language-plaintext highlighter-rouge">service/conf.ml</code>. The two places which used <code class="language-plaintext highlighter-rouge">Releases.recent</code> now use <code class="language-plaintext highlighter-rouge">Releases.significant</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">ovs</span> <span class="o">=</span> <span class="nn">List</span><span class="p">.</span><span class="n">rev</span> <span class="nn">OV</span><span class="p">.</span><span class="nn">Releases</span><span class="p">.</span><span class="n">significant</span> <span class="o">@</span> <span class="nn">OV</span><span class="p">.</span><span class="nn">Releases</span><span class="p">.</span><span class="n">unreleased_betas</span> <span class="k">in</span>
</code></pre></div></div>

<p>and, for the <code class="language-plaintext highlighter-rouge">Minimal</code> profile used in local development:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span><span class="p">[</span><span class="o">@</span><span class="n">warning</span> <span class="s2">"-8"</span><span class="p">]</span> <span class="p">(</span><span class="n">latest</span> <span class="o">::</span> <span class="n">previous</span> <span class="o">::</span> <span class="n">_</span><span class="p">)</span> <span class="o">=</span>
  <span class="nn">List</span><span class="p">.</span><span class="n">rev</span> <span class="nn">OV</span><span class="p">.</span><span class="nn">Releases</span><span class="p">.</span><span class="n">significant</span>
<span class="k">in</span>
</code></pre></div></div>

<p>Basically an identical change as in opam-repo-ci, dropping the OCaml version test matrix from eleven entries (every stable since 4.08) down to the curated <code class="language-plaintext highlighter-rouge">[4.08; 4.11; 4.14; 5.2; 5.3; 5.4]</code> set. This is covered in <a href="https://github.com/ocurrent/ocaml-ci/pull/1050">PR#1050</a></p>

<h1>Adding <code class="language-plaintext highlighter-rouge">5.5.0~beta1</code></h1>

<p>In the same release of ocaml-version, Kate’s PR <a href="https://github.com/ocurrent/ocaml-version/pull/88">ocurrent/ocaml-version#88</a> adds <code class="language-plaintext highlighter-rouge">5.5.0~beta1</code> to <code class="language-plaintext highlighter-rouge">OV.Releases.unreleased_betas</code>.</p>

<h1>Dropping i386 and Arm32</h1>

<p>OCaml-CI’s builds each compiler across <code class="language-plaintext highlighter-rouge">Dockerfile_opam.Distro.distro_arches</code>. That list is now filtered to skip 32-bit:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">arches</span> <span class="o">=</span>
  <span class="nn">DD</span><span class="p">.</span><span class="n">distro_arches</span> <span class="n">ov</span> <span class="p">(</span><span class="n">distro</span> <span class="o">:&gt;</span> <span class="nn">DD</span><span class="p">.</span><span class="n">t</span><span class="p">)</span>
  <span class="o">|&gt;</span> <span class="nn">List</span><span class="p">.</span><span class="n">filter</span> <span class="p">(</span><span class="k">function</span> <span class="nt">`I386</span> <span class="o">|</span> <span class="nt">`Aarch32</span> <span class="o">-&gt;</span> <span class="bp">false</span> <span class="o">|</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="bp">true</span><span class="p">)</span>
<span class="k">in</span>
</code></pre></div></div>

<p>This also makes the existing <code class="language-plaintext highlighter-rouge">excluded_selection</code> fudge in <code class="language-plaintext highlighter-rouge">lib/pipeline.ml</code> unused. This was a workaround for <a href="https://github.com/ocurrent/ocaml-ci/issues/931">issue #931</a> that suppressed <code class="language-plaintext highlighter-rouge">conf-capnproto</code> selections on debian-12/i386. With i386 gone from the matrix, the filter has nothing left to filter, so it has been removed.</p>

<h1>Dropping the vendored submodules</h1>

<p>In <a href="https://github.com/ocurrent/ocaml-ci/pull/1049">PR #1049</a>, I removed the vendored submodules: <a href="https://github.com/ocurrent/ocurrent">ocurrent</a>, <a href="https://github.com/ocurrent/ocluster">ocluster</a>, <a href="https://github.com/ocurrent/ocaml-dockerfile">ocaml-dockerfile</a>, <a href="https://github.com/ocurrent/ocaml-version">ocaml-version</a> and <a href="https://github.com/ocurrent/solver-service">solver-service</a>. This mirrors <a href="https://github.com/ocurrent/opam-repo-ci/pull/349">PR#349</a> from 2024, which did the same in opam-repo-ci.</p>

<p>As a result, the Dockerfile’s are substantially shorter with only the solver-service pin remaining. <code class="language-plaintext highlighter-rouge">Dockerfile.gitlab</code> and <code class="language-plaintext highlighter-rouge">Dockerfile.web</code> were also updated at the same time: debian-13 / opam 2.5 / <code class="language-plaintext highlighter-rouge">docker-cli</code> (in place of the full <code class="language-plaintext highlighter-rouge">docker.io</code> engine package, which pulled in the daemon).</p>
