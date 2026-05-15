---
title: 'Day10: opam package testing tool'
description: ocurrent/obuilder is the workhorse of OCaml CI testing, but the current
  deployment causes packages to be built repeatedly because the opam switch is assembled
  from scratch for each package, leading to common dependencies being frequently recompiled.
  day10 uses an alternative model whereby switches are assembled from their component
  packages.
url: https://www.tunbury.org/2026/02/16/day10/
date: 2026-02-16T19:30:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p><a href="https://github.com/ocurrent/obuilder">ocurrent/obuilder</a> is the workhorse of OCaml CI testing, but the current deployment causes packages to be built repeatedly because the opam switch is assembled from scratch for each package, leading to common dependencies being frequently recompiled. <code class="language-plaintext highlighter-rouge">day10</code> uses an alternative model whereby switches are assembled from their component packages.</p>

<p>Assuming a package A depends upon B and C, while package B depends upon D and E, which is represented by the graph below. <code class="language-plaintext highlighter-rouge">day10</code> would build package D in isolation, capturing the files written to the opam switch and the operating system dependencies. This would be repeated for all the leaf packages E and C. Then, the sets of changed files for both D and E are merged into a new switch, and package B is installed in that switch using the same capturing methodology. For package A, the file sets for D, E, C and B, in order, are merged, and package A is installed.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>    A
   / \
  B   C
 / \
D   E
</code></pre></div></div>

<p>On its own, this is slower than using opam to create the same switch, as opam processes these steps in parallel. However, to create a new switch for package F, <code class="language-plaintext highlighter-rouge">day10</code> can reuse the file sets for B, D, and E without recreating them.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>    F
   / \
  B   G
 / \
D   E
</code></pre></div></div>

<p>In general, each package is installed exactly once and reused on other switches. However, in some cases, packages enable different functionality depending on which other packages are installed. <code class="language-plaintext highlighter-rouge">logs</code> is a good example, with optional libraries such as <code class="language-plaintext highlighter-rouge">fmt</code>, <code class="language-plaintext highlighter-rouge">cmdliner</code>, <code class="language-plaintext highlighter-rouge">lwt</code>, etc. In this case, the package would be installed once for each dependency combination.</p>

<p>The original concept of merging files and recreating the switch came from Jon’s Opam hijinx tool. <a href="https://github.com/jonludlam/opamh">jonludlam/opamh</a>. This functionality is distilled in <code class="language-plaintext highlighter-rouge">day10</code> in this function, which builds the switch state from the directory listing of the installed packages.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">dump_state</span> <span class="n">packages_dir</span> <span class="n">state_file</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">content</span> <span class="o">=</span> <span class="nn">Sys</span><span class="p">.</span><span class="n">readdir</span> <span class="n">packages_dir</span> <span class="o">|&gt;</span> <span class="nn">Array</span><span class="p">.</span><span class="n">to_list</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">packages</span> <span class="o">=</span> <span class="nn">List</span><span class="p">.</span><span class="n">filter_map</span> <span class="p">(</span><span class="k">fun</span> <span class="n">x</span> <span class="o">-&gt;</span> <span class="nn">OpamPackage</span><span class="p">.</span><span class="n">of_string_opt</span> <span class="n">x</span><span class="p">)</span> <span class="n">content</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">sel_compiler</span> <span class="o">=</span> <span class="nn">List</span><span class="p">.</span><span class="n">filter</span> <span class="p">(</span><span class="k">fun</span> <span class="n">x</span> <span class="o">-&gt;</span> <span class="nn">List</span><span class="p">.</span><span class="n">mem</span> <span class="p">(</span><span class="nn">OpamPackage</span><span class="p">.</span><span class="n">name</span> <span class="n">x</span><span class="p">)</span> <span class="n">compiler_packages</span><span class="p">)</span> <span class="n">packages</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">new_state</span> <span class="o">=</span>
    <span class="k">let</span> <span class="n">s</span> <span class="o">=</span> <span class="nn">OpamPackage</span><span class="p">.</span><span class="nn">Set</span><span class="p">.</span><span class="n">of_list</span> <span class="n">packages</span> <span class="k">in</span>
    <span class="p">{</span> <span class="nn">OpamTypes</span><span class="p">.</span><span class="n">sel_installed</span> <span class="o">=</span> <span class="n">s</span><span class="p">;</span> <span class="n">sel_roots</span> <span class="o">=</span> <span class="n">s</span><span class="p">;</span> <span class="n">sel_pinned</span> <span class="o">=</span> <span class="nn">OpamPackage</span><span class="p">.</span><span class="nn">Set</span><span class="p">.</span><span class="n">empty</span><span class="p">;</span> <span class="n">sel_compiler</span> <span class="o">=</span> <span class="nn">OpamPackage</span><span class="p">.</span><span class="nn">Set</span><span class="p">.</span><span class="n">of_list</span> <span class="n">sel_compiler</span> <span class="p">}</span>
  <span class="k">in</span>
  <span class="nn">OpamFilename</span><span class="p">.</span><span class="n">write</span> <span class="p">(</span><span class="nn">OpamFilename</span><span class="p">.</span><span class="n">raw</span> <span class="n">state_file</span><span class="p">)</span> <span class="p">(</span><span class="nn">OpamFile</span><span class="p">.</span><span class="nn">SwitchSelections</span><span class="p">.</span><span class="n">write_to_string</span> <span class="n">new_state</span><span class="p">)</span>
</code></pre></div></div>

<p>opam could be used to install the package in the “recreated” switch, but opam does unnecessary checks, such as finding and checking whether the necessary dependencies are installed. This led to the tool <a href="https://github.com/mtelvers/opam-build">mtelvers/opam-build</a>, which assumes everything is already in place and calls the opam library to install the package without any checks!</p>

