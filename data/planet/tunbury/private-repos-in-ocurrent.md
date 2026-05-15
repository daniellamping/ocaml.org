---
title: Private repos in OCurrent
description: OCurrent has long wanted to access private repositories. You can achieve
  this by embedding a scoped PAT in the .git-credentials file, typically within the
  Docker container; however, this is untidy, to say the least! The approach presented
  works in cases where a GitHub app is used.
url: https://www.tunbury.org/2025/12/05/ocurrent-private-repos/
date: 2025-12-05T11:30:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p><a href="https://github.com/ocurrent/ocurrent">OCurrent</a> has long wanted to access private repositories. You can achieve this by embedding a scoped PAT in the <code class="language-plaintext highlighter-rouge">.git-credentials</code> file, typically within the Docker container; however, this is untidy, to say the least! The approach presented works in cases where a GitHub app is used.</p>

<p>OCurrent authenticates to GitHub using a JWT (JSON Web Token). This token is signed using the application’s RSA private key (from <code class="language-plaintext highlighter-rouge">--github-private-key-file</code>) and contains the app_id. GitHub verifies this signature to confirm it’s really from the GitHub app. OCurrent then calls <code class="language-plaintext highlighter-rouge">get_token</code>, which POSTs to GitHub’s API to get an installation access token. This is a short-lived token (60 min) that can access the repositories the app has permission to. In summary, OCurrent already has the token, but there is no accessor function.</p>

<p>Git supports the <code class="language-plaintext highlighter-rouge">https://x-access-token:ghs_XXXX@github.com/...</code> access method to pass the password; however, OCurrent displays logs in real-time, so this would show in plain text on the web GUI. You can pass a custom pretty-print function and use it to mask the value. Alternatively, you can pass an environment variable to <code class="language-plaintext highlighter-rouge">git</code>, for example <code class="language-plaintext highlighter-rouge">GIT_CONFIG_PARAMETERS="'http.extraHeader=Authorization: Basic dXNlcjpwYXNz'"</code>.</p>

<p>I have added <code class="language-plaintext highlighter-rouge">get_cached_token</code>, which returns the cached token from the GitHub API plugin. Essentially, this is <code class="language-plaintext highlighter-rouge">let get_cached_token t = t.token</code>. This token then becomes the context parameter for the <code class="language-plaintext highlighter-rouge">git fetch</code> operation, replacing the original <code class="language-plaintext highlighter-rouge">No_context</code>.</p>

<p>The environment variable is created by calling <code class="language-plaintext highlighter-rouge">Base64.encode_string</code> on the <code class="language-plaintext highlighter-rouge">x-access-token:ghs_XXXX</code>.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">make_auth_env</span> <span class="n">token</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">b64</span> <span class="o">=</span> <span class="nn">Base64</span><span class="p">.</span><span class="n">encode_string</span> <span class="p">(</span><span class="s2">"x-access-token:"</span> <span class="o">^</span> <span class="n">token</span><span class="p">)</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">header</span> <span class="o">=</span> <span class="nn">Printf</span><span class="p">.</span><span class="n">sprintf</span> <span class="s2">"'http.extraHeader=Authorization: Basic %s'"</span> <span class="n">b64</span> <span class="k">in</span>
  <span class="p">[</span><span class="o">|</span> <span class="s2">"GIT_CONFIG_PARAMETERS="</span> <span class="o">^</span> <span class="n">header</span> <span class="o">|</span><span class="p">]</span>
</code></pre></div></div>

<p>The remaining changes in the PR thread the <code class="language-plaintext highlighter-rouge">env</code> parameter through the <code class="language-plaintext highlighter-rouge">git</code> module to the <code class="language-plaintext highlighter-rouge">process</code> module, where it is ultimately passed to <code class="language-plaintext highlighter-rouge">Lwt_process.open_process</code>.</p>

