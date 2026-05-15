---
title: 'Opam 104: Sharing Your Code'
description: 'Curious about the origins of opam?

  Check out this short history on its evolution as the de facto package manager and
  environment manager for OCaml. Welcome back to the opam deep-dives series! In this
  article, we cover two essential topics for any OCaml developer: Setting up a development
  environment...'
url: https://ocamlpro.com/blog/2026_01_08_opam_104_sharing_your_code
date: 2026-01-08T08:16:41-00:00
preview_image: https://ocamlpro.com/blog/assets/img/dalle_camel_on_paperplane.png
authors:
- "\n    Raja Boujbel\n  "
source:
ignore:
---

<p></p>
<p>
</p><div class="figure">
  <p>
    <a href="https://ocamlpro.com/blog/assets/img/dalle_camel_on_paperplane.png">
      <img src="https://ocamlpro.com/blog/assets/img/dalle_camel_on_paperplane.png" alt="A camel riding a paper plane into the sky, like for every other OCaml package, all it took was a push to the opam-repository.">
    </a>
    </p><div class="caption">
      A camel riding a paper plane into the sky, like for every other OCaml package, all it took was a push to the opam-repository.
    </div>
  <p></p>
</div>
<p></p>
<blockquote>
<p><strong>Curious about the origins of opam?</strong></p>
<p>Check out this <a href="https://opam.ocaml.org/about.html#A-little-bit-of-History">short
history</a> on its
evolution as the de facto package manager and environment manager for OCaml.</p>
</blockquote>
<h3>Welcome back to the <code>opam deep-dives</code> series!</h3>
<p>In this article, we cover two essential topics for any OCaml developer:</p>
<ol>
<li><strong>Setting up a development environment</strong> — how to quickly bootstrap a
working environment for an existing OCaml project, including test
dependencies, documentation tools, and IDE integration.
</li>
<li><strong>Releasing your package</strong> — how to publish your OCaml library or tool to
the official <code>opam-repository</code>, making it available to the entire community.
</li>
</ol>
<p>To get the most out of this article, you should be familiar with <code>opam</code>
fundamentals: what switches are, how to install packages, and how to write a
basic <code>.opam</code> file for your project.</p>
<p>If you're new to <code>opam</code> or need a refresher, check out our previous posts in
this series:</p>
<ul>
<li><a href="https://ocamlpro.com/blog/2024_01_23_opam_101_the_first_steps">Opam 101</a> — Getting started with <code>opam</code>
</li>
<li><a href="https://ocamlpro.com/blog/2024_03_25_opam_102_pinning_packages">Opam 102</a> — Pinning packages
</li>
<li><a href="https://ocamlpro.com/blog/2025_04_29_opam_103_starting_new_project">Opam 103</a> — Starting a new project and writing an <code>opam</code> file
</li>
</ul>
<p></p><div>
<strong>Table of contents</strong><p></p>
<ul>
<li><a href="https://ocamlpro.com/blog/feed#settingupadevenv">Setting up a development environment</a>
<ul>
<li><a href="https://ocamlpro.com/blog/feed#depsonly"><code>opam install</code> options and variables</a>
<ul>
<li><a href="https://ocamlpro.com/blog/feed#variables">Opam variables</a>
</li>
</ul>
</li>
<li><a href="https://ocamlpro.com/blog/feed#lockdependencies">Lock your dependencies</a>
<ul>
<li><a href="https://ocamlpro.com/blog/feed#opamlocked">Locking and pinning</a>
</li>
</ul>
</li>
</ul>
</li>
<li><a href="https://ocamlpro.com/blog/feed#releasingyourpackage">Releasing your package</a>
<ul>
<li><a href="https://ocamlpro.com/blog/feed#releasing">Releasing with <code>opam-publish</code></a>
</li>
<li><a href="https://ocamlpro.com/blog/feed#manualpublishing">Manual publishing</a>
</li>
</ul>
</li>
<li><a href="https://ocamlpro.com/blog/feed#wrappingup">Wrapping up the opam10x series</a>
</li></ul></div>


<hr>
<h2>
<a class="anchor"></a><a href="https://ocamlpro.com/blog/feed#settingupadevenv" class="anchor-link">Setting up a development environment</a>
          </h2>
