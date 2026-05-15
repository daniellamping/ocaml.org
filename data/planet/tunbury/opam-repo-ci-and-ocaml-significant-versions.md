---
title: opam-repo-ci and OCaml significant versions
description: Updates to opam-repo-ci which pull in the latest ocaml-version and ocaml-dockerfile
  releases trim the build matrix and add in the latest releases of Alpine and Ubuntu.
url: https://www.tunbury.org/2026/04/27/opam-repo-ci-update/
date: 2026-04-27T07:30:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Updates to <a href="https://github.com/ocurrent/opam-repo-ci">opam-repo-ci</a> which pull in the latest <a href="https://github.com/ocurrent/ocaml-version">ocaml-version</a> and <a href="https://github.com/ocurrent/ocaml-dockerfile">ocaml-dockerfile</a> releases trim the build matrix and add in the latest releases of Alpine and Ubuntu.</p>

<h1>ocaml-version 4.1.0</h1>

<p><a href="https://github.com/ocurrent/ocaml-version/releases/tag/v4.1.0">ocaml-version 4.1.0</a> adds OCaml 5.6 as a <code class="language-plaintext highlighter-rouge">dev</code> version and <code class="language-plaintext highlighter-rouge">5.5.0~beta1</code> to <code class="language-plaintext highlighter-rouge">unreleased_betas</code> <a href="https://github.com/ocurrent/ocaml-version/pull/88">PR#88</a>.</p>

<p>It also exposes a new <code class="language-plaintext highlighter-rouge">significant</code> list <a href="https://github.com/ocurrent/ocaml-version/pull/87">PR#87</a>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">significant</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">last_two</span> <span class="o">=</span> <span class="k">match</span> <span class="nn">List</span><span class="p">.</span><span class="n">rev</span> <span class="n">all</span> <span class="k">with</span>
    <span class="o">|</span> <span class="n">a</span> <span class="o">::</span> <span class="n">b</span> <span class="o">::</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="p">[</span><span class="n">b</span><span class="p">;</span> <span class="n">a</span><span class="p">]</span> <span class="o">|</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="bp">[]</span> <span class="k">in</span>
  <span class="nn">List</span><span class="p">.</span><span class="n">sort_uniq</span> <span class="n">compare</span> <span class="p">([</span> <span class="n">v4_08</span><span class="p">;</span> <span class="n">v4_11</span><span class="p">;</span> <span class="n">v4_14</span><span class="p">;</span> <span class="n">v5_2</span> <span class="p">]</span> <span class="o">@</span> <span class="n">last_two</span><span class="p">)</span>
</code></pre></div></div>

<p>Currently this resolves to <code class="language-plaintext highlighter-rouge">[4.08; 4.11; 4.14; 5.2; 5.3; 5.4]</code> which is a curated base from the 4.x series plus the last two releases as the list of OCaml versions against which opam packages should be regularly tested. For further detail, review the conversation on the thread of <a href="https://github.com/ocurrent/ocaml-version/pull/88">PR#88</a>. This is a deliberate thinning of <code class="language-plaintext highlighter-rouge">recent</code>, which had grown to all stable releases from 4.08 to 5.4 (eleven versions).</p>

<p>In <code class="language-plaintext highlighter-rouge">opam-ci-check</code>, the <code class="language-plaintext highlighter-rouge">all_supported</code> list used to drive the full build matrix is updated to use <code class="language-plaintext highlighter-rouge">significant</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">all_supported</span> <span class="o">=</span> <span class="nn">Ocaml_version</span><span class="p">.</span><span class="nn">Releases</span><span class="p">.</span><span class="n">significant</span> <span class="o">@</span> <span class="nn">Ocaml_version</span><span class="p">.</span><span class="nn">Releases</span><span class="p">.</span><span class="n">unreleased_betas</span>
</code></pre></div></div>

<p>The betas are only built on the master distro on x86_64, so this does not multiply out across the matrix.</p>

<h1>Alpine 3.23 and Ubuntu 25.10 and 26.04</h1>

<p><a href="https://github.com/ocurrent/ocaml-dockerfile/releases/tag/8.3.5">ocaml-dockerfile 8.3.5</a> adds Alpine 3.23, replacing Alpine 3.22 and add Ubuntu 25.10. Additionally, <a href="https://github.com/ocurrent/ocaml-dockerfile/releases/tag/8.3.6">ocaml-dockerfile 8.3.6</a> adds Ubuntu 26.04. No code changes are required in opam-repo-ci; pulling in the updated dockerfile package and bumping the pinned opam-repository SHA in the <code class="language-plaintext highlighter-rouge">Dockerfile</code> and <code class="language-plaintext highlighter-rouge">Dockerfile.web</code> is sufficient.</p>

<h1>Dropping i386 and Arm32</h1>

<p>The <code class="language-plaintext highlighter-rouge">extras</code> function in <code class="language-plaintext highlighter-rouge">lib/build.ml</code> builds each non-master architecture against the two default compilers. Previously, this covered every arch in <code class="language-plaintext highlighter-rouge">Ocaml_version.arches</code> other than <code class="language-plaintext highlighter-rouge">X86_64</code>. <a href="https://github.com/ocurrent/opam-repo-ci/pull/467">PR#467</a> dropped <code class="language-plaintext highlighter-rouge">Aarch32</code>; this has been extended to also drop <code class="language-plaintext highlighter-rouge">I386</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nn">List</span><span class="p">.</span><span class="n">filter_map</span> <span class="p">(</span><span class="k">function</span>
  <span class="o">|</span> <span class="nt">`X86_64</span> <span class="o">|</span> <span class="nt">`Aarch32</span> <span class="o">|</span> <span class="nt">`I386</span> <span class="o">-&gt;</span> <span class="nc">None</span>
  <span class="o">|</span> <span class="nt">`Riscv64</span> <span class="o">-&gt;</span> <span class="o">...</span>
  <span class="o">|</span> <span class="n">arch</span> <span class="o">-&gt;</span> <span class="o">...</span>
<span class="p">)</span> <span class="nn">Ocaml_version</span><span class="p">.</span><span class="n">arches</span>
</code></pre></div></div>

<p>Neither architecture reflects a realistic target for modern opam packages, and both were a steady source of flaky builds that rarely surfaced genuine portability issues. See <a href="https://github.com/ocurrent/opam-repo-ci/issues/466">Issue#366</a> for more information. <a href="https://github.com/ocurrent/docker-base-images/pull/346">PR#346</a> proposes to stop build the 32-bit base images as well.</p>
