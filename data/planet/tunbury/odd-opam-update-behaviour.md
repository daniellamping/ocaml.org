---
title: Odd opam update behaviour
description: "A few days after retiring opam 2.0 from the build pipeline, ocaml-ci
  Jon noticed that some jobs were failing. I immediately concluded that the removal
  was to blame, but it wasn\u2019t."
url: https://www.tunbury.org/2026/05/12/odd-opam-update/
date: 2026-05-12T18:00:00-00:00
preview_image: https://www.tunbury.org/images/opam.png
authors:
- Mark Elvers
source:
ignore:
---

<p>A few days after <a href="https://www.tunbury.org/2026/05/07/removing-opam-2.0/">retiring opam 2.0 from the build pipeline</a>, <a href="https://ocaml.ci.dev">ocaml-ci</a> Jon noticed that some jobs were failing. I immediately concluded that the removal was to blame, but it wasn’t.</p>

<p>Most builds were fine, but some failed to install packages identified as dependencies during the solver step. Strangely, the file was right there in the opam-repository commit we had pinned, so why couldn’t opam see it?</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[ERROR] Package base has no version v0.16.5.
</code></pre></div></div>

<p>This post walks through tracking the bug from a contradiction in the build log to a change in opam 2.4 that makes a fast-forward <code class="language-plaintext highlighter-rouge">opam update -u</code> ineffective when it follows <code class="language-plaintext highlighter-rouge">opam init --reinit</code> step.</p>

<h1>How ocaml-ci pins opam-repository</h1>

<p><a href="https://github.com/ocurrent/ocaml-ci">ocurrent/ocaml-ci</a> drives each build from a solver-computed plan. Its pipeline:</p>

<ol>
  <li>Tracks the current head of <code class="language-plaintext highlighter-rouge">ocaml/opam-repository</code> master.</li>
  <li>Sends that commit, plus the project’s <code class="language-plaintext highlighter-rouge">.opam</code> files and per-platform variables, to the <a href="https://github.com/ocurrent/solver-service">solver service</a>.</li>
  <li>Receives back a list of package versions that satisfy the project at that opam-repository commit.</li>
  <li>Emits an <a href="https://github.com/ocurrent/obuilder">OBuilder</a> spec that, inside the build container, resets <code class="language-plaintext highlighter-rouge">~/opam-repository</code> to that commit, runs <code class="language-plaintext highlighter-rouge">opam install --depext</code>, then <code class="language-plaintext highlighter-rouge">opam install</code>.</li>
</ol>

<p>The solver and the build container are supposed to see the same opam-repository state. If the solver picks <code class="language-plaintext highlighter-rouge">base.v0.16.5</code>, the build should be able to install it.</p>

<h1>The failing job</h1>

<p>Jon’s failing job was <a href="https://github.com/ocaml/odoc/pull/1424"><code class="language-plaintext highlighter-rouge">ocaml/odoc#1424</code></a>, and I picked this variant <code class="language-plaintext highlighter-rouge">linux-ppc64:debian-13-4.14_ppc64_opam-2.5</code>. The relevant fragment of the build log:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>RUN cd ~/opam-repository &amp;&amp; (git cat-file -e &lt;SHA&gt; || git fetch origin master) \
    &amp;&amp; git reset -q --hard &lt;SHA&gt; &amp;&amp; git log --no-decorate -n1 --oneline \
    &amp;&amp; opam update -u
From https://github.com/ocaml/opam-repository
 * branch                  master     -&gt; FETCH_HEAD
   95972b8834..773d384256  master     -&gt; origin/master
29a3156587 Merge pull request #29871 from Leonidas-from-XIV/openbsd-jq

&lt;&gt;&lt;&gt; Updating package repositories &gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;&lt;&gt;
[default] synchronised from git+file:///home/opam/opam-repository
Everything as up-to-date as possible (run with --verbose to show unavailable upgrades).
Nothing to do.
</code></pre></div></div>

<p>So the container fetched upstream master, reset to <code class="language-plaintext highlighter-rouge">29a31565...</code>, and <code class="language-plaintext highlighter-rouge">opam update -u</code> reported that the <code class="language-plaintext highlighter-rouge">default</code> remote was synchronised. Then, after a few <code class="language-plaintext highlighter-rouge">opam pin</code>s, <code class="language-plaintext highlighter-rouge">opam install --cli=2.5 --depext-only -y ...</code>, then said:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[ERROR] Package base has no version v0.16.5.
</code></pre></div></div>

<p>Two questions come to mind from this:</p>

<ol>
  <li>Was <code class="language-plaintext highlighter-rouge">base.v0.16.5</code> actually in that commit?</li>
  <li>If yes, what did <code class="language-plaintext highlighter-rouge">opam update -u</code> not actually update?</li>
</ol>

<h1>Step 1: confirming the version exists</h1>

