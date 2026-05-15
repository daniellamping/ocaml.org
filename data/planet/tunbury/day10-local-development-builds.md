---
title: 'Day10: local development builds'
description: My typical OCaml development workflow starts with git clone ..., then
  opam switch create . 5.4.1 --deps-only followed by dune build. This creates a local
  _opam directory containing the compiler and all dependencies. It works well, but
  the _opam directories add up, and the build takes time.
url: https://www.tunbury.org/2026/04/17/day10-build/
date: 2026-04-17T16:00:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>My typical OCaml development workflow starts with <code class="language-plaintext highlighter-rouge">git clone ...</code>, then <code class="language-plaintext highlighter-rouge">opam switch create . 5.4.1 --deps-only</code> followed by <code class="language-plaintext highlighter-rouge">dune build</code>. This creates a local <code class="language-plaintext highlighter-rouge">_opam</code> directory containing the compiler and all dependencies. It works well, but the <code class="language-plaintext highlighter-rouge">_opam</code> directories add up, and the build takes time.</p>

<p>Across my projects, the cumulative size of these <code class="language-plaintext highlighter-rouge">_opam</code> directories is 88 GB, with individual switches ranging from around 400 MB to over 3 GB. Most of that space is taken up by duplicated packages, rebuilt from scratch for each project.</p>

<p><code class="language-plaintext highlighter-rouge">day10</code> already solves this problem for testing packages by building dependency layers once and reusing them across packages. The new <code class="language-plaintext highlighter-rouge">day10 build</code> subcommand brings the same approach to local development. Instead of creating an opam switch, it assembles a container from cached dependency layers and runs <code class="language-plaintext highlighter-rouge">dune build</code> inside it, with the source directory bind-mounted so that <code class="language-plaintext highlighter-rouge">_build/</code> appears on the host.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 build <span class="nb">.</span>
</code></pre></div></div>

<p>This is equivalent to <code class="language-plaintext highlighter-rouge">opam switch create . --deps-only &amp;&amp; dune build</code>, but dependencies are never rebuilt if they already exist in the cache. On a warm cache, the solve takes under a second, and the container assembly is near-instant, so you go straight to your project’s compilation.</p>

<p>On a cold cache, <code class="language-plaintext highlighter-rouge">day10 build</code> is slower than <code class="language-plaintext highlighter-rouge">opam</code> for a single project because it builds each dependency sequentially, one layer at a time, whereas <code class="language-plaintext highlighter-rouge">opam</code> parallelises the installation. The advantage comes when you work on multiple projects: the cached layers are shared, so dependencies compiled for one project are reused by the next.</p>

<p>Because everything runs inside the container, including the compiler, opam, dune, and all dependencies, the host machine does not need to install any OCaml toolchain. The only requirements are <code class="language-plaintext highlighter-rouge">day10</code> itself and <code class="language-plaintext highlighter-rouge">runc</code>, which could be made available as a binary release. Then, a new contributor can clone a project and run <code class="language-plaintext highlighter-rouge">day10 build .</code> without installing opam, configuring a switch, or even having OCaml on their system.</p>

<p>Extra arguments are passed through to <code class="language-plaintext highlighter-rouge">dune build</code>:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 build <span class="nb">.</span> @install
day10 build <span class="nb">.</span> @runtest
</code></pre></div></div>

<p>Test dependencies are included with <code class="language-plaintext highlighter-rouge">--with-test</code>:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 build <span class="nt">--with-test</span> <span class="nb">.</span> @runtest
</code></pre></div></div>

<p>For non-dune projects, a custom build command can be specified:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 build <span class="nt">--command</span> <span class="s2">"make"</span> <span class="nb">.</span>
</code></pre></div></div>

<p>Multiple opam repositories can be specified, which is useful for projects that depend on packages not yet in the main repository or that use a development overlay:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 build <span class="nt">--opam-repository</span> ~/custom-repo <span class="nt">--opam-repository</span> ~/opam-repository <span class="nb">.</span>
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">day10 build</code> recursively discovers all <code class="language-plaintext highlighter-rouge">.opam</code> files in the project directory, including vendored subdirectories. These are pinned during dependency resolution, so the solver uses the local versions rather than pulling packages from the opam repository. The local packages themselves are not installed into the switch; <code class="language-plaintext highlighter-rouge">dune</code> builds them directly from the workspace source.</p>

<p>This means projects like <a href="https://github.com/ocurrent/ocluster">ocluster</a>, which vendors <a href="https://github.com/ocurrent/obuilder">obuilder</a> as a subdirectory with its own <code class="language-plaintext highlighter-rouge">dune-project</code>, work correctly. The vendored packages are built by <code class="language-plaintext highlighter-rouge">dune</code> as part of the workspace, while their transitive dependencies from the opam repository are provided as cached layers.</p>

<p>Common options can be set in <code class="language-plaintext highlighter-rouge">~/.day10</code> or a local <code class="language-plaintext highlighter-rouge">.day10</code> file (which overrides the global one). The format is simple <code class="language-plaintext highlighter-rouge">KEY=VALUE</code>, and the keys are prefixed with <code class="language-plaintext highlighter-rouge">DAY10_</code> before being passed to the environment:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>CACHE_DIR=/var/cache/day10
OPAM_REPOSITORY=/home/user/opam-repository
</code></pre></div></div>

<p>Multiple opam repositories can be specified on separate lines:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>OPAM_REPOSITORY=/home/user/opam-repository
OPAM_REPOSITORY=/home/user/custom-repo
</code></pre></div></div>

<p>Per-project settings like the compiler version or a custom build command can go in <code class="language-plaintext highlighter-rouge">./.day10</code>:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>OCAML_VERSION=ocaml.5.3.0
WITH_TEST=true
COMMAND=make
</code></pre></div></div>

<p>The layer cache is shared with <code class="language-plaintext highlighter-rouge">day10 health-check</code>, so dependencies are reused for local development and vice versa. The total cache size for all my projects is 5GB.</p>

<p>The container base images are currently built for the Debian OS family, which includes Debian and Ubuntu. The distribution and version are detected from the host system by default, or can be overridden with <code class="language-plaintext highlighter-rouge">--os-distribution</code> and <code class="language-plaintext highlighter-rouge">--os-version</code>. FreeBSD and Windows are supported by <code class="language-plaintext highlighter-rouge">day10 health-check</code> but not yet by <code class="language-plaintext highlighter-rouge">day10 build</code>.</p>

<p>The project code is available at <a href="https://github.com/mtelvers/day10">mtelvers/day10</a>.</p>
