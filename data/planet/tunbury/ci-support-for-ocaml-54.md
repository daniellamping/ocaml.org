---
title: CI support for OCaml 5.4
description: Following the release of OCaml 5.4 the CI systems need to be updated
  to use it.
url: https://www.tunbury.org/2025/10/18/ci-support-for-ocaml-54/
date: 2025-10-18T00:00:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Following the release of <a href="https://ocaml.org/releases/5.4.0">OCaml 5.4</a> the CI systems need to be updated to use it.</p>

<p>This process starts with the update of <a href="https://github.com/ocurrent/ocaml-version">ocaml-version</a>, which Octachron added through <a href="https://github.com/ocurrent/ocaml-version/pull/85">PR#85</a>.</p>

<p>The base images now need to be updated, which consists of updating the <a href="https://images.ci.ocaml.org">base image builder</a>, <a href="https://github.com/ocaml/macos-infra">macos-infra</a> and <a href="https://github.com/ocurrent/freebsd-infra">freebsd-infra</a>. The latter two are Ansible scripts <a href="https://github.com/ocurrent/macos-infra/pull/57">PR#57</a> for macOS and <a href="https://github.com/ocurrent/freebsd-infra/pull/19">PR#19</a> for FreeBSD. New base images are also required for OpenBSD 7.7 and Windows Server 2022. This needs minor edits to the <code class="language-plaintext highlighter-rouge">Makefile</code>,  which are included in <a href="https://github.com/ocurrent/obuilder/pull/201">PR#201</a>.</p>

<p>The base image builder was updated with <a href="https://github.com/ocurrent/docker-base-images/pull/335">PR#335</a> which pulled in the latest <a href="https://github.com/ocurrent/ocaml-version">ocaml-version</a> and <a href="https://github.com/ocurrent/ocaml-dockerfile">ocaml-dockerfile</a>. <a href="https://github.com/ocurrent/ocaml-dockerfile">ocaml-dockerfile</a> contains the build instructions for the base images across different OS distributions and architectures as Dockerfiles.</p>

<p><a href="https://github.com/ocurrent/ocaml-dockerfile">ocaml-dockerfile</a> had recently been updated with <a href="https://github.com/ocurrent/ocaml-dockerfile/pull/243">PR#243</a>, which added CentOS Stream 9 and 10, Oracle Linux 10 and Ubuntu 25.10. However, this resulted in a couple of build failures, plus MisterDA opened <a href="https://github.com/ocurrent/ocaml-dockerfile/issues/244">issue#244</a>, noting openSUSE and Windows Server 2025 needed to be updated.</p>

<p>There were build failures in OpenSUSE that came from <code class="language-plaintext highlighter-rouge">RUN yum install -y ... curl ...</code> which conflicted with the <code class="language-plaintext highlighter-rouge">curl</code> which was already installed. Easily fixed by removing <code class="language-plaintext highlighter-rouge">curl</code> as it was already installed.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>#17 1.503 Error: 
#17 1.503  Problem: problem with installed package curl-minimal-7.76.1-34.el9.x86_64
#17 1.503   - package curl-minimal-7.76.1-34.el9.x86_64 from @System conflicts with curl provided by curl-7.76.1-34.el9.x86_64 from baseos
#17 1.503   - package curl-minimal-7.76.1-26.el9.x86_64 from baseos conflicts with curl provided by curl-7.76.1-34.el9.x86_64 from baseos
#17 1.503   - package curl-minimal-7.76.1-28.el9.x86_64 from baseos conflicts with curl provided by curl-7.76.1-34.el9.x86_64 from baseos
#17 1.503   - package curl-minimal-7.76.1-29.el9.x86_64 from baseos conflicts with curl provided by curl-7.76.1-34.el9.x86_64 from baseos
#17 1.503   - package curl-minimal-7.76.1-31.el9.x86_64 from baseos conflicts with curl provided by curl-7.76.1-34.el9.x86_64 from baseos
#17 1.503   - package curl-minimal-7.76.1-34.el9.x86_64 from baseos conflicts with curl provided by curl-7.76.1-34.el9.x86_64 from baseos
#17 1.503   - cannot install the best candidate for the job
</code></pre></div></div>

<p>The next issue was with <code class="language-plaintext highlighter-rouge">RUN yum config-manager --set-enabled powertools</code> as this repository had changed its name (again):</p>

<ul>
  <li>CentOS 7: Uses yum-config-manager</li>
  <li>CentOS 8: Uses powertools</li>
  <li>CentOS Stream 9+: Uses crb</li>
</ul>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>#34 [stage-1 13/41] RUN yum config-manager --set-enabled powertools
#34 0.447 Error: No matching repo to modify: powertools.
#34 ERROR: process "/bin/sh -c yum config-manager --set-enabled powertools" did not complete successfully: exit code: 1
------
 &gt; [stage-1 13/41] RUN yum config-manager --set-enabled powertools:
