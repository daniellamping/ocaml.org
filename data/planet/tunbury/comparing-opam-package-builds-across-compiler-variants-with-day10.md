---
title: Comparing opam package builds across compiler variants with day10
description: This post walks through how to use mtelvers/day10 to compare which opam
  packages build successfully under two different compiler configurations.
url: https://www.tunbury.org/2026/03/16/day10/
date: 2026-03-16T19:30:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>This post walks through how to use <a href="https://github.com/mtelvers/day10">mtelvers/day10</a> to compare which opam packages build successfully under two different compiler configurations.</p>

<p>As a real-world example, I will use the comparison of OCaml 5.4.1 with and without <code class="language-plaintext highlighter-rouge">--disable-ocamldoc</code>, but the same approach applies to any compiler variant, configure flag, or opam-repository fork.</p>

<h1>Prerequisites</h1>

<p>You need a Linux machine with:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">runc</code> installed (the OCI container runtime)</li>
  <li>Docker (used to build the base container image)</li>
  <li>A local clone of <a href="https://github.com/ocaml/opam-repository">opam-repository</a></li>
  <li><code class="language-plaintext highlighter-rouge">day10</code> built from source (requires OCaml &gt;= 5.3.0)</li>
</ul>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>git clone https://github.com/mtelvers/day10
<span class="nb">cd </span>day10
opam <span class="nb">install</span> <span class="nb">.</span> <span class="nt">--deps-only</span> <span class="nt">-y</span>
dune build @install
</code></pre></div></div>

<p>The built binary is at <code class="language-plaintext highlighter-rouge">./_build/install/default/bin/day10</code>.</p>

<p>For this example, I am going to additionally clone the opam-repository from <a href="https://github.com/dra27/opam-repository">dra27/opam-repository</a>, which has a branch that adds a single line (<code class="language-plaintext highlighter-rouge">"--disable-ocamldoc"</code>) to <code class="language-plaintext highlighter-rouge">ocaml-compiler.5.4.1/opam</code>.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>git clone <span class="nt">--branch</span> 5.4.1-no-ocamldoc <span class="se">\</span>
  https://github.com/dra27/opam-repository <span class="se">\</span>
  ~/opam-repository-no-ocamldoc
</code></pre></div></div>

<h1>Step 1: Generate the package list</h1>

<p><code class="language-plaintext highlighter-rouge">day10 list</code> queries an opam-repository and outputs every package (latest version only unless <code class="language-plaintext highlighter-rouge">--all-versions</code> is specified) whose constraints are compatible with a given compiler, OS and architecture. The <code class="language-plaintext highlighter-rouge">--json</code> flag writes the list in the format that <code class="language-plaintext highlighter-rouge">day10 health-check</code> expects.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 list <span class="se">\</span>
  <span class="nt">--opam-repository</span> ~/opam-repository <span class="se">\</span>
  <span class="nt">--ocaml-version</span> ocaml.5.4.1 <span class="se">\</span>
  <span class="nt">--os-distribution</span> debian <span class="nt">--os-family</span> debian <span class="nt">--os-version</span> 13 <span class="se">\</span>
  <span class="nt">--json</span> packages-5.4.1.json
</code></pre></div></div>

<p>This produced a JSON file containing 4,322 packages:</p>

<div class="language-json highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">{</span><span class="nl">"packages"</span><span class="p">:[</span><span class="s2">"ambient-context-eio.0.2"</span><span class="p">,</span><span class="s2">"bap-emacs-dot.0.1"</span><span class="p">,</span><span class="s2">"opus.1.0.0"</span><span class="p">,</span><span class="w"> </span><span class="err">...</span><span class="p">]}</span><span class="w">
</span></code></pre></div></div>

<p>The same package list is used for both runs so the results are directly comparable.</p>

<h1>Step 2: Solve (dry-run pass)</h1>

