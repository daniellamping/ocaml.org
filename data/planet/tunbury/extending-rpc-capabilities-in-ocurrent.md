---
title: Extending RPC capabilities in OCurrent
description: As our workflows become more agentic, CLI tools are becoming preferred
  over web GUIs; OCurrent pipelines are no exceptions.
url: https://www.tunbury.org/2026/01/26/ocurrent-rpc/
date: 2026-01-26T12:00:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>As our workflows become more agentic, CLI tools are becoming preferred over web GUIs; OCurrent pipelines are no exceptions.</p>

<p>OCurrent already had an RPC endpoint which allowed functions such as listing active jobs, viewing a job log and rebuilding. <a href="https://github.com/ocurrent/ocurrent/pull/469">PR#469</a> extends this, adding full pipeline observability and control, including statistics, state, history queries, bulk rebuild, and pipeline visualisation and configuration management. This all works over <a href="https://capnproto.org">Cap’n Proto</a>.</p>

<p><code class="language-plaintext highlighter-rouge">rpc_client.ml</code> can be used as a standalone executable to query any OCurrent pipeline. Alternatively, by including the cmdliner term, your application can be its own client.</p>

<p>In the server code, the RPC endpoint must be specifically exposed. Many OCurrent applications do this already, such as the <a href="https://github.com/ocurrent/docker-base-images">Docker base image builder</a>. Any application which currently supports <code class="language-plaintext highlighter-rouge">--capnp-address</code>.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">module</span> <span class="nc">Rpc</span> <span class="o">=</span> <span class="nn">Current_rpc</span><span class="p">.</span><span class="nc">Impl</span><span class="p">(</span><span class="nc">Current</span><span class="p">)</span>

<span class="c">(* In the main function, set up Cap'n Proto serving *)</span>
<span class="k">let</span> <span class="n">serve_rpc</span> <span class="n">engine</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">config</span> <span class="o">=</span> <span class="nn">Capnp_rpc_unix</span><span class="p">.</span><span class="nn">Vat_config</span><span class="p">.</span><span class="n">create</span> <span class="o">~</span><span class="n">secret_key</span> <span class="o">~</span><span class="n">public_address</span> <span class="n">listen_address</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">service</span> <span class="o">=</span> <span class="nn">Rpc</span><span class="p">.</span><span class="n">engine</span> <span class="n">engine</span> <span class="k">in</span>
  <span class="nn">Capnp_rpc_unix</span><span class="p">.</span><span class="n">serve</span> <span class="n">config</span> <span class="n">service</span> <span class="o">&gt;&gt;=</span> <span class="k">fun</span> <span class="n">vat</span> <span class="o">-&gt;</span>
  <span class="nn">Capnp_rpc_unix</span><span class="p">.</span><span class="nn">Cap_file</span><span class="p">.</span><span class="n">save_service</span> <span class="n">vat</span> <span class="n">service</span> <span class="n">cap_file</span>
</code></pre></div></div>

<p>Then the <a href="https://github.com/dbuenzli/cmdliner">Cmdliner</a> command group needs to be added.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">client_cmd</span> <span class="o">=</span>
  <span class="nn">Current_rpc</span><span class="p">.</span><span class="nn">Client</span><span class="p">.</span><span class="nn">Cmdliner</span><span class="p">.</span><span class="n">client_cmd</span>
      <span class="o">~</span><span class="n">name</span><span class="o">:</span><span class="s2">"client"</span>
      <span class="o">~</span><span class="n">cap_file</span><span class="o">:</span><span class="s2">"/capnp-secrets/base-images.cap"</span>
      <span class="bp">()</span>

<span class="c">(* Add to your command group *)</span>
<span class="k">let</span> <span class="bp">()</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">cmds</span> <span class="o">=</span> <span class="p">[</span><span class="n">main_cmd</span><span class="p">;</span> <span class="n">client_cmd</span><span class="p">]</span> <span class="k">in</span>
  <span class="n">exit</span> <span class="o">@@</span> <span class="nn">Cmdliner</span><span class="p">.</span><span class="nn">Cmd</span><span class="p">.</span><span class="n">eval</span> <span class="p">(</span><span class="nn">Cmdliner</span><span class="p">.</span><span class="nn">Cmd</span><span class="p">.</span><span class="n">group</span> <span class="n">info</span> <span class="n">cmds</span><span class="p">)</span>
</code></pre></div></div>

<p>All 12 sub-commands are now available:-</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>base-images client overview
base-images client <span class="nb">jobs
</span>base-images client status &lt;job_id&gt;
base-images client log &lt;job_id&gt;
base-images client cancel &lt;job_id&gt;
base-images client rebuild &lt;job_id&gt;
base-images client start &lt;job_id&gt;
base-images client query <span class="o">[</span><span class="nt">--ok</span><span class="o">=</span>...] <span class="o">[</span><span class="nt">--prefix</span><span class="o">=</span>...] <span class="o">[</span><span class="nt">--op</span><span class="o">=</span>...] <span class="o">[</span><span class="nt">--rebuild</span><span class="o">=</span>...]
base-images client ops
base-images client dot
base-images client confirm <span class="o">[</span><span class="nt">--set</span><span class="o">=</span>...]
base-images client rebuild-all &lt;job_id&gt; ...
</code></pre></div></div>
