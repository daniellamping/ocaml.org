---
title: Update solver-service in OCaml-CI local mode
description: "When I (mostly) unvendored ocaml-ci\u2019s submodules a few days ago.
  Four out of the five were published in the opam-repository, but solver-service was
  not, so it ended up as a pin-depends block in ocaml-ci.opam.template pinned at the
  same SHA the submodule had pointed at."
url: https://www.tunbury.org/2026/04/30/ocaml-ci-solver-service/
date: 2026-04-30T21:00:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>When I (mostly) <a href="https://www.tunbury.org/2026/04/29/ocaml-ci-update/">unvendored ocaml-ci’s submodules</a> a few days ago. Four out of the five were published in the opam-repository, but <code class="language-plaintext highlighter-rouge">solver-service</code> was not, so it ended up as a <code class="language-plaintext highlighter-rouge">pin-depends</code> block in <code class="language-plaintext highlighter-rouge">ocaml-ci.opam.template</code> pinned at the same SHA the submodule had pointed at.</p>

<p>Patrick’s <a href="https://github.com/ocurrent/ocaml-ci/issues/1044">Issue #1044</a> caused me to revisit it. <code class="language-plaintext highlighter-rouge">ocaml-ci-local /path/to/repo</code> fails the analysis step with <code class="language-plaintext highlighter-rouge">Invalid_argument("filter_deps")</code>. The upstream fix is <a href="https://github.com/ocurrent/solver-service/commit/86d37c716be36c0712dcc2be60b135497ead1132"><code class="language-plaintext highlighter-rouge">86d37c7</code></a>, a one line in <code class="language-plaintext highlighter-rouge">service/git_context.ml</code>:</p>

<div class="language-diff highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gd">-  |&gt; OpamFilter.filter_deps ~build:true ~post:true ~test ~doc:false ~dev
</span><span class="gi">+  |&gt; OpamFilter.filter_deps ~build:true ~post:true ~test ~doc:false ~dev ~dev_setup:false
</span>        ~default:false
</code></pre></div></div>

<p>Newer opam-format raises <code class="language-plaintext highlighter-rouge">Invalid_argument</code> from <code class="language-plaintext highlighter-rouge">OpamFilter.deps_var_env</code> if the filter formula references <code class="language-plaintext highlighter-rouge">with-dev-setup</code> and the caller hasn’t passed <code class="language-plaintext highlighter-rouge">~dev_setup</code>. I had <code class="language-plaintext highlighter-rouge">opam-format.2.5.1</code> installed; the bug was easily reproduced.</p>

<p>The problem is that the pin can’t be bumped as <code class="language-plaintext highlighter-rouge">f14bc6f</code> is the last commit before <code class="language-plaintext highlighter-rouge">12f49f6</code> “Initial OCaml 5 / Eio port” which changes everything:</p>

<ul>
  <li>OCaml &gt;= 5.1.0 (we still build OCaml-CI on 4.14 along with the rest of the stack is Lwt + OCurrent).</li>
  <li>Eio, <code class="language-plaintext highlighter-rouge">lwt_eio</code>, and an Eio_main-based <code class="language-plaintext highlighter-rouge">solver-service</code> binary.</li>
  <li><code class="language-plaintext highlighter-rouge">opam-core</code>/<code class="language-plaintext highlighter-rouge">opam-state</code>/<code class="language-plaintext highlighter-rouge">opam-repository</code>/<code class="language-plaintext highlighter-rouge">opam-format</code> pinned to <code class="language-plaintext highlighter-rouge">2.3.0~alpha1</code> for performance fixes #6144/#6122 — none released to opam-repository.</li>
  <li>The deletion of the <code class="language-plaintext highlighter-rouge">solver-worker</code> opam package, whose functionality is now in <code class="language-plaintext highlighter-rouge">solver-service</code>.</li>
  <li>The deletion of <code class="language-plaintext highlighter-rouge">Solver_worker.Solver_request</code> Lwt library which <code class="language-plaintext highlighter-rouge">lib/backend_solver.ml</code> calls:</li>
</ul>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">type</span> <span class="n">t</span> <span class="o">=</span>
  <span class="o">|</span> <span class="nc">Remote</span> <span class="k">of</span> <span class="nn">Current_ocluster</span><span class="p">.</span><span class="nn">Connection</span><span class="p">.</span><span class="n">t</span>
  <span class="o">|</span> <span class="nc">Local</span> <span class="k">of</span> <span class="nn">Solver_worker</span><span class="p">.</span><span class="nn">Solver_request</span><span class="p">.</span><span class="n">t</span> <span class="nn">Lwt</span><span class="p">.</span><span class="n">t</span>
