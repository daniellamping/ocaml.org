---
title: Retiring opam 2.0 from the build pipeline
description: ocurrent/docker-base-images publishes the ocaml/opam:* Docker images
  which the OCaml CI systems use. For each distro, it tracks 2.0, 2.1, 2.2, 2.3, 2.4,
  2.5, and master opam release branches in parallel and produces both an opam-version-suffixed
  tag (e.g. debian-13-ocaml-5.4_opam-2.5) and an un-suffixed default that points at
  the oldest tracked version.
url: https://www.tunbury.org/2026/05/07/removing-opam-2.0/
date: 2026-05-07T14:00:00-00:00
preview_image: https://www.tunbury.org/images/opam.png
authors:
- Mark Elvers
source:
ignore:
---

<p><a href="https://github.com/ocurrent/docker-base-images">ocurrent/docker-base-images</a> publishes the <code class="language-plaintext highlighter-rouge">ocaml/opam:*</code> Docker images which the OCaml CI systems use. For each distro, it tracks 2.0, 2.1, 2.2, 2.3, 2.4, 2.5, and master opam release branches in parallel and produces both an opam-version-suffixed tag (e.g. <code class="language-plaintext highlighter-rouge">debian-13-ocaml-5.4_opam-2.5</code>) and an un-suffixed default that points at the oldest tracked version.</p>

<p>opam 2.0 is a frequent source of failed builds:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[ERROR] No solution for foo.0.0.1: The actions to process have cyclic dependencies
</code></pre></div></div>

<p><a href="https://github.com/ocurrent/docker-base-images/pull/342">ocurrent/docker-base-images#342</a> drops 2.0 from the support matrix. However, that PR couldn’t be merged in isolation because it depended on <a href="https://github.com/ocurrent/ocaml-dockerfile/pull/262">ocurrent/ocaml-dockerfile#262</a>, and the CI systems still referenced the 2.0 base images.</p>

<p>Four repositories need to be updated:</p>

<ul>
  <li><a href="https://github.com/ocurrent/ocaml-dockerfile">ocurrent/ocaml-dockerfile</a> owns the <code class="language-plaintext highlighter-rouge">Dockerfile_opam.opam_hashes</code> record that includes <code class="language-plaintext highlighter-rouge">opam_2_0_hash</code>. Removing the 2.0 channel here is the primary change.</li>
  <li><a href="https://github.com/ocurrent/docker-base-images">ocurrent/docker-base-images</a> consumes <code class="language-plaintext highlighter-rouge">Dockerfile_opam</code> and threads the opam-2.0 hash through its pipeline. PR#342 removes it.</li>
  <li><a href="https://github.com/ocurrent/opam-repo-ci">ocurrent/opam-repo-ci</a> and <a href="https://github.com/ocurrent/ocaml-ci">ocurrent/ocaml-ci</a> both pull <code class="language-plaintext highlighter-rouge">ocaml/opam:*-opam-2.0</code> tags as part of their build matrices and have type definitions that include the <code class="language-plaintext highlighter-rouge">`V2_0</code> constructor.</li>
</ul>

<h1>ocurrent/opam-repo-ci</h1>

<p><a href="https://github.com/ocurrent/opam-repo-ci">ocurrent/opam-repo-ci</a> uses the <code class="language-plaintext highlighter-rouge">extras</code> function in <code class="language-plaintext highlighter-rouge">lib/build.ml</code> to schedule revdep builds across the full opam matrix:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nn">List</span><span class="p">.</span><span class="n">map</span> <span class="p">(</span><span class="k">fun</span> <span class="n">opam_version</span> <span class="o">-&gt;</span>
  <span class="k">let</span> <span class="n">opam_string</span> <span class="o">=</span> <span class="s2">"opam-"</span> <span class="o">^</span> <span class="nn">Opam_version</span><span class="p">.</span><span class="n">to_string</span> <span class="n">opam_version</span> <span class="k">in</span>
  <span class="n">build</span> <span class="o">~</span><span class="n">opam_version</span> <span class="o">~</span><span class="n">arch</span><span class="o">:</span><span class="nt">`X86_64</span> <span class="o">~</span><span class="n">distro</span><span class="o">:</span><span class="n">master_distro</span>
    <span class="o">~</span><span class="n">compiler</span><span class="o">:</span><span class="p">(</span><span class="n">comp</span><span class="o">,</span> <span class="nc">None</span><span class="p">)</span> <span class="n">opam_string</span><span class="p">)</span>
  <span class="p">[</span> <span class="nt">`V2_0</span><span class="p">;</span> <span class="nt">`V2_1</span><span class="p">;</span> <span class="nt">`V2_2</span><span class="p">;</span> <span class="nt">`V2_3</span><span class="p">;</span> <span class="nt">`V2_4</span><span class="p">;</span> <span class="nt">`V2_5</span> <span class="p">]</span>