<p>Therefore, considering the example, <code class="language-plaintext highlighter-rouge">doc/examples/github_app.ml</code>, the diff would be:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code>   <span class="nn">Github</span><span class="p">.</span><span class="nn">App</span><span class="p">.</span><span class="n">installations</span> <span class="n">app</span> <span class="o">|&gt;</span> <span class="nn">Current</span><span class="p">.</span><span class="n">list_iter</span> <span class="p">(</span><span class="k">module</span> <span class="nn">Github</span><span class="p">.</span><span class="nc">Installation</span><span class="p">)</span> <span class="o">@@</span> <span class="k">fun</span> <span class="n">installation</span> <span class="o">-&gt;</span>
<span class="o">+</span>  <span class="nn">Current</span><span class="p">.</span><span class="n">component</span> <span class="s2">"api"</span> <span class="o">|&gt;</span>
<span class="o">+</span>  <span class="k">let</span><span class="o">**</span> <span class="n">inst</span> <span class="o">=</span> <span class="n">installation</span> <span class="k">in</span>
<span class="o">+</span>  <span class="k">let</span> <span class="n">github</span> <span class="o">=</span> <span class="nn">Github</span><span class="p">.</span><span class="nn">Installation</span><span class="p">.</span><span class="n">api</span> <span class="n">inst</span> <span class="k">in</span>
   <span class="k">let</span> <span class="n">repos</span> <span class="o">=</span> <span class="nn">Github</span><span class="p">.</span><span class="nn">Installation</span><span class="p">.</span><span class="n">repositories</span> <span class="n">installation</span> <span class="k">in</span>
   <span class="n">repos</span> <span class="o">|&gt;</span> <span class="nn">Current</span><span class="p">.</span><span class="n">list_iter</span> <span class="o">~</span><span class="n">collapse_key</span><span class="o">:</span><span class="s2">"repo"</span> <span class="p">(</span><span class="k">module</span> <span class="nn">Github</span><span class="p">.</span><span class="nn">Api</span><span class="p">.</span><span class="nc">Repo</span><span class="p">)</span> <span class="o">@@</span> <span class="k">fun</span> <span class="n">repo</span> <span class="o">-&gt;</span>
   <span class="nn">Github</span><span class="p">.</span><span class="nn">Api</span><span class="p">.</span><span class="nn">Repo</span><span class="p">.</span><span class="n">ci_refs</span> <span class="o">~</span><span class="n">staleness</span><span class="o">:</span><span class="p">(</span><span class="nn">Duration</span><span class="p">.</span><span class="n">of_day</span> <span class="mi">90</span><span class="p">)</span> <span class="n">repo</span>
   <span class="o">|&gt;</span> <span class="nn">Current</span><span class="p">.</span><span class="n">list_iter</span> <span class="p">(</span><span class="k">module</span> <span class="nn">Github</span><span class="p">.</span><span class="nn">Api</span><span class="p">.</span><span class="nc">Commit</span><span class="p">)</span> <span class="o">@@</span> <span class="k">fun</span> <span class="n">head</span> <span class="o">-&gt;</span>
<span class="o">-</span>  <span class="k">let</span> <span class="n">src</span> <span class="o">=</span> <span class="nn">Git</span><span class="p">.</span><span class="n">fetch</span> <span class="p">(</span><span class="nn">Current</span><span class="p">.</span><span class="n">map</span> <span class="nn">Github</span><span class="p">.</span><span class="nn">Api</span><span class="p">.</span><span class="nn">Commit</span><span class="p">.</span><span class="n">id</span> <span class="n">head</span><span class="p">)</span> <span class="k">in</span>
<span class="o">+</span>  <span class="k">let</span> <span class="n">token</span> <span class="o">=</span> <span class="nn">Github</span><span class="p">.</span><span class="nn">Api</span><span class="p">.</span><span class="n">get_cached_token</span> <span class="n">github</span> <span class="k">in</span>
<span class="o">+</span>  <span class="k">let</span> <span class="n">src</span> <span class="o">=</span> <span class="nn">Git</span><span class="p">.</span><span class="n">fetch</span> <span class="o">?</span><span class="n">token</span> <span class="p">(</span><span class="nn">Current</span><span class="p">.</span><span class="n">map</span> <span class="nn">Github</span><span class="p">.</span><span class="nn">Api</span><span class="p">.</span><span class="nn">Commit</span><span class="p">.</span><span class="n">id</span> <span class="n">head</span><span class="p">)</span> <span class="k">in</span>
   <span class="nn">Docker</span><span class="p">.</span><span class="n">build</span> <span class="o">~</span><span class="n">pool</span> <span class="o">~</span><span class="n">pull</span><span class="o">:</span><span class="bp">false</span> <span class="o">~</span><span class="n">dockerfile</span> <span class="p">(</span><span class="nt">`Git</span> <span class="n">src</span><span class="p">)</span>
   <span class="o">|&gt;</span> <span class="n">check_run_status</span>
   <span class="o">|&gt;</span> <span class="nn">Github</span><span class="p">.</span><span class="nn">Api</span><span class="p">.</span><span class="nn">CheckRun</span><span class="p">.</span><span class="n">set_status</span> <span class="n">head</span> <span class="n">program_name</span>
</code></pre></div></div>

<p>This adds an <code class="language-plaintext highlighter-rouge">api</code> node in the graph for each installation, which is semantically correct as the token is per organisation.</p>

<p>I considered that the token might be stale or uninitialised before the <code class="language-plaintext highlighter-rouge">Git.fetch</code> call, but the only way to get a <code class="language-plaintext highlighter-rouge">Github.Api.Commit.id</code> is through an API call, so the token will always be refreshed. When a webhook is received, it triggers the reevaluation of the graph, which again refreshes the API token.</p>

<p>ref <a href="https://github.com/ocurrent/ocurrent/pull/466">ocurrent/ocurrent PR#466</a></p>
