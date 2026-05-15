---
title: "Prefetch opam files for day10 \u2013fork"
description: Last month, I wrote a walkthrough on using mtelvers/day10 and while stuck
  in traffic yesterday, I was thinking about all those individual opam files which
  are read for every solve.
url: https://www.tunbury.org/2026/04/21/day10-context-backends/
date: 2026-04-21T19:00:00-00:00
preview_image: https://www.tunbury.org/images/opam.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Last month, I <a href="https://www.tunbury.org/2026/03/16/day10/">wrote</a> a walkthrough on using <a href="https://github.com/mtelvers/day10">mtelvers/day10</a> and while stuck in traffic yesterday, I was thinking about all those individual opam files which are read for every solve.</p>

<p><code class="language-plaintext highlighter-rouge">day10</code>’s solver is built around <code class="language-plaintext highlighter-rouge">Opam_0install.Solver.Make(Dir_context)</code>, and <code class="language-plaintext highlighter-rouge">Dir_context</code> reads opam files directly from an <code class="language-plaintext highlighter-rouge">opam-repository</code> working tree. For a typical package such as <code class="language-plaintext highlighter-rouge">0install.2.18</code>, I was seeing ~0.79s per solve, and on my machine , it showed 3m30s of user time for 200 packages at <code class="language-plaintext highlighter-rouge">--fork 10</code>. The wall time was 23.6s.</p>

<p>A quick check showed that <a href="https://github.com/ocaml-opam/opam-0install-solver">ocaml-opam/opam-0install-solver</a> wasn’t far off the installed upstream <code class="language-plaintext highlighter-rouge">Opam_0install.Dir_context</code> (same ~0.80s). The upstream <code class="language-plaintext highlighter-rouge">opam-0install</code> CLI reports 0.24s per solve, but that uses <code class="language-plaintext highlighter-rouge">Switch_context</code> over a pre-loaded opam switch state, so all the opam files are already parsed.</p>

<h1>Git object backends</h1>

<p>My original thought was to use the git database directly, then multiple instances of <code class="language-plaintext highlighter-rouge">day10</code> could read different commits simultaneously without needing a working tree. Git’s object database is content-addressed and fine with multiple concurrent readers, and Thomas uses the git store in the <a href="https://github.com/ocurrent/solver-service">ocurrent/solver-service</a>.</p>

<p>I tried four backends on the same solve (<code class="language-plaintext highlighter-rouge">0install.2.18</code> against my opam-repository HEAD, <code class="language-plaintext highlighter-rouge">caab044f22</code>).</p>

<table>
  <thead>
    <tr>
      <th>backend</th>
      <th>solve time</th>
      <th>notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">Dir_context</code> (filesystem, baseline)</td>
      <td>0.79s</td>
      <td>re-parses every opam file on every solve</td>
    </tr>
    <tr>
      <td>subprocess <code class="language-plaintext highlighter-rouge">git cat-file --batch</code> (per-query)</td>
      <td>9.27s</td>
      <td>one roundtrip per opam blob</td>
    </tr>
    <tr>
      <td>subprocess + Hashtbl cache</td>
      <td>6.42s</td>
      <td>dropped redundant probes too</td>
    </tr>
    <tr>
      <td><strong>ocaml-git</strong> (native OCaml, <code class="language-plaintext highlighter-rouge">git-unix</code> + packfile reader)</td>
      <td>42.57s</td>
      <td>packfile inflate + delta resolution per object</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">Sql_context</code> (SQLite ingestion)</td>
      <td>0.77s</td>
      <td>75 MB DB, 0.73s one-off ingest</td>
    </tr>
  </tbody>
</table>

<p>The <code class="language-plaintext highlighter-rouge">git cat-file --batch</code> subprocess version was OK-ish, but six seconds per solve wasn’t going to win any awards. <code class="language-plaintext highlighter-rouge">ocaml-git</code> was by far the worst, as every <code class="language-plaintext highlighter-rouge">Search.find</code> walks from commit to tree to sub-tree to blob, and each hop hits a zlib inflate. Caching the results took it from unusably slow to only very slow. It also drags in lwt + mimic + carton + decompress + digestif + happy-eyeballs + tls.</p>

<p>SQLite was the surprise winner among the new backends. One sequential-scan prepared statement plus a blob lookup per <code class="language-plaintext highlighter-rouge">load</code> matched <code class="language-plaintext highlighter-rouge">Dir_context</code> almost exactly.</p>

