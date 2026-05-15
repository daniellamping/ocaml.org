---
title: Claude and Dune
description:
url: https://jon.recoil.org/blog/2025/12/claude-and-dune.html
date: 2025-12-18T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#claude-and-dune" class="anchor"></a>Claude and Dune</h1>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2025-12-18</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/odoc.html" title="odoc">odoc</a> <a href="https://jon.recoil.org/tags/ai.html" title="ai">ai</a></p></li></ul>
<p>Back in March of this year we released <a href="https://ocaml.github.io/odoc/odoc/index.html">odoc 3.0.0</a>, a major new version of the OCaml documentation generator. It had a whole load of <a href="https://discuss.ocaml.org/t/ann-odoc-3-beta-release/16043">new features</a>, many of which came with new demands on the build system driving it. We decided when working on it to build a new driver for odoc so that we could adjust it as we were building the new features, and this driver is now used to <a href="https://jon.recoil.org/07/odoc-3-live-on-ocaml-org.html" title="odoc-3-live-on-ocaml-org">build the documentation</a> that appears on <a href="https://ocaml.org/p/base/latest/doc/index.html">ocaml.org</a>. However, it was always the plan to integrate the new features into <a href="https://dune.build">Dune</a> so that everyone could just run <code>dune build @doc</code> and be able to use all of the new odoc 3 features.</p>
<p>So over the last few weeks I have been wrestling with getting Claude to update the odoc rules in Dune to support <i>some</i> of the new features of odoc v3. What began as a background experiment during a lecture series has turned into a multi-week effort to turn mostly-working code into a clean, reviewable patch. AI-developed software is clearly going to be a big part of our future, and Anil is showing us all the way with his <a href="https://anil.recoil.org/notes/aoah-2025-1">Advent of Agentic Humps</a> by building <i>new</i> software, but upstreaming AI-generated changes to an existing, well established code base <a href="https://github.com/ocaml/ocaml/pull/14369">hasn't got off to a good start</a> in the OCaml community, so I wanted to be extra careful to get this right.</p>
<h3><a href="https://jon.recoil.org/atom.xml#claude-as-a-protyping-tool" class="anchor"></a>Claude as a protyping tool</h3>
<p>The initial progress was pretty amazing, despite my initial worries that the dune code-base would be <a href="https://github.com/ocaml/dune/pull/12529">too large and subtle</a> for an LLM to be able to make workable changes. In order to get going, first I had it look at several bits of example code:</p>
<p>1. <a href="https://github.com/ocaml/dune/blob/3.20.2/src/dune_rules/odoc.ml">dune_rules/odoc.ml</a> - this is the current home of the odoc rules in dune. It's local-only, meaning it only builds the docs for the current package in isolation, so no resolution of links to stdlib, other packages or libraries.</p>
<p>2. <a href="https://github.com/ocaml/dune/blob/3.20.2/src/dune_rules/odoc_new.ml">dune_rules/odoc_new.ml</a> - these are the rules for odoc v2, which allow you to build the docs for your package plus all of the dependencies. I wrote this mostly myself some time ago. It does a pretty poor job of caching, error reporting, and has none of the odoc v3 features like assets, source rendering, hierarchical docs, better errors and so on.</p>
<p>3. <a href="https://github.com/ocaml/odoc/tree/d8460cdaa2b91a03434a9a045d673703b7fabfb2/src/driver">odoc_driver</a> - this is the driver we wrote when building odoc v3. It's fully featured, but not at all incremental, and actually external to the dune codebase. It's the reference implementation that's used to build the docs that appear on <a href="https://ocaml.org/p/base/latest/doc/index.html">ocaml.org</a>.</p>
<p>Armed with these three code-bases, I asked Claude to synthesise a new incremental version of the odoc rules for dune that has some of the features of <code>odoc_driver</code>.</p>
<h3><a href="https://jon.recoil.org/atom.xml#the-working-prototype" class="anchor"></a>The working prototype</h3>
<p>Claude quickly produced a prototype that actually compiled and generated documentation. At that stage I was not interested in the quality of the generated source; I only needed to know whether Claude could navigate Dune's codebase and produce something that <b>works</b>. I let the prototype evolve incrementally, adding in new features one at a time, for example, fixing the error reporting so that you only get warned about documentation errors that you can actually fix.</p>
<p>When the lectures finished, it turned out I had something that was pretty useful to me, and had a good chance to be useful to others too. So I opened up my editor and had a look through what had been produced, at this point hoping that a little bit of polishing should be enough - after all, it <i>was</i> working!</p>
<p>It was dreadful.</p>
<p>There were long, rambling functions, code duplication, bad comments, it was unstructured, with repeated-but-slightly-different chunks all over the place. It wasn't just bad on one length scale - it was bad from the large-scale organisation of the code down to small scale baffling weirdnesses on one line. The more I looked, the more bonkers it appeared. But it did <i>work</i>! So I thought I'd get Claude to clean up its own messes.</p>
<h3><a href="https://jon.recoil.org/atom.xml#the-clean-up" class="anchor"></a>The clean-up</h3>
<p>I resolved that I would continue to let Claude do <i>all</i> of the editing, and not do <i>any</i> myself, and so thus began the more frustrating part of this adventure! I ended up giving a mix of very specific instructions: "move this code here", "factorize out this functionality", "rename this function", and sometimes more general ones: "Remove any comments that don't add anything of value", or "Think of a better way to do this". The constant was that I needed to be looking over each change that it did, because while most of them were pretty good, there were still a few, even with the very explicit instructions, where it messed up. From the very broad, where at one point it told me "I'll remove this code to create odoc files for external dependencies, as they're installed by opam", which isn't true, down to the very small - for example, it produced the following:</p>
<div><pre class="language-ocaml"><code>let lib_names = deps.Odoc_config.libraries in
if List.is_empty lib_names
then Memo.return []
else Memo.List.filter_map lib_names ~f:(fun lib_name -&gt; Lib.DB.find lib_db lib_name)</code></pre></div>
<p>where it has come up with a totally redundant check for the empty list.</p>
<p>It was at this point where it became frustrating, because although it's almost magical that Claude can do what it does in the time it does, this fact of having to keep a constant eye on it meant the the tens-of-seconds to minutes delay in between it doing something meant I ended up either twiddling my thumbs for long periods of time, or getting started on some other task and forgetting to come back to Claude, sometimes for hours!</p>
<h3><a href="https://jon.recoil.org/atom.xml#ocaml-is-not-the-problem" class="anchor"></a>OCaml is <b>not</b> the problem</h3>
<p>One part that particularly impressed, and also quite surprised me, was with its knowledge of OCaml. In particular, I had at one point two different types representing the 'target' - either a library or a package - and a 'kind' - either a module or a page. Now pages can only be associated with package targets, and modules can only be associated with libraries, but these two values were distinct, so there was a fair bit of code pattern matching invalid combinations and either throwing exceptions or picking some random value, depending on the whims of Claude's context. I bravely suggested it think of a better way to represent this, maybe using GADTs, and it did indeed come up with a pretty nice refactoring of the types:</p>
<p>Before:</p>
<div><pre class="language-ocaml"><code>type target =
  | Lib of Package.Name.t * Lib.t
  | Pkg of Package.Name.t

