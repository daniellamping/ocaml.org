---
title: Moving to opam 2.5
description: opam 2.5.0 was released on 27th November, and this update needs to be
  propagated through the CI infrastructure. This post mirrors the steps taken for
  the release of opam 2.4.1.
url: https://www.tunbury.org/2026/01/12/opam-25/
date: 2026-01-12T21:00:00-00:00
preview_image: https://www.tunbury.org/images/opam.png
authors:
- Mark Elvers
source:
ignore:
---

<p><a href="https://opam.ocaml.org/blog/opam-2-5-0/">opam 2.5.0</a> was released on 27th November, and this update needs to be propagated through the CI infrastructure. This post mirrors the steps taken for the release of <a href="https://www.tunbury.org/2025/07/30/opam-24/">opam 2.4.1</a>.</p>

<h1>Base Images</h1>

<h2>Linux</h2>

<p><a href="https://github.com/ocurrent/docker-base-images">ocurrent/docker-base-images</a></p>

<p>The Linux base images are created using the <a href="https://images.ci.ocaml.org/">Docker base image builder</a>, which uses <a href="https://github.com/ocurrent/ocaml-dockerfile">ocurrent/ocaml-dockerfile</a> to know which versions of opam are available. Antonin submitted <a href="https://github.com/ocurrent/ocaml-dockerfile/pull/249">PR#249</a> with the necessary changes. This was released as v8.3.4.</p>

<p>With v8.3.4 released, <a href="https://github.com/ocurrent/docker-base-images/pull/336">PR#336</a> can be opened to update the pipeline to build images which include opam 2.5. Rebuilding the base images requires a significant amount of time, especially since it’s marked as a low-priority task on the cluster.</p>

<h2>macOS</h2>

<p><a href="https://github.com/ocurrent/macos-infra">ocurrent/macos-infra</a></p>

<p>Including opam 2.5 in the macOS required <a href="https://github.com/ocurrent/macos-infra/pull/58">PR#58</a>, which adds 2.5 to the list of opam packages to download. There are Ansible playbooks that build the macOS base images and recursively remove the old images and their (ZFS) clones. They take about half an hour per machine. I run the Intel and Apple Silicon updates in parallel, but process each pool one at a time.</p>

<p>The Ansible command is: <code class="language-plaintext highlighter-rouge">ansible-playbook update-ocluster.yml</code></p>

<h2>FreeBSD</h2>

<p><a href="https://github.com/ocurrent/freebsd-infra">ocurrent/freebsd-infra</a></p>

<p>The FreeBSD update parallels the macOS update, requiring that 2.5 be added to the loop of available versions. <a href="https://github.com/ocurrent/freebsd-infra/pull/20">PR#20</a>.</p>

<p>The Ansible command is: <code class="language-plaintext highlighter-rouge">ansible-playbook update.yml</code></p>

<h2>Windows (thyme.caelum.ci.dev)</h2>

<p><a href="https://github.com/ocurrent/obuilder">ocurrent/obuilder</a></p>

<p>The Windows base images are built using a <code class="language-plaintext highlighter-rouge">Makefile</code> which runs unattended builds of Windows using QEMU virtual machines. The Makefile requires <a href="https://github.com/ocurrent/obuilder/pull/202">PR#202</a>. The build command is <code class="language-plaintext highlighter-rouge">make windows</code>.</p>

<p>Once the new images have been built, stop <code class="language-plaintext highlighter-rouge">ocluster worker</code> and move the new base images into place. The next is to remove <code class="language-plaintext highlighter-rouge">results/*</code> as these layers will link to the old base images, and remove <code class="language-plaintext highlighter-rouge">state/*</code> so obuilder will create a new empty database on startup. Avoid removing <code class="language-plaintext highlighter-rouge">cache/*</code> as this is the download cache for opam objects.</p>

<p>The unattended installation can be monitored via VNC by connecting to localhost:5900.</p>

<h2>OpenBSD (oregano.caelum.ci.dev)</h2>

<p><a href="https://github.com/ocurrent/obuilder">ocurrent/obuilder</a></p>

<p>The OpenBSD base images are built using the same Makefile used for Windows. There is a separate commit in <a href="https://github.com/ocurrent/obuilder/pull/202">PR#202</a> for the changes needed for OpenBSD, which include moving from OpenBSD 7.6 to 7.7. Run <code class="language-plaintext highlighter-rouge">make openbsd</code>.</p>

<p>Once the new images have been built, stop <code class="language-plaintext highlighter-rouge">ocluster worker</code> and move the new base images into place. The next is to remove <code class="language-plaintext highlighter-rouge">results/*</code> as these layers will link to the old base images, and remove <code class="language-plaintext highlighter-rouge">state/*</code> so obuilder will create a new empty database on startup. Avoid removing <code class="language-plaintext highlighter-rouge">cache/*</code> as this is the download cache for opam objects.</p>

<p>As with Windows, the unattended installation can be monitored via VNC by connecting to localhost:5900.</p>

<h1>OCaml-CI</h1>

<p><a href="https://ocaml.ci.dev">OCaml-CI</a> uses <a href="https://github.com/ocurrent/ocaml-dockerfile">ocurrent/ocaml-dockerfile</a> as a submodule, so the module needs to be updated to the released version. Edits are needed to <code class="language-plaintext highlighter-rouge">lib/opam_version.ml</code> to include V2_5, then the pipeline needs to be updated in <code class="language-plaintext highlighter-rouge">service/conf.ml</code> to use version 2.5 rather than 2.4 for all the different operating systems. Linux is rather more automated than the others.</p>

<h1>opam-repo-ci</h1>

<p><a href="https://opam.ci.ocaml.org">opam-repo-ci</a> tests using the latest tagged version of opam, which is called <code class="language-plaintext highlighter-rouge">opam-dev</code> within the base images. It also explicitly tests against the latest release in each of the 2.x series. With 2.5 being tagged, this will automatically become the used <code class="language-plaintext highlighter-rouge">dev</code> version once the base images are updated, but over time, 2.5 and the latest tagged version will diverge, so <a href="https://github.com/ocurrent/opam-repo-ci/pull/463">PR#463</a> is needed to ensure we continue to test with the released version of 2.5.</p>