<p>How did Thomas do it in <a href="https://github.com/ocurrent/solver-service">ocurrent/solver-service</a>? It does use <code class="language-plaintext highlighter-rouge">ocaml-git</code>, but it defers almost all of the cost to an <a href="https://github.com/ocaml-multicore/eio"><code class="language-plaintext highlighter-rouge">Eio.Lazy</code></a> on a per package name basis:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">type</span> <span class="n">t</span> <span class="o">=</span> <span class="nn">OpamFile</span><span class="p">.</span><span class="nn">OPAM</span><span class="p">.</span><span class="n">t</span> <span class="nn">OpamPackage</span><span class="p">.</span><span class="nn">Version</span><span class="p">.</span><span class="nn">Map</span><span class="p">.</span><span class="n">t</span>
         <span class="nn">Eio</span><span class="p">.</span><span class="nn">Lazy</span><span class="p">.</span><span class="n">t</span>
         <span class="nn">OpamPackage</span><span class="p">.</span><span class="nn">Name</span><span class="p">.</span><span class="nn">Map</span><span class="p">.</span><span class="n">t</span>
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">of_commit</code> reads exactly one tree object, the top-level <code class="language-plaintext highlighter-rouge">packages/</code> directory, and records each name’s subtree SHA. No opam files are touched at startup. When the solver asks for candidates of <code class="language-plaintext highlighter-rouge">lwt</code>, the lazy for <code class="language-plaintext highlighter-rouge">lwt</code> is forced: it reads the <code class="language-plaintext highlighter-rouge">packages/lwt/</code> subtree and, in one batch, reads and parses every version’s opam file. That result is then cached. Second and subsequent <code class="language-plaintext highlighter-rouge">candidates(lwt)</code> calls are map lookups.</p>

<p>The solver’s access pattern is all versions of a single package name, so I ported this idea to the plain <code class="language-plaintext highlighter-rouge">cat-file --batch</code> subprocess, and it ran the first solve in 1.1s and subsequent solves in 0.25s. That’s pretty close to the <code class="language-plaintext highlighter-rouge">Switch_context</code> on warm solves.</p>

<p>Full scoreboard after all this:</p>

<table>
  <thead>
    <tr>
      <th>approach</th>
      <th>setup</th>
      <th>solve</th>
      <th>total</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>upstream CLI (<code class="language-plaintext highlighter-rouge">Switch_context</code>)</td>
      <td>0.37s</td>
      <td>0.24s</td>
      <td>0.61s</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">Sql_context</code></td>
      <td>—</td>
      <td>0.77s</td>
      <td>0.77s</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">Dir_context</code></td>
      <td>—</td>
      <td>0.79s</td>
      <td>0.79s</td>
    </tr>
    <tr>
      <td>bulk <code class="language-plaintext highlighter-rouge">git archive</code> + tar, lazy parse</td>
      <td>0.59s</td>
      <td>0.73s</td>
      <td>1.33s</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">git cat-file</code>, lazy per package name</td>
      <td>0.03s</td>
      <td>1.10s</td>
      <td>1.13s</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">git cat-file</code>, per query</td>
      <td>—</td>
      <td>6.42s</td>
      <td>6.42s</td>
    </tr>
    <tr>
      <td>ocaml-git + caches</td>
      <td>—</td>
      <td>42.57s</td>
      <td>42.57s</td>
    </tr>
  </tbody>
</table>

<h1>Actually addressing the problem</h1>

<p>All of the above was interesting, but I concluded that it was largely irrelevant to <code class="language-plaintext highlighter-rouge">day10 --fork 256</code>. The real workload isn’t one cold solve; it’s <a href="https://www.tunbury.org/2026/03/16/day10/#step-2-solve-dry-run-pass">thousands of solves per invocation</a>, more like the solver service.</p>

<p>The <code class="language-plaintext highlighter-rouge">--fork</code> parameter in <code class="language-plaintext highlighter-rouge">day10</code> does <code class="language-plaintext highlighter-rouge">Os.fork ~np run_with_package packages</code>. Each of the 256 children starts with its own empty context, reads opam files off disk, parses them, caches nothing across solves. Each child does the same parsing work. Both the duplicated work and the fact that they are forks mean they can’t efficiently share a cache.</p>

<p>However, Linux’s <code class="language-plaintext highlighter-rouge">fork</code> is copy-on-write, so if the parent process parses all the opam files before the fork, every child inherits the parsed <code class="language-plaintext highlighter-rouge">OpamFile.OPAM.t</code> values for free via shared pages.</p>

<p>The smallest possible change: a process-wide <code class="language-plaintext highlighter-rouge">Hashtbl</code> in <code class="language-plaintext highlighter-rouge">Dir_context</code>, and a <code class="language-plaintext highlighter-rouge">prefetch</code> function that populates it:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">cache</span> <span class="o">:</span> <span class="p">(</span><span class="nn">OpamPackage</span><span class="p">.</span><span class="n">t</span><span class="o">,</span> <span class="nn">OpamFile</span><span class="p">.</span><span class="nn">OPAM</span><span class="p">.</span><span class="n">t</span><span class="p">)</span> <span class="nn">Hashtbl</span><span class="p">.</span><span class="n">t</span> <span class="o">=</span> <span class="nn">Hashtbl</span><span class="p">.</span><span class="n">create</span> <span class="mi">32768</span>

