---
title: Keeping your branch up-to-date
description: My Arm32 branch will quickly go stale and will need to be rebased and
  tested. Can GitHub Actions do that for me automatically?
url: https://www.tunbury.org/2025/12/01/github-actions/
date: 2025-12-01T23:20:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>My Arm32 branch will quickly go stale and will need to be rebased and tested. Can GitHub Actions do that for me automatically?</p>

<p>Adding a self-hosted runner is pretty straightforward. Go to your repository, then navigate to Settings, Actions, Runners, and click “New self-hosted runner”. Select your OS and architecture, and the customised installation instructions are provided:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Create a folder</span>
<span class="nv">$ </span><span class="nb">mkdir </span>actions-runner <span class="o">&amp;&amp;</span> <span class="nb">cd </span>actions-runner
<span class="c"># Download the latest runner package</span>
<span class="nv">$ </span>curl <span class="nt">-o</span> actions-runner-linux-arm-2.329.0.tar.gz <span class="nt">-L</span> https://github.com/actions/runner/releases/download/v2.329.0/actions-runner-linux-arm-2.329.0.tar.gz
<span class="c"># Optional: Validate the hash</span>
<span class="nv">$ </span><span class="nb">echo</span> <span class="s2">"b958284b8af869bd6d3542210fbd23702449182ba1c2b1b1eef575913434f13a  actions-runner-linux-arm-2.329.0.tar.gz"</span> | shasum <span class="nt">-a</span> 256 <span class="nt">-c</span>
<span class="c"># Extract the installer</span>
<span class="nv">$ </span><span class="nb">tar </span>xzf ./actions-runner-linux-arm-2.329.0.tar.gz
</code></pre></div></div>

<p>Then the configuration as follows:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># Create the runner and start the configuration experience</span>
<span class="nv">$ </span>./config.sh <span class="nt">--url</span> https://github.com/mtelvers/ocaml <span class="nt">--token</span> YOUR_TOKEN
<span class="c"># Last step, run it!</span>
<span class="nv">$ </span>./run.sh
</code></pre></div></div>

<p>I choose not to run it and instead configure it to run via systemd using:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo</span> ./svc.sh <span class="nb">install</span>
<span class="nv">$ </span><span class="nb">sudo</span> ./svc.sh start
</code></pre></div></div>

<p>My problems began as my Raspbian OS was out of date, and the GitHub runner requires Node.js 20. Runner version 2.303.0, which uses Node.js 16, was still available, so I installed it from <code class="language-plaintext highlighter-rouge">https://github.com/actions/runner/releases/download/v2.303.0/actions-runner-linux-arm-2.303.0.tar.gz</code>. This installation was successful, but it immediately updated itself to 2.329.0, resulting in the same problem.</p>

<p>Adding <code class="language-plaintext highlighter-rouge">--disableupdate</code> to <code class="language-plaintext highlighter-rouge">config.sh</code> prevented behaviour, but the error message was now terminal:</p>

<blockquote>
  <p>runsvc.sh[20543]: An error occurred: Runner version v2.303.0 is deprecated and cannot receive messages.</p>
</blockquote>

<p>I updated the OS to the latest Raspberry Pi OS (32-bit) based on Debian Trixie, and the installation completed as expected. My runner was now ready.</p>

<p>Scheduled workflows only run on the default branch, so I changed my fork’s default branch to <code class="language-plaintext highlighter-rouge">arm32-multicore</code> and committed a GitHub Action workflow, as shown in <a href="https://gist.github.com/mtelvers/c08b324cab705cf0ad84f04f3e79a9ab">this gist</a>. The workflow checks out my branch, rebases it on <code class="language-plaintext highlighter-rouge">upstream/trunk</code>, builds the compiler and runs the test suite.</p>
