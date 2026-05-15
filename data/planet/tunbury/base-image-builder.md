---
title: Base Image Builder
description: "The base image builder has a growing number of failed builds; it\u2019s
  time to address these."
url: https://www.tunbury.org/2026/01/16/base-image-builder/
date: 2026-01-16T17:20:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>The base image builder has a growing number of failed builds; it’s time to address these.</p>

<h1>OCaml &lt; 5.1 with GCC &gt;= 15</h1>

<p>Distributions that have moved to GCC 15 have had failing builds since last <a href="https://github.com/ocurrent/docker-base-images/issues/320">April</a>. This affects builds older than OCaml 5.1.1 but not OCaml 4.14.2.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code># gcc -c -O2 -fno-strict-aliasing -fwrapv -pthread -g -Wall -fno-common -fexcess-precision=standard -ffunction-sections  -I./runtime  -D_FILE_OFFSET_BITS=64  -DCAMLDLLIMPORT= -DIN_CAML_RUNTIME -DDEBUG  -o runtime/main.bd.o runtime/main.c
In file included from runtime/interp.c:34:
runtime/interp.c: In function 'caml_interprete':
runtime/caml/prims.h:33:23: error: too many arguments to function '(value (*)(void))*(caml_prim_table.contents + (sizetype)((long unsigned int)*pc * 8))'; expected 0, have 1
33 | #define Primitive(n) ((c_primitive)(caml_prim_table.contents[n]))
   |                      ~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
runtime/interp.c:1037:14: note: in expansion of macro 'Primitive'
1037 |       accu = Primitive(*pc)(accu);
     |              ^~~~~~~~~
</code></pre></div></div>

<p>I was about to create the patches, but I noticed that @dra27 had already done so. <a href="https://github.com/ocaml-opam/ocaml/branches">ocaml-opam/ocaml</a>. The patches can be added as an overlay repository. I have done this before for GCC 14 when a similar issue occurred for OCaml &lt; 4.08. <a href="https://github.com/ocurrent/docker-base-images/pull/298">PR#298</a>. The new PR is <a href="https://github.com/ocurrent/docker-base-images/pull/337">PR#337</a></p>

<h1>Ubuntu 25.10</h1>

<p>The GCC 15 patch resolved most Ubuntu issues, but Ubuntu 25.10 persisted. Ubuntu 25.10 switched to the Rust-based Coreutils, which does not support commas in the install command until version 0.5.0. Ubuntu 25.10 ships with 0.2.2.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>#9 137.0 # /usr/bin/install -c -m u=rw,g=rw,o=r \
#9 137.0 #   VERSION \
#9 137.0 #   "/home/opam/.opam/4.09/lib/ocaml"
#9 137.0 # /usr/bin/install: Invalid mode string: invalid operator (expected +, -, or =, but found ,)
</code></pre></div></div>

<p><a href="https://github.com/ocurrent/ocaml-dockerfile/pull/255">PR#255</a> switches to GNU Coreutils. I expect this problem will be cleared in subsequent releases of Ubuntu.</p>

<h1>Windows</h1>

<p>The Windows workers needed to be updated to Windows Server 2025, as older kernels cannot run newer containers. Furthermore, the OCluster code is not yet using native Windows opam.</p>

<p>The Windows Server virtual machines are created with Packer. I’ve pushed my scripts to <a href="https://github.com/mtelvers/packer">mtelvers/packer</a>.</p>

<p>OCluster worker is deployed using Ansible. My scripts are at <a href="https://github.com/mtelvers/windows_worker">mtelvers/windows_worker</a>.</p>
