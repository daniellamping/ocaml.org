---
title: Devcontainer for using O(x)Caml and Claude in your projects
description: A prebuilt Docker devcontainer for sandboxed OCaml and OxCaml development
  with Claude Code, including multiarch builds and network isolation.
url: https://anil.recoil.org/notes/ocaml-claude-dev
date: 2026-01-08T00:00:00-00:00
preview_image: https://anil.recoil.org/images/faces/avsm.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>I got a few questions about the dev setup I used for my <a href="https://anil.recoil.org/notes/aoah-2025">AoAH</a> sprint last month.
I've cleaned this up and published
<strong><a href="https://github.com/avsm/claude-ocaml-devcontainer">claude-ocaml-devcontainer</a></strong>,
a <a href="https://devcontainers.io">Devcontainer</a> of everything you need to do OCaml
or OxCaml development in Claude Code in a sandboxed Docker container. This means you
can (reasonably safely) run it in unattended mode with permissions bypass enabled.</p>
<h2><a href="https://anil.recoil.org/news.xml#using-the-ocamlclaude-devcontainers" class="anchor" aria-hidden="true"></a>Using the OCaml/Claude devcontainers</h2>
<p>A devcontainer can either be used in an editor that supports it like VSCode, or directly from the CLI (which is what I do for Claude Code). Adding it to your project is as simple as:</p>
<pre><code>$ mkdir .devcontainer
$ cd .devcontainer
$ curl -OL https://raw.githubusercontent.com/avsm/claude-ocaml-devcontainer/refs/heads/main/.devcontainer/devcontainer.json
</code></pre>
<p>Edit the JSON file to add any other post-installation or extensions that you might need for that project. It's intended to be customisable.</p>
<p>To spin the devcontainer up, I use the npx CLI:</p>
<pre><code>$ npx @devcontainers/cli up   --workspace-folder .
$ npx @devcontainers/cli exec --workspace-folder . bash -l
</code></pre>
<p>This will mount your current project into <code>/workspace</code> in the dev container. The set of network domains that can be accessed are limited to a <a href="https://github.com/avsm/claude-ocaml-devcontainer/blob/main/.devcontainer/init-firewall.sh#L68-L83">select few</a>, but I'll parameterise this into the project metadata in the future.</p>
<p>When you're in the workspace container, you have two preinstalled OCaml switches:</p>
<pre><code class="language-bash">$ opam switch
#  switch    compiler                 description
   5.2.0+ox  ocaml-variants.5.2.0+ox  5.2.0+ox
→  default   ocaml.5.4.0              default
</code></pre>
<p>And <code>claude</code> maps its config from your home directory, so you can start up sessions once you've authenticated as normal, and everything runs in the container reasonably sandboxed.</p>
<h2><a href="https://anil.recoil.org/news.xml#customising-the-devcontainers" class="anchor" aria-hidden="true"></a>Customising the devcontainers</h2>
<p>Since compiling up O[x]Caml can take a while, it's a prebuilt image, but you can also clone the <a href="https://github.com/avsm/claude-ocaml-devcontainer">repository</a> and customise the Dockerfile in <code>.devcontainer</code> to your heart's content.
The default disk space in a GitHub Action wasn't sufficient, but it turns out that the default <code>ubuntu-latest</code> has a ton of pre-installed packages <a href="https://carlosbecker.com/posts/github-actions-disk-space/">that you can just delete to double your disk space</a>.</p>
<p>One pretty cool thing about the action is that it doesn't use qemu to build the
multiarch images. Instead, the
<a href="https://github.com/avsm/claude-ocaml-devcontainer/blob/main/.github/workflows/multi-build.yaml">multibuild.yml</a>
dispatches separate builds to the native arm64 and amd64 hosts, and then
combines them together. This is faster and more reliable than the conventional
path of going through CPU emulation for the non-native host.</p>
<p>I also modify my own devcontainer to mount some limited SSH keys and a
<code>.gitconfig</code> so I can commit from within the container, which allows for more
unattended feedback loops.</p>