<p>Last time, we got to the point where your project was set for a steady
development process. Now, we look into how you can ease other developers into
joining your development efforts.</p>
<p>Naturally, the first <code>opam</code> command a newcomer developer will use to
participate in your project is <code>opam install</code>. This is because their first goal
would be to reproduce the environment that you have set up for your project.</p>
<p>The default behaviour for a call to <code>opam install</code> is to install all necessary
dependencies for your project's binary to <strong>run</strong>. You can see this as <code>opam</code>
choosing to tailor the default usage to non-developer users.</p>
<p>However, of course, <code>opam</code> is a developer's best friend too and, for that
matter, provides a complete set of features, variables and command-line options
to help developers accomplish their goals. In the case of <code>opam install</code>, we
need to cover the following options, which all make a developer's life much
easier:</p>
<ul>
<li><code>--deps-only</code>, for minimal requirements;
</li>
<li><code>--with-test</code>, for testing requirements;
</li>
<li><code>--with-doc</code>, for building documentation;
</li>
<li><code>--with-dev-setup</code>, for additional IDE tooling and extended QoL requirements.
</li>
</ul>
<h3>
<a class="anchor"></a><a href="https://ocamlpro.com/blog/feed#depsonly" class="anchor-link"><code>opam install</code> options and variables</a>
          </h3>
<p>Keep in mind that the following options can be used together, and that there
exist other <code>opam</code> features which utilise them too.</p>
<h5><code>--deps-only</code></h5>
<p>This option is crucial when setting up your development environment <strong>without
installing or rebuilding your project</strong>. It installs only the dependencies
listed in your <code>*.opam</code> file.</p>
<p>The recommended approach is to create a <strong>local switch</strong> for your project. This
keeps your development environment isolated and project-specific:</p>
<pre><code class="language-shell-session">$ opam switch create . --deps-only
</code></pre>
<p>This creates a new switch in the current directory and installs the <strong>minimal
set of dependencies</strong> needed to build the project. Local switches are stored in
a <code>_opam/</code> directory within your project and are automatically selected when
you enter the project folder.</p>
<p>All the options described below (<code>--with-test</code>, <code>--with-doc</code>, <code>--with-dev-setup</code>)
can be combined with <code>opam switch create</code>:</p>
<pre><code class="language-shell-session">$ opam switch create . --deps-only --with-test --with-doc --with-dev-setup
</code></pre>
<blockquote>
<p>Note: If you already have a switch and want to install dependencies into it
without creating a new one, use <code>opam install . --deps-only</code> instead.</p>
</blockquote>
<h4>
<a class="anchor"></a><a href="https://ocamlpro.com/blog/feed#variables" class="anchor-link">Opam variables</a>
          </h4>
