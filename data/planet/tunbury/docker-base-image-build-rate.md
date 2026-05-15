---
title: Docker base image build rate
description: 'We are increasingly hitting the Docker Hub rate limits when pushing
  the Docker base images. This issue was previously identified in issue #267. However,
  this is now becoming critical as many more jobs are failing.'
url: https://www.tunbury.org/2025/10/10/docker-base-images/
date: 2025-10-10T00:00:00-00:00
preview_image: https://www.tunbury.org/images/docker-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>We are increasingly hitting the Docker Hub rate limits when pushing the Docker base images. This issue was previously identified in <a href="https://github.com/ocurrent/docker-base-images/issues/267">issue #267</a>. However, this is now becoming critical as many more jobs are failing.</p>

<p>A typical failure log looks like this:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>#13 [1/7] FROM docker.io/ocurrent/opam-staging@sha256:8ff156dd3a4ad8853b82940ac8965e8f0f4b18245e54fb26b9304f1ab961030b
#13 sha256:6b6519a49e416508fe7152b16035ad70bebba4d8f3486b6c0732c21da9433445
#13 resolve docker.io/ocurrent/opam-staging@sha256:8ff156dd3a4ad8853b82940ac8965e8f0f4b18245e54fb26b9304f1ab961030b
#13 resolve docker.io/ocurrent/opam-staging@sha256:8ff156dd3a4ad8853b82940ac8965e8f0f4b18245e54fb26b9304f1ab961030b 1.6s done
#13 ERROR: failed to copy: httpReadSeeker: failed open: unexpected status from GET request to https://registry-1.docker.io/v2/ocurrent/opam-staging/manifests/sha256:8ff156dd3a4ad8853b82940ac8965e8f0f4b18245e54fb26b9304f1ab961030b: 429 Too Many Requests
toomanyrequests: You have reached your unauthenticated pull rate limit. https://www.docker.com/increase-rate-limit
------
 &gt; [1/7] FROM docker.io/ocurrent/opam-staging@sha256:8ff156dd3a4ad8853b82940ac8965e8f0f4b18245e54fb26b9304f1ab961030b:
------
failed to load cache key: failed to copy: httpReadSeeker: failed open: unexpected status from GET request to https://registry-1.docker.io/v2/ocurrent/opam-staging/manifests/sha256:8ff156dd3a4ad8853b82940ac8965e8f0f4b18245e54fb26b9304f1ab961030b: 429 Too Many Requests
toomanyrequests: You have reached your unauthenticated pull rate limit. https://www.docker.com/increase-rate-limit
docker-build failed with exit-code 1
</code></pre></div></div>

<p>In the base image builder, we create our OCluster connection using the defaults:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code>  <span class="k">let</span> <span class="n">connection</span> <span class="o">=</span> <span class="nn">Current_ocluster</span><span class="p">.</span><span class="nn">Connection</span><span class="p">.</span><span class="n">create</span> <span class="n">submission_cap</span> <span class="k">in</span>
</code></pre></div></div>

<p>Looking at <a href="https://github.com/ocurrent/ocluster/blob/ba26623c6bca8b917c4252fa9739313fb14692ea/ocurrent-plugin/connection.ml#L177">ocurrent/ocluster</a>, the default is 200 jobs <em>per pool</em>. We submit to 6 pools with a rate limit of 200 per pool, resulting in an overall limit of 1,200 jobs.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">create</span> <span class="o">?</span><span class="p">(</span><span class="n">max_pipeline</span><span class="o">=</span><span class="mi">200</span><span class="p">)</span> <span class="n">sr</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">rate_limits</span> <span class="o">=</span> <span class="nn">Hashtbl</span><span class="p">.</span><span class="n">create</span> <span class="mi">10</span> <span class="k">in</span>
  <span class="p">{</span> <span class="n">sr</span><span class="p">;</span> <span class="n">sched</span> <span class="o">=</span> <span class="nn">Lwt</span><span class="p">.</span><span class="n">fail_with</span> <span class="s2">"init"</span><span class="p">;</span> <span class="n">rate_limits</span><span class="p">;</span> <span class="n">max_pipeline</span> <span class="p">}</span>
</code></pre></div></div>

<p>The current <code class="language-plaintext highlighter-rouge">builds.expected</code> file defines 1029 builds. The first 50 jobs building opam can run immediately; then, all the rest of the builds are unleashed. The breakdown of those follow-up compiler builds by pool is as follows: 352 for amd64, 232 for arm64, 102 for ppc64, 102 for s390x, 69 for riscv64, and 28 for Windows.</p>

<p><a href="https://github.com/ocurrent/docker-base-images/pull/333">PR#333</a> reduces the rate to 20 builds per pool.</p>
