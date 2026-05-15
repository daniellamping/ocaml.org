---
title: Weird Windows container version numbers
description: "Running ver in a Windows container doesn\u2019t report the version number
  that you expect."
url: https://www.tunbury.org/2026/05/05/windows-container-ver/
date: 2026-05-05T09:00:00-00:00
preview_image: https://www.tunbury.org/images/docker-base-images-error.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Running <code class="language-plaintext highlighter-rouge">ver</code> in a Windows container doesn’t report the version number that you expect.</p>

<p><a href="https://github.com/ocurrent/docker-base-images">ocurrent/docker-base-images</a> publishes the Docker images that the OCaml CI pipeline systems use. For Windows, it pulls the generic LTSC tag, runs the container to determine the exact version, and then uses that exact tag for the builds.</p>

<p>On Windows 2022 server, pulling <code class="language-plaintext highlighter-rouge">mcr.microsoft.com/windows/server:ltsc2022</code> then running <code class="language-plaintext highlighter-rouge">ver</code> produces <code class="language-plaintext highlighter-rouge">10.0.20348.5020</code> exactly as you would expect, but running that same sequence on a Windows 2025 host returns <code class="language-plaintext highlighter-rouge">10.0.26100.5020</code>.</p>

<p><code class="language-plaintext highlighter-rouge">10.0.26100.5020</code> is not a real Windows release.</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">10.0.20348.x</code> Windows Server 2022 (LTSC, build 20348)</li>
  <li><code class="language-plaintext highlighter-rouge">10.0.26100.x</code> Windows Server 2025 / Windows 11 24H2 (build 26100)</li>
</ul>

<p>The 5020 UBR belongs to the 20348; the 26100 has its own UBR sequence. The combination <code class="language-plaintext highlighter-rouge">26100.5020</code> does not correspond to any released Windows version, so the tag <code class="language-plaintext highlighter-rouge">10.0.26100.5020</code> can’t be pulled.</p>

<p>Rather than running <code class="language-plaintext highlighter-rouge">ver</code>, I changed the probe command to query the containers registry at <code class="language-plaintext highlighter-rouge">HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion</code>, which does report the correct image build version rather than a partial kernel version number:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>CurrentMajorVersionNumber : 10
CurrentMinorVersionNumber : 0
CurrentBuildNumber        : 20348
UBR                       : 4170
</code></pre></div></div>

<p>In the code, this <code class="language-plaintext highlighter-rouge">ver</code> command changed</p>

<pre><code class="language-cmd">for /f "tokens=4 delims=[] " %a in ('ver') do echo %a
</code></pre>

<p>to this PowerShell query:</p>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$k</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">Get-ItemProperty</span><span class="w"> </span><span class="s1">'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion'</span><span class="w">
</span><span class="s1">'{0}.{1}.{2}.{3}'</span><span class="w"> </span><span class="nt">-f</span><span class="w"> </span><span class="nv">$k</span><span class="o">.</span><span class="nf">CurrentMajorVersionNumber</span><span class="p">,</span><span class="w"> </span><span class="nv">$k</span><span class="o">.</span><span class="nf">CurrentMinorVersionNumber</span><span class="p">,</span><span class="w"> </span><span class="nv">$k</span><span class="o">.</span><span class="nf">CurrentBuildNumber</span><span class="p">,</span><span class="w"> </span><span class="nv">$k</span><span class="o">.</span><span class="nf">UBR</span><span class="w">
</span></code></pre></div></div>

<p>The change is in <a href="https://github.com/ocurrent/docker-base-images/pull/348">PR#348</a></p>
