---
title: Forging compilers in opam
description: "As we settle into 2026, I have been doing a little early spring-cleaning.
  A few years ago, we had a slightly chaotic time in opam-repository over what should
  have been a migration from gforge.inria.fr to a new GitLab instance. Unfortunately,
  some release archives effectively disappeared from official locations, and although
  the content was available elsewhere, the precise archives weren\u2019t generally
  available, which is a problem for the checksums in opam files. We\u2019ve had similar
  problems with GitHub in the past. As a \u2018temporary solution\u2019, @avsm created
  ocaml/opam-source-archives to house copies of these archives (I think it\u2019s
  a somewhat prescient sha for that first commit!). As is so often the case with temporary
  solutions, it\u2019s grown somewhat. Rather against my personal better judgement,
  the repo got used to house files which used to be shipped as part of ocaml/opam-repository.
  Removing the files from the repository was a good change, because they were always
  being shipped as part of opam update, but unfortunately moving them to an \u201Carchive\u201D
  repository has made it rather too tempting to add new files, making an archive repository
  a primary source."
url: https://www.dra27.uk/blog/platform/2026/01/11/ocaml-config.html
date: 2026-01-11T00:00:00-00:00
preview_image:
authors:
- ""
source:
ignore:
---

<p>As we settle into 2026, I have been doing a little early spring-cleaning. A few
years ago, we had a slightly chaotic time in opam-repository over what should
have been a migration from gforge.inria.fr to a new <a href="https://gitlab.inria.fr/">GitLab instance</a>.
Unfortunately, some release archives effectively disappeared from official
locations, and although the content was available elsewhere, the precise
archives weren’t generally available, which is a problem for the checksums in
opam files. We’ve had <a href="https://github.com/ocaml/opam-repository/pull/23849">similar problems</a>
<a href="https://github.com/ocaml/opam-repository/pull/10182">with GitHub</a> in the past.
As a ‘temporary solution’, <a href="https://github.com/avsm">@avsm</a> created <a href="https://github.com/ocaml/opam-source-archives/commit/bad1a54d3e64267a3e5b5e9e13083d55cc176307">ocaml/opam-source-archives</a>
to house copies of these archives (I think it’s a somewhat prescient sha for
that first commit!). As is so often the case with temporary solutions, it’s
grown somewhat. Rather against my personal better judgement, the repo got used
to house files which used to be shipped as part of <a href="https://github.com/ocaml/opam-repository">ocaml/opam-repository</a>.
Removing the files from the repository was a good change, because they were
always being shipped as part of <code class="language-plaintext highlighter-rouge">opam update</code>, but unfortunately moving them to
an “archive” repository has made it rather too tempting to add <em>new</em> files,
making an archive repository a primary source.</p>

<p>Back in September, as part of Relocatable OCaml, I needed to update the
ocaml-config package, which houses one of the plumbing scripts used for the
<a href="https://opam.ocaml.org/packages/opam"><code class="language-plaintext highlighter-rouge">ocaml</code></a> package in opam. Separately, for
opam’s CI systems, we wanted to be able to test against trunk OCaml, which
implied <a href="https://github.com/ocaml/opam-repository/issues/23515">some updates to the plumbing</a>
for <a href="https://opam.ocaml.org/packages/ocaml-system"><code class="language-plaintext highlighter-rouge">ocaml-system</code></a> as well. The
right thing to do with these scripts which had lived in opam-respository is to
push them back upstream, which was what I did with a cute piece of Git
spelunking in <a href="https://github.com/ocaml/ocaml/pull/14351/commits">ocaml/ocaml#14351</a>,
which contains commits with files cherry-picked from <a href="https://github.com/ocaml/opam-repository">ocaml/opam-repository</a>
PRs. Each commit contains a reference to an opam-repository commit, which in
turn leads to the original PR. For example, <a href="https://github.com/ocaml/ocaml/commit/d0272f845e90f8280e821f90382b3c189af00ea6">ocaml/ocaml#d0272f8</a>
copies the original files from <a href="https://github.com/ocaml/opam-repository/commit/1bab4537a2f3b2328d771e1bb50b0d0268b0798a">ocaml/opam-repository#1bab453</a>.
The neat thing with putting them into a series under a single path in OCaml now
is what then happens with subsequent changes. For example, <a href="https://github.com/ocaml/opam-repository/pull/17541">ocaml/opam-repository#17541</a>
introduced the <code class="language-plaintext highlighter-rouge">ocaml-config.2</code> package, which was a completely fresh script,
but <a href="https://github.com/ocaml/ocaml/commit/749a918bb85033b0a6370b85a7c6a4be33620c58">ocaml/ocaml#749a918</a>
is instead able to show the actual diff of the script, allowing for much easier
review and so forth. Of course, git doesn’t store patches, so the really useful
part is that although the history is different, the <em>file</em> in each commit is
exactly as in the original commit, which allowed <a href="https://github.com/ocaml/opam-repository/pull/29080">ocaml/opam-repository#29080</a>
just to update the URLs to point to these <em>authoritative</em> upstream sources.</p>

