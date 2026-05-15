---
title: 'OBuilder on Windows: Bringing Native Container Builds with the HCS Backend'
description: Following from my containerd posts last year and my previous work on
  obuilder backends for macOS and QEMU, this post extends obuilder to use the Host
  Compute System (HCS) and containerd on Windows.
url: https://www.tunbury.org/2026/02/19/obuilder-hcs/
date: 2026-02-19T19:25:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Following from my containerd <a href="https://www.tunbury.org/2025/06/11/windows-containerd/">posts</a> <a href="https://www.tunbury.org/2025/06/14/windows-containerd-2/">last</a> <a href="https://www.tunbury.org/2025/06/27/windows-containerd-3/">year</a> and my previous work on obuilder backends for <a href="https://tarides.com/blog/2023-08-02-obuilder-on-macos/">macOS</a> and <a href="https://github.com/ocurrent/obuilder/pull/195">QEMU</a>, this post extends obuilder to use the Host Compute System (HCS) and <a href="https://containerd.io">containerd</a> on Windows.</p>

<p>OBuilder, written by Thomas Leonard, is a sandboxed build executor for OCaml CI pipelines. It takes a build specification, similar to a Dockerfile, but written in S-expression syntax, and executes each step in an isolated environment, caching results at the filesystem level.</p>

<p>OBuilder’s sandbox backends target Linux (via runc), macOS (via user sandboxing), FreeBSD (via jails), and Docker and any else via QEMU. This post introduces the HCS backend, which brings native Windows container builds to OBuilder using Microsoft’s Host Compute Service and containerd.</p>

<h2>How OBuilder Works</h2>

<p>Before looking at the Windows-specific details, let’s recap on how OBuilder works.</p>

<h3>Build Specifications</h3>

<p>A typical OBuilder is shown below:</p>