<p>The dependency graph includes the compiler, so package ‘D’ might be OCaml 5.4.0, and package ‘E’ might be an OS dependency like ‘conf-curl’, and the captured layer would include <code class="language-plaintext highlighter-rouge">libcurl.so</code>. The underlying OS distribution and version are also captured, so <code class="language-plaintext highlighter-rouge">logs</code> on Debian is assumed to be different to <code class="language-plaintext highlighter-rouge">logs</code> on Fedora.</p>

<p>On Linux, <code class="language-plaintext highlighter-rouge">day10</code> uses overlayfs with <code class="language-plaintext highlighter-rouge">runc</code>. Overlayfs has the concept of a read-only lower directory and a writable upper directory. While these can be stacked, the depth is limited. Therefore, <code class="language-plaintext highlighter-rouge">day10</code> assembles the lower directory by creating a file system tree of hard links to the originally captured files, and an initially empty upper directory is used to capture the files written to it. On FreeBSD, unionfs is used similarly using <code class="language-plaintext highlighter-rouge">jails</code>. On Windows, <code class="language-plaintext highlighter-rouge">containerd</code> is used, but the filesystem isn’t isolated as the hard-linked directory is writable. This hasn’t presented a probably in day-to-day use.</p>

<p>A typical command-line for <code class="language-plaintext highlighter-rouge">day10</code> would specify an initially empty layer-cache directory, your clone of the opam repository, an output format of Markdown or JSON, and the package to be installed using opam’s naming syntax:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 health-check <span class="nt">--cache-dir</span> /var/cache/day10 <span class="nt">--opam-repository</span> /home/mtelvers/opam-repository <span class="nt">--md</span> log.md 0install.2.18
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">day10</code> will attempt to detect your system and build an appropriate container, but you can override this <code class="language-plaintext highlighter-rouge">--os</code> and then in detail with <code class="language-plaintext highlighter-rouge">--os-distribution</code>, <code class="language-plaintext highlighter-rouge">--os-family</code> and <code class="language-plaintext highlighter-rouge">--os-version</code></p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>/var/cache/day10/
└── debian-13-x86_64                         # os specific tag
    ├── 0149f9a8b66c1568d0b962417e827d9f     # layer hash
    │   ├── build.log                        # build log
    │   ├── config.json                      # runc configuration file
    │   ├── fs                               # root file system
    │   ├── hosts                            # runc container hosts file
    │   ├── layer.json                       # layer dependencies and their hashes
    │   └── opam-repository                  # copy of the opam files used to create the switch
    │       ├── packages                     #   laid out in opam repository layout
    │       └── repo
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">day10</code> uses lock files on each layer to allow multiple instances to be invoked at the same time to build different packages. <code class="language-plaintext highlighter-rouge">day10</code> also accepts a list of packages using <code class="language-plaintext highlighter-rouge">@packages.json</code> rather than a specific package name, which can be used along with <code class="language-plaintext highlighter-rouge">--fork</code> to internally create multiple instances.</p>

<p>It’s difficult to know how many processes to fork at once, particularly when packages may be partially or entirely cached and only require the SAT solver to process the request (typically takes less than one second). Therefore, it is often useful to separate the solving step from the building step and run them with different levels of parallelism.  For example, <code class="language-plaintext highlighter-rouge">day10 health-check ... --json /path/to/output --fork $(nproc) --dry-run @packages.json</code> which will solve every package and output a JSON file containing a status field. This allows a second pass to be made with a more conservative <code class="language-plaintext highlighter-rouge">--fork N</code> parameter, as actual building of packages will be performed, only those with a status of “solution” actually need to be submitted.</p>

<table>
  <thead>
    <tr>
      <th>Status</th>
      <th>Meaning</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>success</td>
      <td>The package built successfully, and the build log and dependency graph are included in the output.</td>
    </tr>
    <tr>
      <td>failure</td>
      <td>The package itself fails to build. The build log is included in the output.</td>
    </tr>
    <tr>
      <td>no_solution</td>
      <td>The dependencies of the package cannot be satisfied with the current constraints: compiler version, OS, etc</td>
    </tr>
    <tr>
      <td>dependency_failed</td>
      <td>A dependency failed to build; the log of that failure is included.</td>
    </tr>
    <tr>
      <td>solution</td>
      <td>A solution is available, but a dependency and/or the package itself has not been built. This is only generated with <code class="language-plaintext highlighter-rouge">--dry-run</code>.</td>
    </tr>
  </tbody>
</table>

<p>There is a list command to extract a list of packages from an opam repository. This accepts <code class="language-plaintext highlighter-rouge">--all-version</code> but defaults to the latest version.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 list <span class="nt">--opam-repository</span> ~/opam-repository <span class="nt">--os-distribution</span> debian <span class="nt">--os-family</span> debian <span class="nt">--os-version</span> 13 <span class="nt">--json</span> packages.json
</code></pre></div></div>

<p>Run the build with <code class="language-plaintext highlighter-rouge">--fork 20</code>.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>day10 health-check <span class="nt">--cache-dir</span> ~/cache/ <span class="nt">--opam-repository</span> ~/opam-repository <span class="nt">--os-distribution</span> debian <span class="nt">--os-family</span> debian <span class="nt">--os-version</span> 13 <span class="nt">--json</span> /tmp/foo <span class="nt">--fork</span> 20 @packages.json
</code></pre></div></div>

<p>On my E5-2640 machine (2 x 10C 20T) with an SATA SSD, building the latest version of every package for a single compiler version and OS variant takes a little over an hour.</p>

<p>The project code is available at <a href="https://github.com/mtelvers/day10">mtelvers/day10</a>.</p>