<p>The solver step is embarrassingly parallel and requires little disk ok, so we run it with high parallelism to quickly identify which packages have a valid solution and which do not. The <code class="language-plaintext highlighter-rouge">--dry-run</code> flag skips building entirely.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 health-check <span class="se">\</span>
  <span class="nt">--cache-dir</span> ~/cache <span class="se">\</span>
  <span class="nt">--opam-repository</span> ~/opam-repository <span class="se">\</span>
  <span class="nt">--ocaml-version</span> ocaml.5.4.1 <span class="se">\</span>
  <span class="nt">--os-distribution</span> debian <span class="nt">--os-family</span> debian <span class="nt">--os-version</span> 13 <span class="se">\</span>
  <span class="nt">--json</span> output-dryrun/ <span class="se">\</span>
  <span class="nt">--fork</span> 256 <span class="se">\</span>
  <span class="nt">--dry-run</span> <span class="se">\</span>
  @packages-5.4.1.json
</code></pre></div></div>

<p>This writes one JSON file per package into <code class="language-plaintext highlighter-rouge">output-dryrun/</code>. Each file contains a <code class="language-plaintext highlighter-rouge">status</code> field:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">solution</code>: a valid dependency solution exists (but nothing was built)</li>
  <li><code class="language-plaintext highlighter-rouge">no_solution</code>: no dependency solution for this compiler/OS combination</li>
</ul>

<blockquote>
  <p>Other possible values of status are <code class="language-plaintext highlighter-rouge">dependency_failed</code>, <code class="language-plaintext highlighter-rouge">failed</code>, or <code class="language-plaintext highlighter-rouge">success</code>, but these cannot occur with an empty cache directory. More on this later.</p>
</blockquote>

<p>On my test machine (AMD EPYC 9965) this completed in 36 seconds for 4,322 packages.</p>

<p>Extract just the solvable packages with <code class="language-plaintext highlighter-rouge">jq</code> or Python.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>python3 <span class="nt">-c</span> <span class="s1">'
import json, glob
packages = []
for f in glob.glob("output-dryrun/*.json"):
    with open(f) as fh:
        d = json.load(fh)
        if d.get("status") == "solution":
            packages.append(d["name"])
packages.sort()
with open("packages-solvable.json", "w") as fh:
    json.dump({"packages": packages}, fh)
print(f"Solvable packages: {len(packages)}")
'</span>
</code></pre></div></div>

<p>Result: 3,312 solvable, 1,010 no solution.</p>

<h1>Step 3: Build (run A — standard compiler)</h1>

<p>Now build all solvable packages for real. The <code class="language-plaintext highlighter-rouge">--fork 64</code> flag runs 64 concurrent container builds. Each container uses <code class="language-plaintext highlighter-rouge">runc</code> with an overlay filesystem layered on top of the cached dependency layers. This needs to be adjusted to suit your machine.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 health-check <span class="se">\</span>
  <span class="nt">--cache-dir</span> ~/cache <span class="se">\</span>
  <span class="nt">--opam-repository</span> ~/opam-repository <span class="se">\</span>
  <span class="nt">--ocaml-version</span> ocaml.5.4.1 <span class="se">\</span>
  <span class="nt">--os-distribution</span> debian <span class="nt">--os-family</span> debian <span class="nt">--os-version</span> 13 <span class="se">\</span>
  <span class="nt">--json</span> output-541/ <span class="se">\</span>
  <span class="nt">--fork</span> 64 <span class="se">\</span>
  @packages-solvable.json
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">day10</code> first builds the Debian 13 base image (via Docker), then the compiler, then all packages. The threads use lock files to wait for dependencies to be built.</p>

<p>This produces one JSON file per package in <code class="language-plaintext highlighter-rouge">output-541/</code>. For example, <code class="language-plaintext highlighter-rouge">output-541/ocamlfind.1.9.8.json</code>:</p>

<div class="language-json highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">{</span><span class="w">
  </span><span class="nl">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"ocamlfind.1.9.8"</span><span class="p">,</span><span class="w">
  </span><span class="nl">"status"</span><span class="p">:</span><span class="w"> </span><span class="s2">"success"</span><span class="p">,</span><span class="w">
  </span><span class="nl">"sha"</span><span class="p">:</span><span class="w"> </span><span class="s2">"4f056bfedf536e66065c3783e694e6aa0b38261a"</span><span class="p">,</span><span class="w">
  </span><span class="nl">"layer"</span><span class="p">:</span><span class="w"> </span><span class="s2">"abc123..."</span><span class="p">,</span><span class="w">
  </span><span class="nl">"log"</span><span class="p">:</span><span class="w"> </span><span class="s2">"..."</span><span class="p">,</span><span class="w">
  </span><span class="nl">"solution"</span><span class="p">:</span><span class="w"> </span><span class="s2">"digraph opam { ... }"</span><span class="w">
</span><span class="p">}</span><span class="w">
</span></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">status</code> field is one of:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">success</code>: built and installed successfully</li>
  <li><code class="language-plaintext highlighter-rouge">failure</code>: the package itself failed to build</li>
  <li><code class="language-plaintext highlighter-rouge">dependency_failed</code>: a dependency failed, so this package was not attempted</li>
  <li><code class="language-plaintext highlighter-rouge">solution</code>: has a solution but was not built (dry-run only)</li>
  <li><code class="language-plaintext highlighter-rouge">no_solution</code>: no valid dependency solution</li>