<div class="language-scheme highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">((</span><span class="nf">from</span> <span class="nv">ocaml/opam:debian</span><span class="p">)</span>
 <span class="p">(</span><span class="nf">workdir</span> <span class="nv">/src</span><span class="p">)</span>
 <span class="p">(</span><span class="nf">user</span> <span class="p">(</span><span class="nf">uid</span> <span class="mi">1000</span><span class="p">)</span> <span class="p">(</span><span class="nf">gid</span> <span class="mi">1000</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">run</span> <span class="p">(</span><span class="nf">shell</span> <span class="s">"sudo chown opam /src"</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">copy</span> <span class="p">(</span><span class="nf">src</span> <span class="nv">obuilder-spec</span><span class="o">.</span><span class="nv">opam</span> <span class="nv">obuilder</span><span class="o">.</span><span class="nv">opam</span><span class="p">)</span> <span class="p">(</span><span class="nf">dst</span> <span class="o">.</span><span class="nv">/</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">run</span> <span class="p">(</span><span class="nf">shell</span> <span class="s">"opam pin add -yn ."</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">run</span>
  <span class="p">(</span><span class="nf">network</span> <span class="nv">host</span><span class="p">)</span>
  <span class="p">(</span><span class="nf">shell</span> <span class="s">"opam install --deps-only -t obuilder"</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">copy</span> <span class="p">(</span><span class="nf">src</span> <span class="o">.</span><span class="p">)</span> <span class="p">(</span><span class="nf">dst</span> <span class="nv">/src/</span><span class="p">)</span> <span class="p">(</span><span class="nf">exclude</span> <span class="o">.</span><span class="nv">git</span> <span class="nv">_build</span> <span class="nv">_opam</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">run</span> <span class="p">(</span><span class="nf">shell</span> <span class="s">"opam exec -- dune build @install @runtest"</span><span class="p">)))</span>
</code></pre></div></div>

<p>Each operation, such as <code class="language-plaintext highlighter-rouge">from</code>, <code class="language-plaintext highlighter-rouge">run</code>, <code class="language-plaintext highlighter-rouge">copy</code>, <code class="language-plaintext highlighter-rouge">workdir</code>, <code class="language-plaintext highlighter-rouge">env</code>, <code class="language-plaintext highlighter-rouge">shell</code>, is executed in sequence inside a sandboxed container. The resulting filestem is the aggregation of all the previous steps and is recorded as the hash of all the steps up to that point. OBuilder will reuse these layers as a cache of the build steps up to that point instead of re-executing the step.</p>

<p>OBuilder’s functor architecture allows it to be easily extended by providing new store, sandbox, and fetcher implementations. The new Windows backend uses <code class="language-plaintext highlighter-rouge">hcs_store.ml</code>, <code class="language-plaintext highlighter-rouge">hcs_sandbox.ml</code> and <code class="language-plaintext highlighter-rouge">hcs_fetch.ml</code>.</p>

<h3>The Build Flow</h3>

<p>When OBuilder processes a spec, it:</p>

<ol>
  <li>Fetches the base image: (<code class="language-plaintext highlighter-rouge">from</code> directive) using the fetcher module</li>
  <li>For each operation, compute a content hash from the operation and its inputs</li>
  <li>Checks the cache: if a result for that hash exists, skip execution</li>
  <li>Creates a snapshot from the previous step’s result using the store module</li>
  <li>Runs the operation inside the sandbox using the sandbox</li>
  <li>Commits the result as a new snapshot, keyed by the content hash</li>
</ol>

<p>This means repeat builds are very fast, and with carefully constructed spec files, incremental builds due to code changes can be built without needing to rebuild the project dependencies (the opam switch).</p>

<h2>The HCS Backend</h2>

<p>The Host Compute Service (HCS) backend enables native Windows container builds using <code class="language-plaintext highlighter-rouge">containerd</code>.</p>

<h3>Architecture</h3>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>┌────────────────────────────────────────────────────┐
│              OBuilder CLI (main.ml)                │
│     obuilder build --store=hcs:C:\obuilder         │
└────────────────────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────┐
│            Builder Functor (build.ml)              │
│     Build.Make(Hcs_store)(Hcs_sandbox)(Hcs_fetch)  │
└────────────────────────────────────────────────────┘
        │                │                │
        ▼                ▼                ▼
  ┌───────────┐   ┌────────────┐   ┌───────────┐
  │ Hcs_store │   │Hcs_sandbox │   │ Hcs_fetch │
  │           │   │            │   │           │
  │ Snapshot  │   │ Container  │   │ Base image│
  │ mgmt via  │   │ exec via   │   │ import via│
  │ ctr snap  │   │ ctr run    │   │ ctr pull  │
  └───────────┘   └────────────┘   └───────────┘
        │                │                │
        └────────────────┼────────────────┘
                         ▼
┌────────────────────────────────────────────────────┐
│              containerd (Windows)                  │
│   Images  │  Snapshots (VHDX)  │  Runtime (HCS)    │
└────────────────────────────────────────────────────┘
</code></pre></div></div>

<h3>Split Storage Model</h3>

<p>Obuilder backends, typically use filesystem features, such as BTRFS or ZFS snapshots to store the cache layer within the obuilder results directory, typically <code class="language-plaintext highlighter-rouge">/var/cache/obuilder/results/&lt;hashid&gt;/rootfs</code>. However, HCS automatically stores the actual filesystem snaphots in VHDX files in <code class="language-plaintext highlighter-rouge">C:\ProgramData\containerd\snapshots\&lt;N&gt;</code>, so the obuilder results directory contains only a JSON file with a pointer to this directory.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>OBuilder Store (C:\obuilder\)         Containerd (C:\ProgramData\containerd\)
├── result\&lt;id&gt;\                      ├── snapshots\
│   ├── rootfs\                       │   ├── 1\    ← VHDX layer data
│   │   └── layerinfo.json ────────►  │   ├── 2\    ← VHDX layer data
│   ├── log                           │   └── 3\    ← VHDX layer data
│   └── env                           └── metadata.db
├── state\db\db.sqlite
└── cache\
</code></pre></div></div>

<h2>Walking Through a Build</h2>

<p>Let’s trace what happens when you run:</p>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">obuilder</span><span class="w"> </span><span class="nx">build</span><span class="w"> </span><span class="nt">-f</span><span class="w"> </span><span class="nx">example.windows.hcs.spec</span><span class="w"> </span><span class="o">.</span><span class="w"> </span><span class="nt">--store</span><span class="o">=</span><span class="n">hcs:C:\obuilder</span><span class="w">
</span></code></pre></div></div>

<p>with the following spec:</p>

<div class="language-scheme highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">((</span><span class="nf">from</span> <span class="nv">mcr</span><span class="o">.</span><span class="nv">microsoft</span><span class="o">.</span><span class="nv">com/windows/nanoserver:ltsc2025</span><span class="p">)</span>
 <span class="p">(</span><span class="nf">run</span> <span class="p">(</span><span class="nf">shell</span> <span class="s">"echo hello"</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">run</span> <span class="p">(</span><span class="nf">shell</span> <span class="s">"mkdir C:\\app"</span><span class="p">)))</span>
</code></pre></div></div>

<h3>Step 1: Fetch the Base Image (hcs_fetch.ml)</h3>

<p>The fetcher pulls the base image from the Microsoft Container Registry and prepares an initial snapshot.</p>

<p>First, it normalises the image reference. Docker Hub images need a <code class="language-plaintext highlighter-rouge">docker.io/</code> prefix for containerd (e.g. <code class="language-plaintext highlighter-rouge">ubuntu:latest</code> becomes <code class="language-plaintext highlighter-rouge">docker.io/library/ubuntu:latest</code>), but Microsoft Container Registry (MCR) images are used as-is.</p>

<p>The equivalent manual commands are:</p>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Pull the image</span><span class="w">
</span><span class="n">ctr</span><span class="w"> </span><span class="nx">image</span><span class="w"> </span><span class="nx">pull</span><span class="w"> </span><span class="nx">mcr.microsoft.com/windows/nanoserver:ltsc2025</span><span class="w">

</span><span class="c"># Get the chain ID (the snapshot key for the image's top layer)</span><span class="w">
</span><span class="n">ctr</span><span class="w"> </span><span class="nx">images</span><span class="w"> </span><span class="nx">pull</span><span class="w"> </span><span class="nt">--print-chainid</span><span class="w"> </span><span class="nt">--local</span><span class="w"> </span><span class="nx">mcr.microsoft.com/windows/nanoserver:ltsc2025</span><span class="w">
</span><span class="c"># Output includes: "image chain ID: sha256:abc123..."</span><span class="w">

</span><span class="c"># Prepare a writable snapshot from the image</span><span class="w">
</span><span class="n">ctr</span><span class="w"> </span><span class="nx">snapshot</span><span class="w"> </span><span class="nx">prepare</span><span class="w"> </span><span class="nt">--mounts</span><span class="w"> </span><span class="nx">obuilder-base-</span><span class="err">&lt;</span><span class="nx">hash</span><span class="err">&gt;</span><span class="w"> </span><span class="nx">sha256:abc123...</span><span class="w">
</span><span class="c"># Returns JSON with mount information:</span><span class="w">
</span><span class="c"># [{"Type":"windows-layer","Source":"C:\\...\\snapshots\\42",</span><span class="w">
</span><span class="c">#   "Options":["rw","parentLayerPaths=[\"C:\\\\...\\\\snapshots\\\\20\"]"]}]</span><span class="w">
</span></code></pre></div></div>

<p>The fetcher parses this mount JSON to extract the source path and parent layer paths, then writes <code class="language-plaintext highlighter-rouge">layerinfo.json</code>:</p>

<div class="language-json highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">{</span><span class="w">
  </span><span class="nl">"snapshot_key"</span><span class="p">:</span><span class="w"> </span><span class="s2">"obuilder-base-&lt;hash&gt;"</span><span class="p">,</span><span class="w">
  </span><span class="nl">"source"</span><span class="p">:</span><span class="w"> </span><span class="s2">"C:</span><span class="se">\\</span><span class="s2">ProgramData</span><span class="se">\\</span><span class="s2">containerd</span><span class="se">\\</span><span class="s2">...</span><span class="se">\\</span><span class="s2">snapshots</span><span class="se">\\</span><span class="s2">42"</span><span class="p">,</span><span class="w">
  </span><span class="nl">"parent_layer_paths"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w">
    </span><span class="s2">"C:</span><span class="se">\\</span><span class="s2">ProgramData</span><span class="se">\\</span><span class="s2">containerd</span><span class="se">\\</span><span class="s2">...</span><span class="se">\\</span><span class="s2">snapshots</span><span class="se">\\</span><span class="s2">20"</span><span class="p">,</span><span class="w">
    </span><span class="s2">"C:</span><span class="se">\\</span><span class="s2">ProgramData</span><span class="se">\\</span><span class="s2">containerd</span><span class="se">\\</span><span class="s2">...</span><span class="se">\\</span><span class="s2">snapshots</span><span class="se">\\</span><span class="s2">21"</span><span class="w">
  </span><span class="p">]</span><span class="w">
</span><span class="p">}</span><span class="w">
</span></code></pre></div></div>

<p>Finally, it extracts environment variables from the image config:</p>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Get the config digest</span><span class="w">
</span><span class="n">ctr</span><span class="w"> </span><span class="nx">images</span><span class="w"> </span><span class="nx">inspect</span><span class="w"> </span><span class="nx">mcr.microsoft.com/windows/nanoserver:ltsc2025</span><span class="w">
</span><span class="c"># Look for: "application/vnd.docker.container.image.v1+json @sha256:def456..."</span><span class="w">

</span><span class="c"># Get the config content</span><span class="w">
</span><span class="n">ctr</span><span class="w"> </span><span class="nx">content</span><span class="w"> </span><span class="nx">get</span><span class="w"> </span><span class="nx">sha256:def456...</span><span class="w">
</span><span class="c"># Parse the config.Env array from the JSON</span><span class="w">
</span></code></pre></div></div>

<h3>Step 2: Run “echo hello” (hcs_store.ml + hcs_sandbox.ml)</h3>

<p>For each <code class="language-plaintext highlighter-rouge">run</code> directive, the store creates a new snapshot from the previous step, the sandbox executes the command, and the store commits the result.</p>

<h4>Store: prepare a snapshot</h4>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Read layerinfo.json from parent to get its snapshot key</span><span class="w">
</span><span class="c"># Prepare a new writable snapshot from the parent's committed snapshot</span><span class="w">
</span><span class="n">ctr</span><span class="w"> </span><span class="nx">snapshot</span><span class="w"> </span><span class="nx">prepare</span><span class="w"> </span><span class="nt">--mounts</span><span class="w"> </span><span class="nx">obuilder-</span><span class="err">&lt;</span><span class="nx">id2</span><span class="err">&gt;</span><span class="w"> </span><span class="nx">obuilder-base-</span><span class="err">&lt;</span><span class="nx">hash</span><span class="err">&gt;</span><span class="nt">-committed</span><span class="w">
</span></code></pre></div></div>

<h4>Sandbox: generate OCI config and run</h4>

<p>The sandbox reads <code class="language-plaintext highlighter-rouge">layerinfo.json</code> and generates an OCI runtime config:</p>

<div class="language-json highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">{</span><span class="w">
  </span><span class="nl">"ociVersion"</span><span class="p">:</span><span class="w"> </span><span class="s2">"1.1.0"</span><span class="p">,</span><span class="w">
  </span><span class="nl">"process"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w">
    </span><span class="nl">"terminal"</span><span class="p">:</span><span class="w"> </span><span class="kc">false</span><span class="p">,</span><span class="w">
    </span><span class="nl">"user"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"> </span><span class="nl">"username"</span><span class="p">:</span><span class="w"> </span><span class="s2">"ContainerUser"</span><span class="w"> </span><span class="p">},</span><span class="w">
    </span><span class="nl">"args"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="s2">"cmd"</span><span class="p">,</span><span class="w"> </span><span class="s2">"/S"</span><span class="p">,</span><span class="w"> </span><span class="s2">"/C"</span><span class="p">,</span><span class="w"> </span><span class="s2">"echo hello"</span><span class="p">],</span><span class="w">
    </span><span class="nl">"env"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="s2">"PATH=C:</span><span class="se">\\</span><span class="s2">Windows</span><span class="se">\\</span><span class="s2">System32;C:</span><span class="se">\\</span><span class="s2">Windows"</span><span class="p">],</span><span class="w">
    </span><span class="nl">"cwd"</span><span class="p">:</span><span class="w"> </span><span class="s2">"C:</span><span class="se">\\</span><span class="s2">"</span><span class="w">
  </span><span class="p">},</span><span class="w">
  </span><span class="nl">"root"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"> </span><span class="nl">"path"</span><span class="p">:</span><span class="w"> </span><span class="s2">""</span><span class="p">,</span><span class="w"> </span><span class="nl">"readonly"</span><span class="p">:</span><span class="w"> </span><span class="kc">false</span><span class="w"> </span><span class="p">},</span><span class="w">
  </span><span class="nl">"hostname"</span><span class="p">:</span><span class="w"> </span><span class="s2">"builder"</span><span class="p">,</span><span class="w">
  </span><span class="nl">"windows"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w">
    </span><span class="nl">"layerFolders"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w">
      </span><span class="s2">"C:</span><span class="se">\\</span><span class="s2">ProgramData</span><span class="se">\\</span><span class="s2">containerd</span><span class="se">\\</span><span class="s2">...</span><span class="se">\\</span><span class="s2">snapshots</span><span class="se">\\</span><span class="s2">20"</span><span class="p">,</span><span class="w">
      </span><span class="s2">"C:</span><span class="se">\\</span><span class="s2">ProgramData</span><span class="se">\\</span><span class="s2">containerd</span><span class="se">\\</span><span class="s2">...</span><span class="se">\\</span><span class="s2">snapshots</span><span class="se">\\</span><span class="s2">21"</span><span class="p">,</span><span class="w">
      </span><span class="s2">"C:</span><span class="se">\\</span><span class="s2">ProgramData</span><span class="se">\\</span><span class="s2">containerd</span><span class="se">\\</span><span class="s2">...</span><span class="se">\\</span><span class="s2">snapshots</span><span class="se">\\</span><span class="s2">42"</span><span class="p">,</span><span class="w">
      </span><span class="s2">"C:</span><span class="se">\\</span><span class="s2">ProgramData</span><span class="se">\\</span><span class="s2">containerd</span><span class="se">\\</span><span class="s2">...</span><span class="se">\\</span><span class="s2">snapshots</span><span class="se">\\</span><span class="s2">43"</span><span class="w">
    </span><span class="p">]</span><span class="w">
  </span><span class="p">}</span><span class="w">
</span><span class="p">}</span><span class="w">
</span></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">layerFolders</code> array lists all parent layers followed by the writable scratch layer. This is the Windows container equivalent of an overlay filesystem — the HCS merges all these layers together when the container starts.</p>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Run the container</span><span class="w">
</span><span class="n">ctr</span><span class="w"> </span><span class="nx">run</span><span class="w"> </span><span class="nt">--rm</span><span class="w"> </span><span class="nt">--config</span><span class="w"> </span><span class="nx">config.json</span><span class="w"> </span><span class="nx">obuilder-run-0</span><span class="w">
</span></code></pre></div></div>