<p>The <code>opam</code> file can contain conditional dependencies, guarded by
variables. These variables (<code>with-test</code>, <code>with-doc</code>, <code>with-dev-setup</code>) allow
you to <strong>toggle sets of dependencies</strong> depending on your needs. For example,
test dependencies are only installed when the <code>with-test</code> variable is set.</p>
<blockquote>
<p>💡 Interesting factoid, these variables can also be used in other fields such
as <code>build</code> and <code>install</code>, check out <a href="https://opam.ocaml.org/doc/Manual.html#Package-variables">the
manual</a>.</p>
</blockquote>
<h5><code>--with-test</code></h5>
<pre><code class="language-shell-session">$ opam install . --deps-only --with-test
</code></pre>
<p>This installs both main and test dependencies. It’s essential when preparing
an environment where you or others want to run or write tests. In your <code>opam</code>
file, such dependencies are usually declared with the <code>{with-test}</code> filter.</p>
<h5><code>--with-doc</code></h5>
<pre><code class="language-shell-session">$ opam install . --deps-only --with-doc
</code></pre>
<p>Installs dependencies required to build documentation. Same as for
<code>--with-test</code>, such dependencies are declared with the <code>{with-doc}</code> filter
inside the <code>depends:</code> field. This is useful if you’re generating docs locally,
for instance with <code>odoc</code> through <code>dune build @doc</code>.</p>
<h5><code>--with-dev-setup</code></h5>
<pre><code class="language-shell-session">$ opam install . --deps-only --with-dev-setup
</code></pre>
<p>This flag brings in development tooling dependencies, like linters,
formatters (e.g., <code>ocamlformat</code>), or editor integration (e.g., <code>merlin</code>).
These dependencies aren't required to build or test the project, but they
improve the <strong>quality-of-life for developers</strong>. In your <code>opam</code> file, they
would be tagged with <code>{with-dev-setup}</code>.</p>
<h5>Quick setup: one command to rule them all</h5>
<p>For a complete development environment dedicated to your project, use this
single command:</p>
<pre><code class="language-shell-session">$ opam switch create . --deps-only --with-test --with-doc --with-dev-setup
</code></pre>
<p>This command:</p>
<ul>
<li><code>opam switch create .</code> — creates a local switch in your project directory
</li>
<li><code>--deps-only</code> — installs only dependencies, without building your project yet
</li>
<li><code>--with-test</code> — includes test frameworks (e.g., <code>alcotest</code>, <code>ppx_expect</code>)
</li>
<li><code>--with-doc</code> — includes documentation tools (e.g., <code>odoc</code>)
</li>
<li><code>--with-dev-setup</code> — includes developer tooling (e.g., <code>merlin</code>, <code>ocamlformat</code>)
</li>
</ul>
<p>You can omit any flag you don't need. For example, if you just want to run
tests without building docs, drop <code>--with-doc</code>.</p>
<hr>
<p>At this stage, you should have a fresh switch ready—one to compile and run the
project you are onboarding to. However, there’s a catch: an <code>.opam</code> file might
not state which <strong>exact versions of dependencies</strong> are required by that
project. This can cause two developers on different machines to end up with
slightly different versions of the same packages — a common source of subtle
and hard-to-debug compatibility issues.</p>
<p>In practice, it’s common not to be overly specific about package versions in
the <code>.opam</code> file. The reason is that locking versions too tightly reduces the
range of compatible packages, which adds friction and makes life harder for
anyone who wants to use your package alongside other software (whether they are
developing it themselves or simply using it in another context). Instead, it’s
generally better to specify <strong>version ranges</strong>, which helps maintain flexibility
and avoids unnecessary restriction of the set of compatible packages within the
same switch.</p>
<pre><code class="language-opam">depends: [
  "ocaml" {&gt;= "4.08"}
  "dune" {&gt;= "2.8"}
  "menhir" {&gt;= "2.1"}
  "js_of_ocaml" {&gt;= "3.9"}
  "js_of_ocaml-ppx" {build &amp; &gt;= "3.9"}
  "bisect_ppx" {with-test &amp; &gt;= "2.6" &amp; dev}
  "odoc" {with-doc}
]
</code></pre>
<h3>
<a class="anchor"></a><a href="https://ocamlpro.com/blog/feed#lockdependencies" class="anchor-link">Lock your dependencies with <code>opam lock</code></a>
          </h3>
<p>There is a way however to make sure anybody who joins your project can quickly
setup a switch for themselves. Opam supports what is called <code>opam.locked</code> files.</p>
<p>You might be familiar with such configuration files in the form of
<code>package-lock.json</code> (for Javascript) or <code>Pipfile.lock</code> (for Python) which are
both much more verbose than an ordinary <code>helloer.opam.locked</code> file is.</p>
<p>To ensure <strong>everyone installs the exact same dependencies and version</strong>, you can use:</p>
<pre><code class="language-shell-session">$ opam lock &lt;package&gt;
# Or use a local path to your project's directory
$ opam lock .
</code></pre>
<p>This generates a <code>helloer.opam.locked</code> file that freezes versions of all
dependencies as <strong>currently installed in your switch</strong>. It captures the exact
versions <strong>in use</strong> (notice the hard equality <code>=</code> in version constraints).</p>
<p>The process assumes your local switch is in a good state (build and tests
succeed), then uses it to record the versions you have yourself used in
practice.</p>
<pre><code class="language-shell-session"># the local opam file
$ cat helloer.opam
opam-version: "2.0"
depends: [
  "cmdliner"
  "ocaml"
  "alcotest" {with-test}
]
build: [
 [ "dune" "build" "-p" name ]
 [ "dune" "runtest" ] {with-test}
]
install: [ "dune" "install" ]