</code></pre></div></div>

<p>Once docker-base-images stops publishing the 2.0 tags, the <code class="language-plaintext highlighter-rouge">`V2_0</code> entry here would pull an increasingly out-of-date base image. <a href="https://github.com/ocurrent/opam-repo-ci/pull/473">PR#473</a> drops it, along with the 2.0-specific code paths in <code class="language-plaintext highlighter-rouge">opam-ci-check</code>:</p>

<ul>
  <li>The <code class="language-plaintext highlighter-rouge">`V2_0</code> constructor in <code class="language-plaintext highlighter-rouge">Opam_ci_check.Opam_version.t</code>, <code class="language-plaintext highlighter-rouge">Spec.list_revdeps</code>, and <code class="language-plaintext highlighter-rouge">opam_install</code>’s depext branch.</li>
  <li>The 2.0-specific solver-setup and depext-update commands in <code class="language-plaintext highlighter-rouge">setup_repository</code> (opam 2.0 used <code class="language-plaintext highlighter-rouge">opam depext -u</code> and didn’t support <code class="language-plaintext highlighter-rouge">opam option solver=builtin-0install</code>; everything from 2.1 onwards uses <code class="language-plaintext highlighter-rouge">opam update --depexts</code> and the builtin-0install solver).</li>
</ul>

<p>After dropping the <code class="language-plaintext highlighter-rouge">`V2_0</code> cases, the per-version match expressions are all in the same 2.1+ form. <code class="language-plaintext highlighter-rouge">test/specs.expected</code> showed the expected diff dropping the two <code class="language-plaintext highlighter-rouge">opam-2.0</code> blocks.</p>

<h1>ocurrent/ocaml-ci</h1>

<p><a href="https://github.com/ocurrent/ocaml-ci">ocurrent/opam-repo-ci</a> was more interesting, it had the same <code class="language-plaintext highlighter-rouge">`V2_0</code> constructor and 2.0-specific depext branch existed in <code class="language-plaintext highlighter-rouge">lib/opam_version.ml</code> and <code class="language-plaintext highlighter-rouge">lib/opam_build.ml</code>, but tracing the actual usage showed that it didn’t actually use it:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* service/conf.ml — every platform *)</span>
<span class="n">opam_version</span> <span class="o">=</span> <span class="nt">`V2_5</span><span class="p">;</span>

<span class="c">(* lib/lint.ml *)</span>
<span class="k">let</span> <span class="n">opam_version</span> <span class="o">=</span> <span class="nt">`V2_2</span>
</code></pre></div></div>

