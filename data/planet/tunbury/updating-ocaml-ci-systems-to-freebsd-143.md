---
title: Updating OCaml CI systems to FreeBSD 14.3
description: The FreeBSD CI worker rosemary needs to be updated to FreeBSD 14.3.
url: https://www.tunbury.org/2025/10/07/freebsd-14.3/
date: 2025-10-07T00:00:00-00:00
preview_image: https://www.tunbury.org/images/freebsd-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>The FreeBSD CI worker <code class="language-plaintext highlighter-rouge">rosemary</code> needs to be updated to FreeBSD 14.3.</p>

<p>The upgrade went without issue following the notes from last <a href="https://www.tunbury.org/2025/03/26/freebsd-14.2/">time</a>. <a href="https://github.com/ocurrent/freebsd-infra">ocurrent/freebsd-infra</a> was updated with <a href="https://github.com/ocurrent/freebsd-infra/pull/18">PR#18</a> and the base images were recreated with <code class="language-plaintext highlighter-rouge">ansible-playbook update.yml</code>.</p>

<p><a href="https://github.com/ocurrent/ocaml-ci/pull/1029">PR#1029</a> for OCaml CI and <a href="https://github.com/ocurrent/opam-repo-ci/pull/459">PR#459</a> for opam-repo-ci were pushed to their respective live branches.</p>