<p>Cloned <code class="language-plaintext highlighter-rouge">ocaml/opam-repository</code> locally and asked git:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>$ git show 29a31565874bc1a23f438f21575c5e5cbe087068:packages/base/base.v0.16.5/opam | head
opam-version: "2.0"
maintainer: "Jane Street developers"
...
depends: [
  "ocaml"             {&gt;= "4.14.0"}
  "sexplib0"          {&gt;= "v0.16" &amp; &lt; "v0.17"}
  ...
]
</code></pre></div></div>

<p>So the solver was correct. The version exists at the pinned commit, has no exotic <code class="language-plaintext highlighter-rouge">available:</code> filter, so it should install fine. The bug was elsewhere.</p>

<h1>Step 2: a clue from the base image</h1>

<p>The build started from <code class="language-plaintext highlighter-rouge">ocaml/opam@sha256:d34d8012...</code>. That image’s bundled <code class="language-plaintext highlighter-rouge">~/opam-repository</code> had been baked at SHA <code class="language-plaintext highlighter-rouge">95972b8834</code>, which was 474 commits behind the target SHA. The container’s first move was to fast-forward the local clone, and then run <code class="language-plaintext highlighter-rouge">opam update -u</code>. That should be enough; <code class="language-plaintext highlighter-rouge">opam update</code> is the command that re-reads opam-repository’s package set.</p>

<p>The base image was created before the release of <code class="language-plaintext highlighter-rouge">base.v0.16.5</code>, so the package isn’t in the bundled state. That’s why we reset and update. The interesting question is what changed such that this used to work and now doesn’t: the spec ordering hasn’t changed, and base images have always lagged master to some extent.</p>

<h1>Step 3: reproducing on real hardware</h1>

<p>The failing job ran on <code class="language-plaintext highlighter-rouge">orithia.caelum.ci.dev</code>, a ppc64le worker. ssh in, replay the exact spec inside the exact base image:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>docker run --rm ocaml/opam@sha256:d34d8012... bash -ec '
  sudo ln -f /usr/bin/opam-2.5 /usr/bin/opam
  opam init --reinit -ni
  cd ~/opam-repository
  ( git cat-file -e &lt;SHA&gt; || git fetch origin master )
  git reset -q --hard &lt;SHA&gt;
  opam update -u
  opam show base.v0.16.5
'
</code></pre></div></div>

<p>Output:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[ERROR] No package matching base.v0.16.5 found
</code></pre></div></div>

<p>Reproduced. Now compare the same flow with the <code class="language-plaintext highlighter-rouge">git reset</code> moved before the <code class="language-plaintext highlighter-rouge">opam init --reinit</code>:</p>

<table>
  <thead>
    <tr>
      <th>Scenario</th>
      <th>Order</th>
      <th><code class="language-plaintext highlighter-rouge">base.v0.16.5</code> visible?</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>current spec</td>
      <td><code class="language-plaintext highlighter-rouge">opam init --reinit</code> -&gt; <code class="language-plaintext highlighter-rouge">git reset</code> -&gt; <code class="language-plaintext highlighter-rouge">opam update -u</code></td>
      <td>failed</td>
    </tr>
    <tr>
      <td>reset first</td>
      <td><code class="language-plaintext highlighter-rouge">git reset</code> -&gt; <code class="language-plaintext highlighter-rouge">opam init --reinit</code> -&gt; <code class="language-plaintext highlighter-rouge">opam update -u</code></td>
      <td>success</td>
    </tr>
    <tr>
      <td>extra update</td>
      <td>…spec A… then an extra <code class="language-plaintext highlighter-rouge">opam update default</code></td>
      <td>success</td>
    </tr>
  </tbody>
</table>

<p>When <code class="language-plaintext highlighter-rouge">opam init --reinit -ni</code> records repository state, the subsequent <code class="language-plaintext highlighter-rouge">git reset &amp;&amp; opam update -u</code> isn’t replacing it. <code class="language-plaintext highlighter-rouge">opam show base.v0.16.5</code> still can’t find the package.</p>

<h1>Step 4: which opam version changed this?</h1>

<p>The spec’s order has lived in <a href="https://github.com/ocurrent/ocaml-ci/blob/master/lib/opam_build.ml"><code class="language-plaintext highlighter-rouge">lib/opam_build.ml</code></a> since <a href="https://github.com/ocurrent/ocaml-ci/commit/3087a3f">commit <code class="language-plaintext highlighter-rouge">3087a3f</code></a> in December 2022 (“Changing default opam version requires reinit”). It clearly worked for years. Recently, we’d bumped the default opam used by builds: <code class="language-plaintext highlighter-rouge">V2_4</code> last November, <code class="language-plaintext highlighter-rouge">V2_5</code> in January. Let’s pin everything else and step through opam binaries:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>for V in 2.2 2.3 2.4 2.5; do
  docker run --rm ocaml/opam@sha256:d34d8012... bash -ec '
    sudo ln -f /usr/bin/opam-'"$V"' /usr/bin/opam
    opam init --reinit -ni
    cd ~/opam-repository
    ( git cat-file -e &lt;SHA&gt; || git fetch origin master )
    git reset -q --hard &lt;SHA&gt;
    opam update -u
    opam show base.v0.16.5 2&gt;&amp;1 | head -2
  '