<p>The service uses 2.5 for builds and 2.2 for linting. The <code class="language-plaintext highlighter-rouge">Opam_version.default</code> constant was set to <code class="language-plaintext highlighter-rouge">`V2_0</code>, but that was referenced only from the function <code class="language-plaintext highlighter-rouge">Variant.of_string</code> as the fallback for unsuffixed variants, but the service never produces unsuffixed strings, so the fallback case didn’t apply.</p>

<p><a href="https://github.com/ocurrent/ocaml-ci/pull/1054">ocurrent/ocaml-ci#1054</a> takes the opportunity to clean up the surrounding code:</p>

<ul>
  <li>Drops <code class="language-plaintext highlighter-rouge">`V2_0</code> from the type and its references in <code class="language-plaintext highlighter-rouge">Variant.pp</code>, <code class="language-plaintext highlighter-rouge">opam_build.ml</code>, and <code class="language-plaintext highlighter-rouge">Opam_version.of_string</code>.</li>
  <li>Drops the <code class="language-plaintext highlighter-rouge">`V2_0</code> empty suffix special case in <code class="language-plaintext highlighter-rouge">Variant.pp</code> as all variants print with an explicit <code class="language-plaintext highlighter-rouge">_opam-X.X</code> suffix.</li>
  <li>Removed the now redundant <code class="language-plaintext highlighter-rouge">Opam_version.default</code> and made <code class="language-plaintext highlighter-rouge">Variant.of_string</code> fail for unsuffixed inputs.</li>
  <li>Delete <code class="language-plaintext highlighter-rouge">Opam_version.to_string_with_patch</code> which was declared in the <code class="language-plaintext highlighter-rouge">.mli</code> and dutifully updated by subsequent PRs but never called.</li>
</ul>

<h1>Merging</h1>

<p>The merge order followed the dependency chain in reverse.</p>

<ol>
  <li>ocurrent/opam-repo-ci#473 and ocurrent/ocaml-ci#1054 merged on May 4 removing the requirement for 2.0 images.</li>
  <li>ocurrent/ocaml-dockerfile#262 merged later the same day, and 8.3.8 was tagged.</li>
  <li>ocaml/opam-repository#29843 released dockerfile.8.3.8 merged on May 5.</li>
  <li>ocurrent/docker-base-images#342 could finally proceed now 8.3.8 was available.</li>
</ol>

<h1>Solver timeout</h1>

<p><a href="https://github.com/ocurrent/docker-base-images/pull/342">ocurrent/docker-base-images#342</a> had been rebased onto the current master to clear the CI errors, but the opam-repository SHA in the <code class="language-plaintext highlighter-rouge">Dockerfile</code> needed to be advanced past the 8.3.8 release, and <code class="language-plaintext highlighter-rouge">builds.expected</code> had to be regenerated to drop the opam-2.0 build steps. Trivial changes, but the CI still failed!</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[ERROR] Sorry, resolution of the request timed out.
        ... (currently, it is set to 600.0 seconds).
process "/bin/sh -c opam install -y --deps-only ." did not complete successfully: exit code: 60
</code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">opam install</code> step was timing out at the full 10-minute <code class="language-plaintext highlighter-rouge">OPAMSOLVERTIMEOUT</code>. Locally, the same <code class="language-plaintext highlighter-rouge">base-images.opam</code> solved under a second using <a href="https://github.com/mtelvers/day10">mtelvers/day10</a>, so this was not a hard graph. Both opam and day10 nominally use the 0install solver, so an identical input should produce an identical solve.</p>

<p>A quick <code class="language-plaintext highlighter-rouge">opam config report</code> inside the base image revealed the answer:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code># solver               builtin-mccs+glpk
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">opam</code> and therefore the base image <code class="language-plaintext highlighter-rouge">ocaml/opam:debian-ocaml-4.14</code> defaults to <code class="language-plaintext highlighter-rouge">builtin-mccs+glpk</code>, not <code class="language-plaintext highlighter-rouge">builtin-0install</code>. The fix was a one-line addition to the <code class="language-plaintext highlighter-rouge">Dockerfile</code>:</p>

<div class="language-dockerfile highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">RUN </span>opam option <span class="nt">--global</span> <span class="nv">solver</span><span class="o">=</span>builtin-0install
</code></pre></div></div>

<p>The same <code class="language-plaintext highlighter-rouge">opam install</code> step was then completed in ~100 seconds. The now-redundant <code class="language-plaintext highlighter-rouge">OPAMSOLVERTIMEOUT=600</code> came out at the same time. With the CI green, #342 was merged.</p>

<h1>Testing</h1>

<p>Before relying on the production pipeline to start producing the new images, I wanted to confirm the post-removal output actually works for a CI test. <code class="language-plaintext highlighter-rouge">builds.expected</code> made this straightforward: it’s a verbatim record of every Dockerfile docker-base-images would emit, so I could lift the relevant stages and rebuild them locally without waiting for a registry push.</p>

<p>Three stages, condensed into a single multi-stage Dockerfile:</p>

<ol>
  <li>The opam-binaries build stage (<code class="language-plaintext highlighter-rouge">debian:13</code> plus <code class="language-plaintext highlighter-rouge">git clone https://github.com/ocaml/opam</code>, then six <code class="language-plaintext highlighter-rouge">./configure &amp;&amp; make</code> invocations against branches <code class="language-plaintext highlighter-rouge">2.1</code>, <code class="language-plaintext highlighter-rouge">2.2</code>, <code class="language-plaintext highlighter-rouge">2.3</code>, <code class="language-plaintext highlighter-rouge">2.4</code>, <code class="language-plaintext highlighter-rouge">2.5</code>, and <code class="language-plaintext highlighter-rouge">master</code>).</li>
  <li>The opam-image stage (<code class="language-plaintext highlighter-rouge">debian:13</code> again, <code class="language-plaintext highlighter-rouge">COPY --from=opam-build</code> for each binary, set up the <code class="language-plaintext highlighter-rouge">opam</code> user, sandboxing scripts, and <code class="language-plaintext highlighter-rouge">opam init -k git -a /home/opam/opam-repository --bare</code>).</li>
  <li>The OCaml-5.4.1 stage (<code class="language-plaintext highlighter-rouge">opam switch create 5.4 --packages=ocaml-base-compiler.5.4.1</code>, <code class="language-plaintext highlighter-rouge">apt install libzstd-dev</code>).</li>
</ol>

<p>Built with my local opam-repository as the build context and tagged <code class="language-plaintext highlighter-rouge">local/debian-13-ocaml-5.4:test</code>.</p>

<p>Then I ran a real opam-repo-ci reproduction against it: <a href="https://github.com/ocaml/opam-repository/pull/29869">opam-repository#29869</a>, the release of <code class="language-plaintext highlighter-rouge">ca-certs.1.0.3</code>. opam-repo-ci publishes a script-style “to reproduce locally, do this” recipe per build; I took that verbatim and changed only the <code class="language-plaintext highlighter-rouge">FROM</code> line to point at the local tag:</p>

<div class="language-dockerfile highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">FROM</span><span class="s"> local/debian-13-ocaml-5.4:test</span>
<span class="k">USER</span><span class="s"> 1000:1000</span>
<span class="k">WORKDIR</span><span class="s"> /home/opam</span>
<span class="k">RUN </span><span class="nb">sudo ln</span> <span class="nt">-f</span> /usr/bin/opam-dev /usr/bin/opam
<span class="k">RUN </span>opam init <span class="nt">--reinit</span> <span class="nt">-ni</span>
<span class="k">RUN </span>opam option <span class="nv">solver</span><span class="o">=</span>builtin-0install <span class="o">&amp;&amp;</span> opam config report
...
<span class="k">RUN </span>opam pin add <span class="nt">-k</span> version <span class="nt">-yn</span> ca-certs.1.0.3 1.0.3
<span class="k">RUN </span>opam reinstall ca-certs.1.0.3<span class="p">;</span> ...
</code></pre></div></div>

<p>Note <code class="language-plaintext highlighter-rouge">opam-dev</code> (i.e. opam master) is what the recipe links as <code class="language-plaintext highlighter-rouge">/usr/bin/opam</code> — the new image still ships it, just without <code class="language-plaintext highlighter-rouge">opam-2.0</code> alongside.</p>

<p>Result:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>-&gt; installed bos.0.3.0
-&gt; installed dune.3.23.0
-&gt; installed mirage-crypto.2.1.0
...
-&gt; installed x509.1.0.6
-&gt; installed ca-certs.1.0.3
Done.
DONE 53.1s
</code></pre></div></div>

<p>The new base images will be rebuilt over the weekend, and the CI systems will pick them up soon after.</p>