<p>So far, so good - that had all been merged before Christmas. The support for
explicit-relative paths in <code class="language-plaintext highlighter-rouge">ld.conf</code> added in <a href="https://github.com/ocaml/ocaml/pull/14243">ocaml/ocaml#14243</a>
required an update to this script, as noted in <a href="https://github.com/ocaml/opam-repository/pull/29085">ocaml/opam-repository#29085</a>,
since <code class="language-plaintext highlighter-rouge">opam var ocaml:stubsdir</code> for OCaml 5.5 and onwards was giving an
erroneous <code class="language-plaintext highlighter-rouge">../stublibs:./stublibs:.</code> as it didn’t know to translate the paths
read from <code class="language-plaintext highlighter-rouge">ld.conf</code> as being relative to <code class="language-plaintext highlighter-rouge">ld.conf</code> itself. That an update was
required anyway gave me opportunity to fix three other oddities in that script:</p>

<ol>
  <li>The script gets installed to users’ opam switches, rather than used directly
as part of the build.</li>
  <li>There were several versions of it, and there didn’t need to be.</li>
  <li>The use of opam’s <code class="language-plaintext highlighter-rouge">substs</code> mechanism made it difficult to debug outside of
opam.</li>
</ol>

<p>This work had all been left in <a href="https://github.com/ocaml/ocaml/pull/14250">ocaml/ocaml#14250</a>
<a href="https://github.com/ocaml/ocaml/commits/78ca20415620060c279ed74a0ea5b22c06d7e548">in December</a>
and was straightforward enough. The ocaml-config package had only come into
existence in <a href="https://github.com/ocaml/opam-repository/pull/11928">ocaml/opam-repository#11928</a>
in order to stop the same script being stored multiple times in the same
repository. Once <a href="https://github.com/ocaml/opam-repository/pull/25960">ocaml/opam-repository#25960</a>
reduced that single copy to zero copies, it made more sense for each ocaml
package just to download the script download, and do away with the ocaml-config
package completely. Unifying the scripts, and turning it into a regular OCaml
script flows logically from that. Although, the final lines amuse me somewhat:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">oc</span> <span class="o">=</span> <span class="n">open_out</span> <span class="n">package_config_file</span> <span class="k">in</span>
  <span class="c">(* Quoted strings need OCaml 4.02; "\ " needs OCaml 3.09! *)</span>
  <span class="nn">Printf</span><span class="p">.</span><span class="n">fprintf</span> <span class="n">oc</span> <span class="s2">"\
    opam-version: </span><span class="se">\"</span><span class="s2">2.0</span><span class="se">\"\n</span><span class="s2">\
    variables {</span><span class="se">\n</span><span class="s2">  \
</span></code></pre></div></div>

<p>This week, I updated that commit series to do the same kind of thing to the
ocaml-system script. These are now just regular OCaml scripts (in <a href="https://github.com/ocaml/ocaml/tree/trunk/tools/opam">tools/opam</a>
in <a href="https://github.com/ocaml/ocaml">ocaml/ocaml</a>) which can be run directly and
that allowed me to update <a href="https://github.com/ocaml/ocaml/pull/14354">ocaml/ocaml#14354</a>
which should allow us to be offer an ocaml-system package <em>during</em> development
and release cycles. The autoconf stuff might get revisited in that: cute, but
potentially annoying (the joy of higher order meta-programming… autoconf
generates and updates those files as part of the process of generating
<code class="language-plaintext highlighter-rouge">configure</code>, rather than as part of running <code class="language-plaintext highlighter-rouge">configure</code> itself… what could
possibly go wrong!). There’s also <a href="https://github.com/ocaml/ocaml/pull/14355">ocaml/ocaml#14355</a>
which allows a similar trick to be done with custom compilers. The point here is
that you build a non-standard compiler and install it outside of opam and this
then provides a relatively simple mechanism for opam to be able to use it.
Although “system” compilers are a bit awkward to use from distributions,
because opam interacts very badly with system-installed OCaml packages, if the
<em>only</em> thing you’re installing at a system-level is the compiler, the support is
very good.</p>

<p>The final thing to deal with was a very silly ToDo item I’d added:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>- [ ] `tools/opam/Layout.md`
</code></pre></div></div>

<p>In the excitement of getting the other Relocatable OCaml PRs opened in
September, in a brief from sanity, I had decided some kind of documentation
ought to be in place before <a href="https://github.com/ocaml/ocaml/pull/14250">ocaml/ocaml#14250</a>
was considered for merging 😱 Days turned into weeks, weeks turned into months.
And then, one not-so-very special day, I went to my laptop, I sat down, and I
wrote our compiler packaging story. A story about package names, a story about
compiler variants, a story about ancient libraries, long forgotten. But above
all things, a story about OCaml.</p>

<p>Which is mercifully now merged.</p>