done
</code></pre></div></div>

<table>
  <thead>
    <tr>
      <th>opam</th>
      <th>format upgrade</th>
      <th><code class="language-plaintext highlighter-rouge">base.v0.16.5</code> visible after <code class="language-plaintext highlighter-rouge">opam update -u</code>?</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>2.2.1</td>
      <td><code class="language-plaintext highlighter-rouge">.opam</code> 2.0 -&gt; 2.2</td>
      <td>success</td>
    </tr>
    <tr>
      <td>2.3.0</td>
      <td><code class="language-plaintext highlighter-rouge">.opam</code> 2.0 -&gt; 2.2</td>
      <td>success</td>
    </tr>
    <tr>
      <td>2.4.1</td>
      <td><code class="language-plaintext highlighter-rouge">.opam</code> 2.0 -&gt; 2.2</td>
      <td>failure</td>
    </tr>
    <tr>
      <td>2.5.0</td>
      <td><code class="language-plaintext highlighter-rouge">.opam</code> 2.0 -&gt; 2.2</td>
      <td>failure</td>
    </tr>
  </tbody>
</table>

<p>Every version goes through the same on-disk <code class="language-plaintext highlighter-rouge">.opam</code> format upgrade. But the visibility regression lands in opam 2.4.</p>

<h1>Step 5: what changed between opam 2.3 and 2.4</h1>

<p>The version-by-version test pins the change on opam 2.4. Opam’s <a href="https://github.com/ocaml/opam/blob/master/CHANGES">CHANGES</a> file has several entries in the 2.4 development cycle that rework how repositories are loaded and updated.</p>

<p>These changes share a theme: diffing rather than re-reading, and incremental loading instead of a full re-parse. That theme matches what we observe. <code class="language-plaintext highlighter-rouge">opam init --reinit</code> evidently records some view of the repository, and <code class="language-plaintext highlighter-rouge">opam update -u</code> then diffs against it. There are also marshalled <code class="language-plaintext highlighter-rouge">~/.opam/repo/state-XXXX.cache</code> files, which I suspect have some involvement in this.</p>

<p>The experiment does prove that reordering the steps so init runs after the reset side-steps the failure on every opam version we tested.</p>

<h1>Why didn’t this surface in November?</h1>

<p>The spec ordering has been the same since 2022. The default opam version used by builds was raised to 2.4 in November 2025 (in <a href="https://github.com/ocurrent/ocaml-ci/commit/a993262">commit <code class="language-plaintext highlighter-rouge">a993262</code></a>). From then on, jobs running on a base image whose bundled <code class="language-plaintext highlighter-rouge">~/opam-repository</code> lagged the solver’s target commit by enough to be missing a chosen package would have hit this. In most cases, this isn’t a problem, but this specific job highlighted the problem.</p>

<p>Most jobs evidently haven’t been that unlucky. Either their base image was recent enough, or the packages the solver picked were old enough to predate the bundled state. The bug was latent for months before this particular combination of image and freshly-added Jane Street version surfaced it.</p>

<h1>Summary and fix</h1>

<ol>
  <li><code class="language-plaintext highlighter-rouge">opam init --reinit -ni</code> runs while <code class="language-plaintext highlighter-rouge">~/opam-repository</code> is still at the base image’s bundled, stale commit.</li>
  <li>The build container then fast-forwards <code class="language-plaintext highlighter-rouge">~/opam-repository</code> to the solver-selected commit and runs <code class="language-plaintext highlighter-rouge">opam update -u</code>.</li>
  <li>With opam 2.4 or 2.5, the package set served to subsequent <code class="language-plaintext highlighter-rouge">opam install</code> reflects the repository as it was at step 1, not after the reset.</li>
  <li><code class="language-plaintext highlighter-rouge">opam install</code> doesn’t see packages added in the intervening commits and fails with <code class="language-plaintext highlighter-rouge">[ERROR] Package base has no version v0.16.5.</code></li>
</ol>

<p>The fix is one step’s worth of reordering: do the <code class="language-plaintext highlighter-rouge">git fetch</code> and <code class="language-plaintext highlighter-rouge">git reset</code> before <code class="language-plaintext highlighter-rouge">opam init --reinit</code>. Then init runs against the correct repository state from the start, and the trailing <code class="language-plaintext highlighter-rouge">opam update -u</code> becomes essentially a no-op.</p>

<p><a href="https://github.com/ocurrent/ocaml-ci/pull/1055">ocurrent/ocaml-ci#1055</a> makes that change in <code class="language-plaintext highlighter-rouge">lib/opam_build.ml</code> and updates the snapshot tests in <code class="language-plaintext highlighter-rouge">test/service/test_spec.ml</code> to match the new step order.</p>