# Current content of the switch
$ opam list
# Packages matching: installed
# Name             # Installed # Synopsis
alcotest           1.9.1       Alcotest is a lightweight and colourful test framework
astring            0.8.5       Alternative String module for OCaml
base-bigarray      base
base-domains       base
base-effects       base
base-nnp           base        Naked pointers prohibited in the OCaml heap
base-threads       base
base-unix          base
camlp-streams      5.0.1       The Stream and Genlex libraries for use with Camlp4 and Camlp5
cmdliner           2.0.0       Declarative definition of command line interfaces for OCaml
cppo               1.8.0       Code preprocessor like cpp for OCaml
crunch             4.0.0       Convert a filesystem into a static OCaml module
dune               3.20.2      Fast, portable, and opinionated build system
fmt                0.11.0      OCaml Format pretty-printer combinators
fpath              0.7.3       File system paths for OCaml
heloer             dev         pinned to version dev at file:///home/developer/helloer/
ocaml              5.3.0       The OCaml compiler (virtual package)
ocaml-config       3           OCaml Switch Configuration
ocaml-syntax-shims 1.0.0       Backport new syntax to older OCaml versions
ocaml-system       5.3.0       The OCaml compiler (system version, from outside of opam)
ocamlbuild         0.16.1      OCamlbuild is a build system with builtin rules to easily build most OCaml projects
ocamlfind          1.9.8       A library manager for OCaml
odoc               3.1.0       OCaml Documentation Generator
odoc-parser        3.1.0       Parser for ocaml documentation comments
ptime              1.2.0       POSIX time for OCaml
re                 1.14.0      RE is a regular expression library for OCaml
seq                base        Compatibility package for OCaml's standard iterator type starting from 4.07.
stdlib-shims       0.3.0       Backport some of the new stdlib features to older compiler
topkg              1.1.0       The transitory OCaml software packager
tyxml              4.6.0       A library for building correct HTML and SVG documents
uutf               1.0.4       Non-blocking streaming Unicode codec for OCaml

# Generating the lock file for 'helloer'
$ opam lock helloer

# the content of the opam lock file
$ cat helloer.opam.locked
opam-version: "2.0"
name: "helloer"
version: "dev"
depends: [
  "alcotest" {= "1.9.1" &amp; with-test}
  "astring" {= "0.8.5" &amp; with-test}
  "base-bigarray" {= "base"}
  "base-domains" {= "base"}
  "base-effects" {= "base"}
  "base-nnp" {= "base"}
  "base-threads" {= "base"}
  "base-unix" {= "base"}
  "cmdliner" {= "2.0.0"}
  "dune" {= "3.20.2" &amp; with-test}
  "fmt" {= "0.11.0" &amp; with-test}
  "ocaml" {= "5.3.0"}
  "ocaml-config" {= "3"}
  "ocaml-syntax-shims" {= "1.0.0" &amp; with-test}
  "ocaml-system" {= "5.3.0"}
  "ocamlbuild" {= "0.16.1" &amp; with-test}
  "ocamlfind" {= "1.9.8" &amp; with-test}
  "re" {= "1.14.0" &amp; with-test}
  "stdlib-shims" {= "0.3.0" &amp; with-test}
  "topkg" {= "1.1.0" &amp; with-test}
  "uutf" {= "1.0.4" &amp; with-test}
]
build: [
  ["dune" "build" "-p" name]
  ["dune" "runtest"] {with-test}
]
</code></pre>
<p>Developers who clone your repository and run:</p>
<pre><code class="language-shell-session">$ opam install . --locked
</code></pre>
<p>will get the <strong>same versions</strong> of everything.</p>
<blockquote>
<p>💡 The lock file is especially useful when working in teams or in CI. It
increases reproducibility and reduces <em>"But it works on my machine"</em> issues.</p>
</blockquote>
<p>By default, the lock file is named <code>&lt;package&gt;.opam.locked</code>. You can customize
the suffix with <code>--lock-suffix</code>, but then remember to pass the same suffix when
running <code>opam install</code>.</p>
<blockquote>
<p>⚠️  Note: This guarantees a <strong>reproducible development environment</strong>, but only
if you work on the same <code>opam-repository</code> as your peers, not a bit-for-bit
<strong>reproducible build</strong>, which is a broader topic involving build sandboxes
and source hashes... Maybe a topic for another time.</p>
</blockquote>
<h4>
<a class="anchor"></a><a href="https://ocamlpro.com/blog/feed#opamlocked" class="anchor-link">Locking and pinning</a>
          </h4>