</code></pre></div></div>

<p>The thing is, though, the <code class="language-plaintext highlighter-rouge">Local</code> development mode doesn’t run any solver code in the OCaml-CI process. It sets up a pool of child processes and sends solve requests to them through a pipe; the actual <code class="language-plaintext highlighter-rouge">OpamFilter.filter_deps</code> call is in an exec’d <code class="language-plaintext highlighter-rouge">solver-service</code> binary.</p>

<p>So there’s no reason I can’t have an Eio solver service binary exec’d by OCaml-CI using Lwt. This is just <code class="language-plaintext highlighter-rouge">Lwt_process.open_process</code>! The problem is that it’s untidy to build it this way, since the two components need to be built with different compilers and different dependencies.</p>

<p>Even accepting that, the new <code class="language-plaintext highlighter-rouge">solver-service run-child</code> needs a socket pair where the older <code class="language-plaintext highlighter-rouge">solver-service --sockpath</code> used a socket file:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* OLD: f14bc6f's --sockpath. Parent binds a UNIX socket file,
   exec's the child, child connects to the path, parent accepts. *)</span>
<span class="k">let</span> <span class="n">listener</span> <span class="o">=</span> <span class="nn">Unix</span><span class="p">.</span><span class="n">socket</span> <span class="o">~</span><span class="n">cloexec</span><span class="o">:</span><span class="bp">true</span> <span class="nc">PF_UNIX</span> <span class="nc">SOCK_STREAM</span> <span class="mi">0</span> <span class="k">in</span>
<span class="nn">Unix</span><span class="p">.</span><span class="n">bind</span> <span class="n">listener</span> <span class="p">(</span><span class="nc">ADDR_UNIX</span> <span class="n">name</span><span class="p">);</span>
<span class="nn">Unix</span><span class="p">.</span><span class="n">listen</span> <span class="n">listener</span> <span class="mi">1</span><span class="p">;</span>
<span class="k">let</span> <span class="n">cmd</span> <span class="o">=</span> <span class="p">(</span><span class="s2">""</span><span class="o">,</span> <span class="p">[</span><span class="o">|</span> <span class="s2">"solver-service"</span><span class="p">;</span> <span class="s2">"--sockpath"</span><span class="p">;</span> <span class="n">name</span> <span class="o">|</span><span class="p">])</span> <span class="k">in</span>
<span class="k">let</span> <span class="n">_child</span> <span class="o">=</span> <span class="nn">Lwt_process</span><span class="p">.</span><span class="n">open_process_none</span> <span class="o">~</span><span class="n">cwd</span><span class="o">:</span><span class="n">solver_dir</span> <span class="o">~</span><span class="n">stdin</span><span class="o">:</span><span class="nt">`Close</span> <span class="n">cmd</span> <span class="k">in</span>
<span class="k">let</span> <span class="n">p</span><span class="o">,</span> <span class="n">_</span> <span class="o">=</span> <span class="nn">Unix</span><span class="p">.</span><span class="n">accept</span> <span class="o">~</span><span class="n">cloexec</span><span class="o">:</span><span class="bp">true</span> <span class="n">listener</span> <span class="k">in</span>
<span class="nn">Unix</span><span class="p">.</span><span class="n">close</span> <span class="n">listener</span><span class="p">;</span>
<span class="nn">Unix</span><span class="p">.</span><span class="n">unlink</span> <span class="n">name</span><span class="p">;</span>

<span class="c">(* NEW: HEAD's run-child. Parent makes a socketpair, dups one half
   onto the child's stdin, exec's the child. *)</span>
<span class="k">let</span> <span class="n">parent_fd</span><span class="o">,</span> <span class="n">child_fd</span> <span class="o">=</span> <span class="nn">Unix</span><span class="p">.</span><span class="n">socketpair</span> <span class="nc">PF_UNIX</span> <span class="nc">SOCK_STREAM</span> <span class="mi">0</span> <span class="k">in</span>
<span class="nn">Unix</span><span class="p">.</span><span class="n">set_close_on_exec</span> <span class="n">parent_fd</span><span class="p">;</span>
<span class="k">let</span> <span class="n">_pid</span> <span class="o">=</span>
  <span class="nn">Unix</span><span class="p">.</span><span class="n">create_process</span> <span class="s2">"solver-service"</span>
    <span class="p">[</span><span class="o">|</span> <span class="s2">"solver-service"</span><span class="p">;</span> <span class="s2">"run-child"</span><span class="p">;</span> <span class="s2">"--cache-dir"</span><span class="p">;</span> <span class="n">cache_dir</span> <span class="o">|</span><span class="p">]</span>
    <span class="n">child_fd</span> <span class="nn">Unix</span><span class="p">.</span><span class="n">stderr</span> <span class="nn">Unix</span><span class="p">.</span><span class="n">stderr</span>
<span class="k">in</span>
<span class="nn">Unix</span><span class="p">.</span><span class="n">close</span> <span class="n">child_fd</span><span class="p">;</span>
<span class="k">let</span> <span class="n">p</span> <span class="o">=</span> <span class="n">parent_fd</span> <span class="k">in</span>
<span class="o">...</span>
</code></pre></div></div>

<p>There’s a module called <code class="language-plaintext highlighter-rouge">lib/solver_pool.ml</code> which comes from 2020, prior to the move to the solver-service in 2023. <a href="https://github.com/ocurrent/ocaml-ci/pull/634">PR #634</a>. This was looking like a major tidy up operation.</p>

<p>Two options:</p>

<ol>
  <li>
    <p>Drop the <code class="language-plaintext highlighter-rouge">solver-service</code>/<code class="language-plaintext highlighter-rouge">solver-worker</code> library deps from <code class="language-plaintext highlighter-rouge">lib/dune</code> and <code class="language-plaintext highlighter-rouge">dune-project</code>. For <code class="language-plaintext highlighter-rouge">Local</code> mode the <code class="language-plaintext highlighter-rouge">Backend_solver</code> could reuse the old <code class="language-plaintext highlighter-rouge">lib/solver_pool.ml</code>’s Cap’n Proto-over-pipe path, so OCaml-CI links only <code class="language-plaintext highlighter-rouge">solver-service-api</code> and exec’s the <code class="language-plaintext highlighter-rouge">solver-service</code> binary as an opaque subprocess. Two switches at build time (OCaml-CI on 4.14/Lwt, solver-service on 5.x/Eio); one pipe at runtime.</p>
  </li>
  <li>
    <p>Branch <code class="language-plaintext highlighter-rouge">solver-service</code> from <code class="language-plaintext highlighter-rouge">f14bc6f</code>, cherry-pick <code class="language-plaintext highlighter-rouge">86d37c7</code> onto it, repoint the three <code class="language-plaintext highlighter-rouge">pin-depends</code> lines at the new SHA.</p>
  </li>
</ol>

<p>I prototyped option 1, but it touched a lot of files, showed there’s a bunch of tidy-up work to do, and wasn’t easy to build for local development, so I went with the second option.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>$ git checkout -b f14bc6f-dev-setup f14bc6f
$ git cherry-pick 86d37c7
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">86d37c7</code> adds tests against a framework that doesn’t exist at <code class="language-plaintext highlighter-rouge">f14bc6f</code> it postdates the Eio port, so the test files conflicted. I drop those, taking only the <code class="language-plaintext highlighter-rouge">service/git_context.ml</code> change. The result is a single commit <code class="language-plaintext highlighter-rouge">98c0470</code> on top of <code class="language-plaintext highlighter-rouge">f14bc6f</code> containing the one-line <code class="language-plaintext highlighter-rouge">~dev_setup:false</code> addition.</p>

<p>Pushed to <code class="language-plaintext highlighter-rouge">ocurrent/solver-service</code> as <code class="language-plaintext highlighter-rouge">f14bc6f-dev-setup</code>. The OCaml-CI diff is two files:</p>

<div class="language-diff highlighter-rouge"><div class="highlight"><pre class="highlight"><code> pin-depends: [
<span class="gd">-  ["solver-service.dev" "git+https://github.com/ocurrent/solver-service.git#f14bc6f…"]
-  ["solver-service-api.dev" "git+…f14bc6f…"]
-  ["solver-worker.dev" "git+…f14bc6f…"]
</span><span class="gi">+  ["solver-service.dev" "git+https://github.com/ocurrent/solver-service.git#98c0470…"]
+  ["solver-service-api.dev" "git+…98c0470…"]
+  ["solver-worker.dev" "git+…98c0470…"]
</span> ]
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">opam reinstall solver-service solver-service-api solver-worker</code> rebuilds the binary against the cherry-pick branch; <code class="language-plaintext highlighter-rouge">grep filter_deps</code> in the freshly built tree shows <code class="language-plaintext highlighter-rouge">~dev_setup:false</code> in place. OCaml-CI builds clean and tests pass. The PR is <a href="https://github.com/ocurrent/ocaml-ci/pull/1053">ocurrent/ocaml-ci#1053</a>.</p>