<h4>Store: commit the result</h4>

<p>After the command succeeds:</p>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Commit the writable snapshot to a permanent one</span><span class="w">
</span><span class="n">ctr</span><span class="w"> </span><span class="nx">snapshot</span><span class="w"> </span><span class="nx">commit</span><span class="w"> </span><span class="nx">obuilder-</span><span class="err">&lt;</span><span class="nx">id2</span><span class="err">&gt;</span><span class="nt">-committed</span><span class="w"> </span><span class="nx">obuilder-</span><span class="err">&lt;</span><span class="nx">id2</span><span class="err">&gt;</span><span class="w">
</span></code></pre></div></div>

<p>The result directory is then moved from <code class="language-plaintext highlighter-rouge">result-tmp/&lt;id2&gt;</code> to <code class="language-plaintext highlighter-rouge">result/&lt;id2&gt;</code>.</p>

<h3>Step 3: Run “mkdir C:\app”</h3>

<p>The process repeats: prepare a snapshot from <code class="language-plaintext highlighter-rouge">obuilder-&lt;id2&gt;-committed</code>, run the command, commit the result. Each step builds on the previous one, forming a chain of containerd snapshots.</p>

<h2>Networking</h2>

<p>Windows containers don’t support <code class="language-plaintext highlighter-rouge">--net-host</code> in the way Linux containers do. Instead, network access requires three components working together:</p>