<p>Pins deserve special attention. If your project depends on pinned packages,
<code>opam lock</code> will record them as well. <code>opam</code> will use a specific field in the
<code>opam</code> file named <code>pin-depends</code> which allows you to list the packages (and their
respective URLs) that <code>opam</code> will automatically pin when installing the main
package.</p>
<pre><code class="language-shell-session">pin-depends: [ "js_of_ocaml.dev" "git+https://github.com/ocsigen/js_of_ocaml#win-test" ]
</code></pre>
<p>If the pin is a <strong>local path</strong>, but a remote exists with a branch or hash, <code>opam</code>
will record the remote version.</p>
<pre><code class="language-shell-session">$ opam pin
cmdliner.2.0.0    git    git+file:///home/developer/cmdliner#master                        (at 0123456789c0ffee123456789abcdefedcba9876)
helloer.dev       rsync  file:///home/developer/helloer

$ opam lock .
[NOTE] Local pin git+file:///home/developer/cmdliner#master resolved to git+https://erratique.ch/repos/cmdliner#master
Generated lock files for:
  - helloer.dev: /home/developer/helloer.opam.locked
$ cat helloer.opam.locked
[...]
pin-depends: [ "cmdliner.2.0.0" "git+https://erratique.ch/repos/cmdliner#master" ]
</code></pre>
<p>If no remote is available, the lock file may still include the local pin. If
you want to keep it that way, use <code>--keep-local</code> (available since 2.4). In
this case, you may need to edit the lock file before sharing it.</p>
<pre><code class="language-shell-session">$ opam pin
cmdliner.2.0.0    git    git+file:///home/developer/cmdliner#local-branch                         (at 0123456789deadc0dee0123456789abcdefedcba)

$ opam lock helloer
[WARNING] Referenced git branch for cmdliner.2.0.0 is not available in remote: git+https://erratique.ch/repos/cmdliner, use default branch instead.
[NOTE] Local pin git+file:///home/developer/cmdliner#local-branch resolved to git+https://erratique.ch/repos/cmdliner
Generated lock files for:
  - helloer.dev: /home/developer/helloer.opam.locked
$ cat helloer.opam.locked
[...]
pin-depends: [ "cmdliner.2.0.0" "git+https://erratique.ch/repos/cmdliner" ]
$ opam lock . --keep-local
[NOTE] Dependency cmdliner.2.0.0 is pinned to local target git+file:///home/developer/cmdliner#local-branch, keeping it.
Generated lock files for:
  - helloer.dev: /home/developer/helloer.opam.locked
$ cat helloer.opam.locked
[...]
pin-depends: [ "cmdliner.2.0.0" "git+file:///home/developer/cmdliner#local-branch" ]
</code></pre>
<h2>
<a class="anchor"></a><a href="https://ocamlpro.com/blog/feed#releasingyourpackage" class="anchor-link">Releasing your package</a>
          </h2>
<p>Once your project is stable and you’re ready to release it to the OCaml
ecosystem, it’s time to publish it.</p>
<p>Publishing means submitting your package to an <code>opam</code> repository (most often
<a href="https://github.com/ocaml/opam-repository">the official OCaml one</a>). Once merged,
anyone can install your package with a simple <code>opam install &lt;your-package&gt;</code>.</p>
<h3>
<a class="anchor"></a><a href="https://ocamlpro.com/blog/feed#releasing" class="anchor-link">Releasing with <code>opam-publish</code></a>
          </h3>
<p>The easiest way to publish is via the <code>opam publish</code> plugin.</p>
<p>You can get the exhaustive list of <code>opam</code> plugins with the following call:</p>
<pre><code class="language-shell-session">$ opam list --has-flag plugin
# Packages matching: has-flag(plugin) &amp; (installed | available)
# Name               # Installed # Synopsis
[...]
opam-publish         --          A tool to ease contributions to opam repositories
[...]
</code></pre>
<p><em>Plugins are topics for another time.</em> 😉</p>
<p>If you already have <code>opam</code>, you have nothing more to do than invoke that plugin
and it will automatically be fetched and installed within your current switch.
If you have installed <code>opam publish</code> in the past, <code>opam</code> will find it no matter
your current switch.</p>
<h5>Workflow with GitHub</h5>
<ol>
<li><strong>Tag a release in your git repository</strong>, e.g., inside your <code>world-org/helloer</code>, with version <code>1.2.3</code>.
</li>
<li>Run:
</li>
</ol>
<pre><code class="language-shell-session">$ opam publish world-org/helloer --tag 1.2.3
</code></pre>
<ol start="3">
<li>The tool will:
</li>
</ol>
<ul>
<li>Validate your <code>opam</code> file (linting, formatting, metadata checks).
</li>
<li>Since the GitHub tarball URL is automatically generated when you tag a new
release, <code>opam-publish</code> uses that URL, generates a checksum for it and adds
them both to the <code>opam</code> file.
</li>
<li>The tool will clone the <code>opam repository</code>, commit your package, push a
branch, and open a pull request automatically.
</li>
</ul>
<pre><code class="language-shell-session">$ opam publish world-org/helloer --tag 1.2.3