</ul>

<h1>Step 4: Build (run B — variant compiler)</h1>

<p>The second run is identical except for <code class="language-plaintext highlighter-rouge">--opam-repository</code>:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 health-check <span class="se">\</span>
  <span class="nt">--cache-dir</span> ~/cache <span class="se">\</span>
  <span class="nt">--opam-repository</span> ~/opam-repository-no-ocamldoc <span class="se">\</span>
  <span class="nt">--ocaml-version</span> ocaml.5.4.1 <span class="se">\</span>
  <span class="nt">--os-distribution</span> debian <span class="nt">--os-family</span> debian <span class="nt">--os-version</span> 13 <span class="se">\</span>
  <span class="nt">--json</span> output-541-no-ocamldoc/ <span class="se">\</span>
  <span class="nt">--fork</span> 64 <span class="se">\</span>
  @packages-solvable.json
</code></pre></div></div>

<p>Because both runs share the same cache directory, all layers whose opam files are identical are reused. The compiler layer diverges (different opam file hash due to the added <code class="language-plaintext highlighter-rouge">--disable-ocamldoc</code> flag), so the compiler and everything above it is rebuilt. Layers below the compiler (<code class="language-plaintext highlighter-rouge">conf-*</code> packages, the base image) are shared.</p>

<h2>Step 5: Compare results</h2>

<p>With both output directories populated, a simple Python script compares the status of every package:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kn">import</span> <span class="nn">json</span><span class="p">,</span> <span class="n">glob</span>

<span class="k">def</span> <span class="nf">get_results</span><span class="p">(</span><span class="n">d</span><span class="p">):</span>
    <span class="n">results</span> <span class="o">=</span> <span class="p">{}</span>
    <span class="k">for</span> <span class="n">f</span> <span class="ow">in</span> <span class="n">glob</span><span class="p">.</span><span class="n">glob</span><span class="p">(</span><span class="n">d</span> <span class="o">+</span> <span class="s">"/*.json"</span><span class="p">):</span>
        <span class="k">with</span> <span class="nb">open</span><span class="p">(</span><span class="n">f</span><span class="p">)</span> <span class="k">as</span> <span class="n">fh</span><span class="p">:</span>
            <span class="n">data</span> <span class="o">=</span> <span class="n">json</span><span class="p">.</span><span class="n">load</span><span class="p">(</span><span class="n">fh</span><span class="p">)</span>
            <span class="n">results</span><span class="p">[</span><span class="n">data</span><span class="p">[</span><span class="s">"name"</span><span class="p">]]</span> <span class="o">=</span> <span class="n">data</span>
    <span class="k">return</span> <span class="n">results</span>

<span class="n">std</span> <span class="o">=</span> <span class="n">get_results</span><span class="p">(</span><span class="s">"output-541"</span><span class="p">)</span>
<span class="n">nod</span> <span class="o">=</span> <span class="n">get_results</span><span class="p">(</span><span class="s">"output-541-no-ocamldoc"</span><span class="p">)</span>