type artifact_kind =
  | Module of
      { visible : bool
      ; module_name : Module_name.t
      ; archive : string (* Which archive the module belongs to *)
      }
  | Page of
      { name : string
      ; pkg_libs : Lib.t list
      }</code></pre></div>
<p>After:</p>
<div><pre class="language-ocaml"><code>(* Artifact data types *)
type page = { name : string; pkg_libs : Lib.t list }

type mod_ =
  { visible : bool
  ; module_name : Module_name.t
  ; archive : string (* Which archive the module belongs to *)
  }

type _ target =
  | Lib : Package.Name.t * Lib.t -&gt; mod_ target
  | Pkg : Package.Name.t -&gt; page target

type artifact_kind =
  | Module : mod_ * mod_ target -&gt; artifact_kind
  | Page : page * page target -&gt; artifact_kind</code></pre></div>
<p>This refactoring immediately removed a whole swathe of invalid combinations, making the code both safer and clearer. It's quite clear that Claude had no trouble understanding how GADTs work in OCaml, quite happily also using some existentials to pack them into lists and so on.</p>
<h3><a href="https://jon.recoil.org/atom.xml#odd-behaviours" class="anchor"></a>Odd behaviours</h3>
<p>Sometimes Claude just went a little bit bananas. One annoyance that <i>repeatedly</i> occurred was that it would forget how to build and test the dune executable, despite clear instructions in <code>Claude.md</code>. Most of the time when it went wrong it would build dune, execute <code>dune clean</code>, then try to run the dune binary that it had just removed with the <code>clean</code>. Sometimes it would decide to use the bootstrap binary instead, which isn't rebuilt on every change, sometimes it would run the switch-installed dune binary, and on one occasion it tried to run <code>./configure &amp;&amp; make</code>!</p>
<p>It would usually figure out eventually what the right thing to do was, but when you're waiting for it to complete so you can check what it's done these sorts of delays got a bit frustrating.</p>
<h3><a href="https://jon.recoil.org/atom.xml#reflections" class="anchor"></a>Reflections</h3>
<p>At one point, I ran out of Claude credits (despite paying $100 a month or so), at about 6:20pm one evening, and it told me that I needed to wait until 7pm to carry on. I'd just got to the point when I needed to write a short bit of code rather than refactoring what was already there, and I realised that while it would take me maybe 10 mins, it would take Claude maybe 10 seconds. Now, it could just be that it was the end of a long day and I was running out of steam, but I was content to switch focus elsewhere for a bit to wait for my credits to reset before carrying on! The point being that for the small implementation that I was after, it would be possible for me to get Claude to do it, and to eyeball the result to make sure it was OK in less time than I would have been able to do it myself. But I absolutely wouldn't have trusted Claude to do it in an upstreamable way <b>without</b> looking at the result.</p>
<p>Overall, It's clear that Claude will be an incredibly useful tool for working with software. It's unbelievably good at jumping into a new code-base and figuring things out quickly, but less good at producing high-quality code that can be directly submitted upstream (yet?) - at least, not that <b>I</b> would be comfortable submitting anyway. However, I think it's still a bit of an open question as to what the quality bar <em>should</em> be. If it builds correctly, passes the tests, looks <i>broadly</i> sensible and isn't on the critical path for performance, how much should we care about the line-to-line quality? <b>I</b> certainly care, but am I being old fashioned?</p>
<p>I've submitted a <a href="https://github.com/ocaml/dune/pull/12995">PR with these changes</a> for review, and we'll see what happens there. I ended up squashing all of the commits into one, as the intermediate steps are very likely not useful. However, for historical interest, the branch on which I did most of the work is <a href="https://github.com/ocaml/dune/compare/main...jonludlam:dune:odoc3-global-sidebar">here</a>.</p>