The following will be published:
  - helloer version 1.2.3 with opam file from the upstream archive
    archive at https://github.com/world-org/helloer/archive/refs/tags/1.2.3.tar.gz

You will be shown the patch before submitting.
Please confirm the above data. Continue ?  [Y/n] y

Please generate a Github token at https://github.com/settings/tokens/new to allow access.
The "public_repo" and "workflow" scopes are required ("repo" if submitting to a private opam repository).

Please enter your GitHub personal access token:
The token will be stored in ~/.opam/plugins/opam-publish/ocaml%opam-repository.token.
Fetching the package repository, this may take a while...


commit 0123456789abcdef0123456789abcdef01234567 (HEAD -&gt; master)
Author: Welcomer &lt;hell@er.com&gt;
Date:   Tue Oct 21 11:09:41 2025 +0200

    1 package from world-org/helloer at 1.2.3

diff --git a/packages/helloer/helloer.1.2.3/opam b/packages/helloer/helloer.1.2.3/opam
new file mode 100644
index 0000000000..ea8f4010a0
--- /dev/null
+++ b/packages/helloer/helloer.1.2.3/opam
@@ -0,0 +1,48 @@
+opam-version: "2.0"
+synopsis: "HW"
+description: "A simple tool to display several types of 'hello world'"
+maintainer: ["Welcomer &lt;hell@er.com&gt;"]
+authors: ["Welcomer &lt;hell@er.com&gt;"]
+tags: ["toy project"]
+homepage: "https://github.com/OCamlPro/opam_bp_examples"
+doc: "https://url/to/documentation"
+bug-reports: "https://github.com/OCamlPro/opam_bp_examples"
+depends: [
+  "cmdliner"
+  "ocaml"
+  "alcotest" {with-test}
+]
+build: [
+ [ "dune" "build" "-p" name ]
+ [ "dune" "runtest" ] {with-test}
+ [ "dune" "build" "@doc" ] {with-doc}
+]
+install: [ "dune" "install" ]
+dev-repo: "https://github.com/OCamlPro/opam_bp_examples"
+url {
+  src: "https://github.com/world-org/helloer/archive/refs/tags/1.2.3.tar.gz"
+  checksum: [
+    "md5=0123456789abcdef0123456789abcdef"
+    "sha512=0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
+  ]
+}
 No newline at end of file