<span class="c1"># Summary
</span><span class="k">for</span> <span class="n">label</span><span class="p">,</span> <span class="n">res</span> <span class="ow">in</span> <span class="p">[(</span><span class="s">"Standard"</span><span class="p">,</span> <span class="n">std</span><span class="p">),</span> <span class="p">(</span><span class="s">"No-ocamldoc"</span><span class="p">,</span> <span class="n">nod</span><span class="p">)]:</span>
    <span class="n">statuses</span> <span class="o">=</span> <span class="p">{}</span>
    <span class="k">for</span> <span class="n">r</span> <span class="ow">in</span> <span class="n">res</span><span class="p">.</span><span class="n">values</span><span class="p">():</span>
        <span class="n">statuses</span><span class="p">[</span><span class="n">r</span><span class="p">[</span><span class="s">"status"</span><span class="p">]]</span> <span class="o">=</span> <span class="n">statuses</span><span class="p">.</span><span class="n">get</span><span class="p">(</span><span class="n">r</span><span class="p">[</span><span class="s">"status"</span><span class="p">],</span> <span class="mi">0</span><span class="p">)</span> <span class="o">+</span> <span class="mi">1</span>
    <span class="k">print</span><span class="p">(</span><span class="sa">f</span><span class="s">"</span><span class="si">{</span><span class="n">label</span><span class="si">}</span><span class="s">:"</span><span class="p">)</span>
    <span class="k">for</span> <span class="n">k</span><span class="p">,</span> <span class="n">v</span> <span class="ow">in</span> <span class="nb">sorted</span><span class="p">(</span><span class="n">statuses</span><span class="p">.</span><span class="n">items</span><span class="p">()):</span>
        <span class="k">print</span><span class="p">(</span><span class="sa">f</span><span class="s">"  </span><span class="si">{</span><span class="n">k</span><span class="si">}</span><span class="s">: </span><span class="si">{</span><span class="n">v</span><span class="si">}</span><span class="s">"</span><span class="p">)</span>
    <span class="k">print</span><span class="p">()</span>

<span class="c1"># Differences
</span><span class="k">for</span> <span class="n">name</span> <span class="ow">in</span> <span class="nb">sorted</span><span class="p">(</span><span class="nb">set</span><span class="p">(</span><span class="n">std</span><span class="p">)</span> <span class="o">|</span> <span class="nb">set</span><span class="p">(</span><span class="n">nod</span><span class="p">)):</span>
    <span class="n">s1</span> <span class="o">=</span> <span class="n">std</span><span class="p">.</span><span class="n">get</span><span class="p">(</span><span class="n">name</span><span class="p">,</span> <span class="p">{}).</span><span class="n">get</span><span class="p">(</span><span class="s">"status"</span><span class="p">)</span>
    <span class="n">s2</span> <span class="o">=</span> <span class="n">nod</span><span class="p">.</span><span class="n">get</span><span class="p">(</span><span class="n">name</span><span class="p">,</span> <span class="p">{}).</span><span class="n">get</span><span class="p">(</span><span class="s">"status"</span><span class="p">)</span>
    <span class="k">if</span> <span class="n">s1</span> <span class="o">!=</span> <span class="n">s2</span><span class="p">:</span>
        <span class="k">print</span><span class="p">(</span><span class="sa">f</span><span class="s">"  </span><span class="si">{</span><span class="n">name</span><span class="si">}</span><span class="s">: </span><span class="si">{</span><span class="n">s1</span><span class="si">}</span><span class="s"> -&gt; </span><span class="si">{</span><span class="n">s2</span><span class="si">}</span><span class="s">"</span><span class="p">)</span>
</code></pre></div></div>

<p>To see <em>why</em> a package failed, inspect the <code class="language-plaintext highlighter-rouge">log</code> field in the JSON:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>jq <span class="nt">-r</span> <span class="s1">'.log'</span> output-541-no-ocamldoc/camlpdf.2.8.1.json
</code></pre></div></div>

<h1>Example results: <code class="language-plaintext highlighter-rouge">--disable-ocamldoc</code></h1>

<h2>Build summary</h2>

<table>
  <thead>
    <tr>
      <th>&nbsp;</th>
      <th>Standard 5.4.1</th>
      <th>No-ocamldoc 5.4.1</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>Success</strong></td>
      <td>3,163</td>
      <td>3,136</td>
    </tr>
    <tr>
      <td><strong>Failure</strong></td>
      <td>77</td>
      <td>87</td>
    </tr>
    <tr>
      <td><strong>Dependency failed</strong></td>
      <td>72</td>
      <td>89</td>
    </tr>
    <tr>
      <td><strong>Total</strong></td>
      <td>3,312</td>
      <td>3,312</td>
    </tr>
    <tr>
      <td><strong>Wall-clock time</strong></td>
      <td>29m 47s</td>
      <td>27m 45s</td>
    </tr>
  </tbody>