<ol>
  <li>An Host Networking Service (HNS) NAT network with a specific subnet</li>
  <li>A Container Network Interface (CNI) config at <code class="language-plaintext highlighter-rouge">C:\Program Files\containerd\cni\conf\0-containerd-nat.conf</code> matching that subnet</li>
  <li>An HCN namespace per container</li>
</ol>

<p>The sandbox creates and destroys HCN namespaces around each networked container execution:</p>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Before the container</span><span class="w">
</span><span class="n">hcn-namespace</span><span class="w"> </span><span class="nx">create</span><span class="w">
</span><span class="c"># Returns a GUID, e.g. "a1b2c3d4-..."</span><span class="w">

</span><span class="c"># The GUID is passed in the OCI config:</span><span class="w">
</span><span class="c"># "windows": { "network": { "networkNamespace": "a1b2c3d4-..." } }</span><span class="w">

</span><span class="c"># Run with --cni flag</span><span class="w">
</span><span class="n">ctr</span><span class="w"> </span><span class="nx">run</span><span class="w"> </span><span class="nt">--rm</span><span class="w"> </span><span class="nt">--cni</span><span class="w"> </span><span class="nt">--config</span><span class="w"> </span><span class="nx">config.json</span><span class="w"> </span><span class="nx">obuilder-run-0</span><span class="w">