File a pull-request for this patch ? [Y/n]
</code></pre>
<blockquote>
<p>💡 <strong>About GitHub tokens:</strong> As shown above, <code>opam-publish</code> requires a GitHub
personal access token to clone, commit, and open a PR on your behalf. You only
need to enter it once—it's stored locally at
<code>~/.opam/plugins/opam-publish/ocaml%opam-repository.token</code> until expiration.
The tool prompts you with the necessary URL and scope (<code>public_repo</code> and <code>workflow</code> for
public repositories, <code>repo</code> for private ones). Once you confirm the final
prompt, the PR is automatically opened and your browser displays the PR page
for you to track its progress.</p>
</blockquote>
<p><strong>Once the PR is reviewed and merged by maintainers, your package becomes
available to everyone.</strong></p>
<h5>Without GitHub</h5>
<p>If you don’t use GitHub releases, you can provide a URL to a tarball that
contains the source and the project opam file :</p>
<pre><code class="language-shell-session">$ opam publish https://world.com/helloer-1.0.0.tar.gz
</code></pre>
<p><code>opam-publish</code> will handle the rest in a similar way.</p>
<blockquote>
<p>💡 You may also provide <code>opam-publish</code> with a URL pointing to a different
GitHub repository than the official OCaml one. Indeed, there exist other
repositories in the wild, public or private, which you may want to publish
to. You can use: <code>opam publish --repo other-org/opam-repository</code> for that.</p>
</blockquote>
<h5>What about <code>dune-release</code>?</h5>
<p>You may have heard of
<a href="https://github.com/tarides/dune-release"><code>dune-release</code></a>, another popular tool
for releasing OCaml packages. While <code>opam-publish</code> focuses solely on submitting
your package to an opam repository, <code>dune-release</code> is a more comprehensive
release workflow tool that handles the entire process:</p>
<ul>
<li>Creating and pushing git tags
</li>
<li>Generating changelogs
</li>
<li>Creating GitHub releases with release notes
</li>
<li>Building and uploading distribution tarballs
</li>
<li>Submitting to opam-repository
</li>
</ul>
<p>If you're looking for an all-in-one release automation tool, <code>dune-release</code> is
worth exploring. However, if you prefer to manage your git tags and releases
manually and just need help with the opam submission step, <code>opam-publish</code> is
simpler and does exactly that.</p>
<h3>
<a class="anchor"></a><a href="https://ocamlpro.com/blog/feed#manualpublishing" class="anchor-link">Manual publishing (if you need it)</a>
          </h3>
<p>You can also publish manually, but it’s more work:</p>
<ol>
<li>Clone the <code>opam-repository</code>:
</li>
</ol>
<pre><code class="language-shell-session">$ git clone https://github.com/ocaml/opam-repository
</code></pre>
<ol start="2">
<li>Create a new subdirectory:
</li>
</ol>
<pre><code class="language-shell-session">$ mkdir -p packages/your-package/your-package.version/
</code></pre>
<ol start="3">
<li>Add your <code>opam</code> file there (with a <code>url</code> section containing tarball + checksum).
</li>
<li>Use <code>opam lint</code> to verify that your <code>opam</code> file is valid.
</li>
</ol>
<pre><code class="language-shell-session">$ opam lint packages/your-package/your-package.version/opam
</code></pre>
<ol start="5">
<li>Commit, push and open a pull request.
</li>
</ol>
<p><strong>This works but requires you to take care of validation and formatting
yourself. That's why <code>opam-publish</code> is strongly recommended — it makes things
easier for the opam repository maintainers team.</strong></p>
<h2>
<a class="anchor"></a><a href="https://ocamlpro.com/blog/feed#wrappingup" class="anchor-link">Wrapping up the opam10x series</a>
          </h2>
<p>With these tools in hand:</p>
<ul>
<li><code>opam install . --deps-only --with-doc --with-test --with-dev-setup</code> to
quickly jump into a project;
</li>
<li><code>opam lock</code> to freeze versions of the packages of your current switch;
</li>
<li><code>opam publish</code> to share your work with the world;
</li>
</ul>
<p>…you now have the full pipeline to <strong>build, collaborate, and distribute</strong> OCaml
projects like a pro... An <strong>OCaml<em>Pro</em></strong>...</p>
<hr>
<p><strong>Thank you for tagging along this first salvo of <code>opam deep-dives</code>.</strong></p>
<p>From <a href="https://ocamlpro.com/blog/2024_01_23_opam_101_the_first_steps"><code>Opam 101</code></a> in January 2024 to
<code>Opam 104</code> today, we have covered what we believe is key to stepping into the OCaml
ecosystem organically. With these articles you have a first-hand demonstration
of the tooling, philosophy and workflow, and you should be able to get started
on the path to becoming a fully-fledged OCaml contributor!</p>
<p>Keep building, keep sharing, and keep the
<a href="https://ocaml.org">OCaml</a> <a href="https://opam.ocaml.org">ecosystem</a>
<a href="https://github.com/ocaml/opam-repository">thriving</a>! 🐫💚</p>
<hr>
<p>Thank you for reading,</p>
<p>From 2011, with love,</p>
<p><strong>The OCamlPro Team</strong>.</p>
<blockquote>
<p>This article series was made possible thanks to the support of the <a href="https://ocaml-sf.org/">OCaml
Software Foundation</a>. Thank you for helping us share
knowledge with the OCaml community!</p>
</blockquote>