0.447 Error: No matching repo to modify: powertools.
------
</code></pre></div></div>

<p>The final blocker was building the Ubuntu 25.10 images on RISCV. These images failed on <code class="language-plaintext highlighter-rouge">apt-get update</code>, which I initially assumed was a transitory network issue, but it persisted.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>#14 [stage-0  2/13] RUN apt-get -y update
#14 ERROR: process "/bin/sh -c apt-get -y update" did not complete successfully: exit code: 132
</code></pre></div></div>

<p>Oddly, <code class="language-plaintext highlighter-rouge">docker run --rm -it ubuntu:questing</code> didn’t give me a container and simply returned the command prompt. However, <code class="language-plaintext highlighter-rouge">docker run --rm -it ubuntu:questing-20250830</code> did give me a prompt but I still couldn’t run <code class="language-plaintext highlighter-rouge">apt</code>:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code># docker run --rm -it ubuntu:questing-20250830
root@8754b6373f6f:/# apt update   
Illegal instruction (core dumped)
</code></pre></div></div>

<p>Interestingly, <code class="language-plaintext highlighter-rouge">ubuntu:questing-20250806</code> (even older) could run <code class="language-plaintext highlighter-rouge">apt update</code>. However, attempting to build the Dockerfile didn’t work.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>48.49 Preparing to unpack .../libc6_2.42-0ubuntu3_riscv64.deb ...
49.27 Checking for services that may need to be restarted...
49.30 Checking init scripts...
49.30 Checking for services that may need to be restarted...
49.34 Checking init scripts...
49.34 Nothing to restart.
49.44 Unpacking libc6:riscv64 (2.42-0ubuntu3) over (2.41-9ubuntu1) ...
52.05 dpkg: warning: old libc6:riscv64 package post-removal script subprocess was killed by signal (Illegal instruction), core dumped
52.05 dpkg: trying script from the new package instead ...
52.06 dpkg: error processing archive /var/cache/apt/archives/libc6_2.42-0ubuntu3_riscv64.deb (--unpack):
52.06  new libc6:riscv64 package post-removal script subprocess was killed by signal (Illegal instruction), core dumped
52.07 dpkg: error while cleaning up:
52.07  installed libc6:riscv64 package pre-installation script subprocess was killed by signal (Illegal instruction), core dumped
52.30 Errors were encountered while processing:
52.30  /var/cache/apt/archives/libc6_2.42-0ubuntu3_riscv64.deb
52.46 E: Sub-process /usr/bin/dpkg returned an error code (1)
</code></pre></div></div>

<p>Checking the Ubuntu <a href="https://ubuntu.com/download/risc-v">download</a> page shows that Ubuntu have changed the hardware requirements.</p>

<blockquote>
  <p>We have upgraded the required RISC-V ISA profile to RVA23S64 with the 25.10 release. Hardware that is not RVA23 ready continues to be supported by our 24.04.3 LTS release.</p>
</blockquote>

<p>Searching online found this <a href="https://www.phoronix.com/news/Ubuntu-25.10-RISC-V-QEMU">article</a>.</p>

<blockquote>
  <p>Back in June it was announced by Canonical that for the Ubuntu 25.10 release <a href="https://www.phoronix.com/news/Ubuntu-25.10-To-Require-RVA23">they would be raising the RISC-V baseline to the RVA23 profile even with barely any available RISC-V platforms supporting that newer RISC-V profile</a>. That change is still going ahead and leaves Ubuntu 25.10 on RISC-V currently only supporting the QEMU virtualized target.</p>
</blockquote>

<p>Therefore, I have removed RISCV as a supported platform on for Ubuntu 25.10 until we can get some hardware to support it or set up some QEMU workers.</p>

<p>Additionally, Anil suggested dropping support for Debian 11, Oracle Linux 8 and 9, and Fedora 41 to reduce the size of the build matrix.</p>

<p><a href="https://github.com/ocurrent/ocaml-dockerfile">ocaml-dockerfile</a> release 8.3.3 is now pending on <a href="https://github.com/ocaml/opam-repository/pull/28736">opam repository</a>.</p>

<p>Now that the base images have been successfully built, I can continue with the updates to <a href="https://github.com/ocurrent/opam-repo-ci">ocurrent/opam-repo-ci</a> with <a href="https://github.com/ocurrent/opam-repo-ci/pull/460">PR#460</a>, which only needs the opam repository SHA updated to include the new release of ocaml-version.</p>

<p><a href="https://github.com/ocurrent/ocaml-ci">ocurrent/ocaml-ci</a> uses git submodules for these packages, so these need to be updated: <a href="https://github.com/ocurrent/ocaml-ci/pull/1032">PR#1042</a>.</p>