</span><span class="c"># After the container</span><span class="w">
</span><span class="n">hcn-namespace</span><span class="w"> </span><span class="nx">delete</span><span class="w"> </span><span class="nx">a1b2c3d4-...</span><span class="w">
</span></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">hcn-namespace</code> tool is a small OCaml utility (<a href="https://github.com/mtelvers/hcn-namespace">mtelvers/hcn-namespace</a>) that wraps the Windows HCN API, written last year while working on <code class="language-plaintext highlighter-rouge">day10</code>.</p>

<h2>The COPY Operation</h2>

<p>File copying works differently on Windows due to I/O constraints. On Linux, OBuilder streams tar data through a pipe directly into the sandbox’s stdin. On Windows, the tar data is first written to a temporary file, then the file is passed as stdin to the container:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Linux:   generate tar  ──pipe──►  sandbox stdin  ──►  tar -xf -
Windows: generate tar  ──►  temp file  ──►  sandbox stdin  ──►  tar -xf -
</code></pre></div></div>

<p>This extra step is needed because Lwt’s pipe I/O is unreliable on Windows (more on this below).</p>

<h2>Running It</h2>

<h3>Prerequisites</h3>

<ol>
  <li>Windows Server 2019 or later (tested on LTSC 2019 and LTSC 2025)</li>
  <li>Containerd v2.0+ installed and running as a service</li>
  <li>ctr: CLI available in PATH</li>
  <li><a href="https://github.com/mtelvers/hcn-namespace">hcn-namespace</a>: tool for networking support</li>