<span class="k">let</span> <span class="n">load</span> <span class="n">t</span> <span class="n">pkg</span> <span class="o">=</span>
  <span class="k">let</span> <span class="p">{</span> <span class="nn">OpamPackage</span><span class="p">.</span><span class="n">name</span><span class="p">;</span> <span class="n">_</span> <span class="p">}</span> <span class="o">=</span> <span class="n">pkg</span> <span class="k">in</span>
  <span class="k">match</span> <span class="nn">OpamPackage</span><span class="p">.</span><span class="nn">Name</span><span class="p">.</span><span class="nn">Map</span><span class="p">.</span><span class="n">find_opt</span> <span class="n">name</span> <span class="n">t</span><span class="o">.</span><span class="n">pins</span> <span class="k">with</span>
  <span class="o">|</span> <span class="nc">Some</span> <span class="p">(</span><span class="n">_</span><span class="o">,</span> <span class="n">opam</span><span class="p">)</span> <span class="o">-&gt;</span> <span class="n">opam</span>
  <span class="o">|</span> <span class="nc">None</span> <span class="o">-&gt;</span>
      <span class="k">match</span> <span class="nn">Hashtbl</span><span class="p">.</span><span class="n">find_opt</span> <span class="n">cache</span> <span class="n">pkg</span> <span class="k">with</span>
      <span class="o">|</span> <span class="nc">Some</span> <span class="n">v</span> <span class="o">-&gt;</span> <span class="n">v</span>
      <span class="o">|</span> <span class="nc">None</span> <span class="o">-&gt;</span>
          <span class="k">let</span> <span class="n">opam</span> <span class="o">=</span> <span class="o">...</span> <span class="n">read</span> <span class="ow">and</span> <span class="n">parse</span> <span class="n">from</span> <span class="n">filesystem</span> <span class="o">...</span> <span class="k">in</span>
          <span class="nn">Hashtbl</span><span class="p">.</span><span class="n">add</span> <span class="n">cache</span> <span class="n">pkg</span> <span class="n">opam</span><span class="p">;</span>
          <span class="n">opam</span>

<span class="k">let</span> <span class="n">prefetch</span> <span class="o">~</span><span class="n">packages_dirs</span> <span class="bp">()</span> <span class="o">=</span>
  <span class="o">...</span> <span class="n">walk</span> <span class="n">every</span> <span class="n">packages</span><span class="o">/&lt;</span><span class="n">name</span><span class="o">&gt;/&lt;</span><span class="n">name</span><span class="o">.</span><span class="n">ver</span><span class="o">&gt;/</span><span class="n">opam</span><span class="o">,</span> <span class="n">parse</span><span class="o">,</span> <span class="n">stash</span> <span class="k">in</span> <span class="n">the</span> <span class="n">cache</span> <span class="o">...</span>
</code></pre></div></div>

<p>And in <code class="language-plaintext highlighter-rouge">run_health_check_multi</code>’s fork branch:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">|</span> <span class="nc">Some</span> <span class="n">np</span> <span class="o">-&gt;</span>
    <span class="k">let</span> <span class="n">packages_dirs</span> <span class="o">=</span> <span class="nn">List</span><span class="p">.</span><span class="n">map</span> <span class="p">(</span><span class="k">fun</span> <span class="n">r</span> <span class="o">-&gt;</span> <span class="nn">Path</span><span class="p">.(</span><span class="n">r</span> <span class="o">/</span> <span class="s2">"packages"</span><span class="p">))</span> <span class="n">config</span><span class="o">.</span><span class="n">opam_repositories</span> <span class="k">in</span>
    <span class="nn">Dir_context</span><span class="p">.</span><span class="n">prefetch</span> <span class="o">~</span><span class="n">packages_dirs</span> <span class="bp">()</span><span class="p">;</span>
    <span class="nn">Os</span><span class="p">.</span><span class="n">fork</span> <span class="o">~</span><span class="n">np</span> <span class="n">run_with_package</span> <span class="n">packages</span>
</code></pre></div></div>

<p>The non-fork paths (<code class="language-plaintext highlighter-rouge">Some 1 | None</code>) keep the existing behaviour.</p>

<h1>Results</h1>

<p>Running on the same EPYC 9965 box as the <a href="https://www.tunbury.org/2026/03/16/day10/">original post</a>, with <code class="language-plaintext highlighter-rouge">--fork 256</code>, 4,325 packages at ocaml 5.4.1 / Debian 13 / <code class="language-plaintext highlighter-rouge">--dry-run</code>:</p>

<table>
  <thead>
    <tr>
      <th>run</th>
      <th>wall</th>
      <th>user CPU</th>
      <th>sys</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>baseline</td>
      <td>36.0s</td>
      <td>25m 50s</td>
      <td>98m 19s</td>
    </tr>
    <tr>
      <td>pre-parsed before fork</td>
      <td>10.0s</td>
      <td>7m 10s</td>
      <td>2m 26s</td>
    </tr>
    <tr>
      <td>speedup</td>
      <td>3.6x</td>
      <td>3.6x</td>
      <td>40x</td>
    </tr>
  </tbody>
</table>

<p>The baseline figure matches the 36s I reported originally, which is reassuring!</p>

<p>Wall time drops from 36s to 10s, which I’m pretty happy with, particularly set against the reductions in CPU time. The prefetch step itself took 1.3s on this machine, which is a small price to offset the ~90 minutes of eliminated sys-time IO.</p>