</table>

<h2>Packages broken by <code class="language-plaintext highlighter-rouge">--disable-ocamldoc</code></h2>

<p>27 packages changed status from success to failure or dependency_failed. Of these, 13 are direct failures as they explicitly require <code class="language-plaintext highlighter-rouge">ocamldoc</code> during their build. The remaining 14 are cascading failures from depending on one of the 13.</p>

<h3>Direct failures (13 packages)</h3>

<table>
  <thead>
    <tr>
      <th>Package</th>
      <th>Failure mode</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">broken.0.4.2</code></td>
      <td>Uses bsdowl build system</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">camlgpc.1.2</code></td>
      <td>Runs <code class="language-plaintext highlighter-rouge">ocamldoc -html</code> via OCamlMakefile</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">camlpdf.2.8.1</code></td>
      <td>Runs <code class="language-plaintext highlighter-rouge">ocamldoc -html</code> via OCamlMakefile</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">exn-source.0.1</code></td>
      <td><code class="language-plaintext highlighter-rouge">ocamlfind ocamldoc</code> — “Not supported in your configuration”</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">inspect.0.2.1</code></td>
      <td>Runs <code class="language-plaintext highlighter-rouge">ocamldoc -html</code> via OCamlMakefile</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">llvm.19-static</code></td>
      <td>OCaml bindings build requires ocamldoc</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">mlcuddidl.3.0.8</code></td>
      <td><code class="language-plaintext highlighter-rouge">./configure</code> checks for ocamldoc, fails if absent</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">mlgmpidl.1.3.0</code></td>
      <td><code class="language-plaintext highlighter-rouge">./configure</code> checks for ocamldoc, reports “OCaml not found”</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">ocamldot.1.1</code></td>
      <td><code class="language-plaintext highlighter-rouge">./configure</code> raises <code class="language-plaintext highlighter-rouge">Program_not_found "ocamldoc"</code></td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">ocp-ocamlres.0.4</code></td>
      <td><code class="language-plaintext highlighter-rouge">make doc</code> target fails via ocamlfind</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">ollvm.0.99</code></td>
      <td><code class="language-plaintext highlighter-rouge">./configure</code> requires ocamldoc</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">ollvm-tapir.0.99.1</code></td>
      <td><code class="language-plaintext highlighter-rouge">./configure</code> requires ocamldoc</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">prooftree.0.14</code></td>
      <td><code class="language-plaintext highlighter-rouge">./configure</code> checks for <code class="language-plaintext highlighter-rouge">ocamldoc.opt</code>, fails if absent</td>
    </tr>
  </tbody>
</table>

<p>Additionally, <code class="language-plaintext highlighter-rouge">zarith.1.11</code> (pulled as a dependency of <code class="language-plaintext highlighter-rouge">bls12-381-js</code>) and
<code class="language-plaintext highlighter-rouge">camlpdf.2.5.3</code> (pulled as a dependency of <code class="language-plaintext highlighter-rouge">graphicspdf</code>) also fail their
configure checks for ocamldoc.</p>

<h3>Cascading failures (14 packages)</h3>

<table>
  <thead>
    <tr>
      <th>Package</th>
      <th>Failed dependency</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">absolute.0.3</code></td>
      <td>mlgmpidl</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">apron.v0.9.15</code></td>
      <td>mlgmpidl</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">apronext.1.0.4</code></td>
      <td>mlgmpidl</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">bls12-381-js-gen.0.5.0</code></td>
      <td>zarith (via configure check)</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">bls12-381-js.0.5.0</code></td>
      <td>zarith (via configure check)</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">calli.0.2</code></td>
      <td>llvm</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">configuration.0.4.1</code></td>
      <td>broken</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">cpdf.2.8.1</code></td>
      <td>camlpdf</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">graphicspdf.2.2.1</code></td>
      <td>camlpdf (older version)</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">jasmin.2026.03.0</code></td>
      <td>mlgmpidl</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">libabsolute.0.1</code></td>
      <td>mlgmpidl</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">mopsa.1.2</code></td>
      <td>mlgmpidl</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">opam-graph.0.1.1</code></td>
      <td>ocamldot</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">picasso.0.4.0</code></td>
      <td>mlgmpidl</td>
    </tr>
  </tbody>