</ol>

<h3>Building OBuilder on Windows</h3>

<p>OBuilder builds itself — the provided <code class="language-plaintext highlighter-rouge">example.windows.hcs.spec</code> bootstraps the build using an MSVC-based OCaml image:</p>

<div class="language-scheme highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">((</span><span class="nf">from</span> <span class="nv">ocaml/opam:windows-server-msvc-ltsc2025-ocaml-5</span><span class="o">.</span><span class="mi">4</span><span class="p">)</span>
 <span class="p">(</span><span class="nf">workdir</span> <span class="s">"C:/src"</span><span class="p">)</span>
 <span class="p">(</span><span class="nf">copy</span> <span class="p">(</span><span class="nf">src</span> <span class="nv">obuilder-spec</span><span class="o">.</span><span class="nv">opam</span> <span class="nv">obuilder</span><span class="o">.</span><span class="nv">opam</span><span class="p">)</span> <span class="p">(</span><span class="nf">dst</span> <span class="o">.</span><span class="nv">/</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">run</span> <span class="p">(</span><span class="nf">shell</span> <span class="s">"echo (lang dune 3.0)&gt; dune-project"</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">run</span> <span class="p">(</span><span class="nf">shell</span> <span class="s">"opam pin add -yn ."</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">run</span> <span class="p">(</span><span class="nf">network</span> <span class="nv">host</span><span class="p">)</span>
  <span class="p">(</span><span class="nf">shell</span> <span class="s">"opam install --deps-only -t obuilder"</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">copy</span> <span class="p">(</span><span class="nf">src</span> <span class="o">.</span><span class="p">)</span> <span class="p">(</span><span class="nf">dst</span> <span class="s">"C:/src/"</span><span class="p">)</span> <span class="p">(</span><span class="nf">exclude</span> <span class="o">.</span><span class="nv">git</span> <span class="nv">_build</span> <span class="nv">_opam</span><span class="p">))</span>
 <span class="p">(</span><span class="nf">run</span> <span class="p">(</span><span class="nf">shell</span> <span class="s">"opam exec -- dune build @install @runtest"</span><span class="p">)))</span>
</code></pre></div></div>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">obuilder</span><span class="w"> </span><span class="nx">build</span><span class="w"> </span><span class="nt">-f</span><span class="w"> </span><span class="nx">example.windows.hcs.spec</span><span class="w"> </span><span class="o">.</span><span class="w"> </span><span class="nt">--store</span><span class="o">=</span><span class="n">hcs:C:\obuilder</span><span class="w">
</span></code></pre></div></div>

<h3>Healthcheck</h3>

<p>To verify the setup:</p>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">obuilder</span><span class="w"> </span><span class="nx">healthcheck</span><span class="w"> </span><span class="nt">--store</span><span class="o">=</span><span class="n">hcs:C:\obuilder</span><span class="w">
</span></code></pre></div></div>

<p>This pulls <code class="language-plaintext highlighter-rouge">mcr.microsoft.com/windows/nanoserver:ltsc2025</code>, runs <code class="language-plaintext highlighter-rouge">echo healthcheck</code> inside a container, and confirms everything works end-to-end.</p>

<h2>Addendum: Lwt on Windows</h2>

<p>The HCS backend development highlighted serveral issues with Lwt on Windows:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">Lwt_process.exec</code> child promise isn’t resolved</li>
  <li><code class="language-plaintext highlighter-rouge">Lwt_unix.waitpid</code> hangs indefinitely unless created with <code class="language-plaintext highlighter-rouge">cmd.exe /c</code></li>
  <li><code class="language-plaintext highlighter-rouge">Lwt_unix.write</code> can randomly hang, affecting tar and log streaming.</li>
  <li><code class="language-plaintext highlighter-rouge">Lwt_io.with_file</code> fails with “Permission denied”</li>
  <li><code class="language-plaintext highlighter-rouge">Os.pread_result</code> works intermittently, but frequently fails with <code class="language-plaintext highlighter-rouge">ctr</code></li>
</ul>

<h2>Code</h2>

<p>My code is available at <a href="https://github.com/mtelvers/obuilder/tree/hcs">mtelvers/obuilder/tree/hcs</a>. I have an ocluster and OCaml-CI patch, but the LWT issues dominate reliability.</p>