</table>

<h3>Failure patterns</h3>

<p>The failures fall into a few categories:</p>

<ol>
  <li>
    <p>Configure scripts that hard-require ocamldoc (mlcuddidl, mlgmpidl, ocamldot, ollvm, prooftree, zarith). These check for <code class="language-plaintext highlighter-rouge">ocamldoc</code> during <code class="language-plaintext highlighter-rouge">./configure</code> and abort if it’s missing, even if they don’t strictly need it for compilation.</p>
  </li>
  <li>
    <p>Build targets that generate documentation (camlgpc, camlpdf, inspect, ocp-ocamlres, exn-source). These invoke <code class="language-plaintext highlighter-rouge">ocamldoc</code> or <code class="language-plaintext highlighter-rouge">ocamlfind ocamldoc</code> as part of their default build, not as an optional doc target.</p>
  </li>
  <li>
    <p>Build systems that assume ocamldoc exists (broken via bsdowl, llvm OCaml bindings). These integrate ocamldoc into their build infrastructure.</p>
  </li>
</ol>

<h3>Conclusion</h3>

<p>Removing <code class="language-plaintext highlighter-rouge">ocamldoc</code> from the default OCaml 5.4.1 installation affects 27 out of 3,312 solvable packages (0.8%), with only 15 distinct root causes (13 latest versions plus 2 older versions pulled as dependencies). The remaining 3,136 packages (94.7% of the total, 99.1% of those that built successfully) are completely unaffected.</p>

<p>Most of the affected packages use older build systems (OCamlMakefile, custom configure scripts) that unconditionally check for or invoke <code class="language-plaintext highlighter-rouge">ocamldoc</code>. Modern build systems like dune do not exhibit this problem.</p>

<h2>Tips</h2>

<ul>
  <li>Use the same package list for both runs: Generate it once with <code class="language-plaintext highlighter-rouge">day10 list</code> and pass it to both builds. The solver is unaffected by build instruction changes (like <code class="language-plaintext highlighter-rouge">--disable-ocamldoc</code>), so the solvable set is identical. If you generate the list a section time, invert the logic to use build where solution != “no_solution”.</li>
  <li>Shared cache is safe and beneficial: Both runs can use the same <code class="language-plaintext highlighter-rouge">--cache-dir</code>. Layers are keyed by a hash of their opam file contents and dependency chain, so different compiler configurations naturally get different hashes. Compiler-independent layers (base image, <code class="language-plaintext highlighter-rouge">conf-*</code> packages) are reused automatically.</li>
  <li>Tune <code class="language-plaintext highlighter-rouge">--fork</code> to your machine: The solve pass is I/O light, so can use a <code class="language-plaintext highlighter-rouge">--fork $(nproc)</code>. However, the build pass is I/O intensive so I suggested <code class="language-plaintext highlighter-rouge">nproc / 4</code> and revise accordingly.</li>
</ul>

<h3>Timings</h3>

<p>These timings are from an AMD EPYC 9965 with NMVe storage:</p>

<table>
  <thead>
    <tr>
      <th>Step</th>
      <th>Wall-clock</th>
      <th>User</th>
      <th>Sys</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Solve (4,322 packages, <code class="language-plaintext highlighter-rouge">--fork 256</code>)</td>
      <td>36s</td>
      <td>25m 50s</td>
      <td>101m 36s</td>
    </tr>
    <tr>
      <td>Build standard (3,312 packages, <code class="language-plaintext highlighter-rouge">--fork 64</code>)</td>
      <td>29m 47s</td>
      <td>567m 36s</td>
      <td>444m 17s</td>
    </tr>
    <tr>
      <td>Build no-ocamldoc (3,312 packages, <code class="language-plaintext highlighter-rouge">--fork 64</code>)</td>
      <td>27m 45s</td>
      <td>486m 55s</td>
      <td>398m 24s</td>
    </tr>
  </tbody>
</table>

<p>The no-ocamldoc build was approximately 2 minutes faster, as the <code class="language-plaintext highlighter-rouge">conf-*</code> packages were cached since they are not dependent on the compiler.</p>
