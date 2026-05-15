---
title: Week 23
description:
url: https://jon.recoil.org/blog/2025/06/week23.html
date: 2025-06-09T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#week-23" class="anchor"></a>Week 23</h1>
<ul class="at-tags"><li class="x-ocaml.requires"><span class="at-tag">x-ocaml.requires</span> <p>opam-format,fpath,rresult,bos</p></li></ul>
<ul class="at-tags"><li class="merlinonly"><span class="at-tag">merlinonly</span> </li></ul>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2025-06-09</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/weeknotes.html" title="weeknotes">weeknotes</a> <a href="https://jon.recoil.org/tags/odoc.html" title="odoc">odoc</a> <a href="https://jon.recoil.org/tags/docs-ci.html" title="docs-ci">docs-ci</a></p></li></ul>
<p>Some brief notes on last week.</p>
<h2><a href="https://jon.recoil.org/atom.xml#docs-ci-and-sherlodoc" class="anchor"></a>Docs CI and Sherlodoc</h2>
<p>Anil has been working on an <a href="https://tangled.sh/@anil.recoil.org/odoc-mcp">MCP server</a> that searches through the output of the docs CI to find relevant packages and API information for opam packages. For expediency, this works by scraping the HTML output, but a potentially better solution would be to integrate properly with <a href="https://doc.sherlocode.com">Sherlodoc</a>, <a href="https://github.com/art-w/">Arthur's</a> code search engine.</p>
<h3><a href="https://jon.recoil.org/atom.xml#building-the-index" class="anchor"></a>Building the index</h3>
<p>To make this work with the new docs CI, first we need to build a sherlodoc search database. This involves grabbing all of the <code>.odocl</code> files that odoc produces for the latest version of each library, copying them locally and running <code>sherlodoc index</code> on the output. Getting <em>all</em> of the odocl files is simple, but filtering so we only have the latest version is slightly more complex, as we need to use <code>opam</code>'s library to make sure we're looking at the latest versions.</p>
<p>The simple way to get the odocl files ends up unpacking them into the filesystem in a directory hierarchy that matches the URL on ocaml.org, so we see files like:</p>
<pre>p/odoc/3.0.0/doc/odoc.document/odoc_document.odocl</pre>
<p>So finding the version number is a matter of listing the directories, for which I took <a href="https://github.com/ocurrent/ocaml-docs-ci/blob/4dfe7e6265610da4e0ce2a386cfbf0b8eac3d9bd/src/lib/track.ml#L58-L76">some code</a> from docs CI:</p>
<div><pre class="language-ocaml"><code>type p = {
  opam : OpamPackage.t;
  path : Fpath.t;
}

let rec take n l =
  match n, l with
  | n, x::xs when n &gt; 0 -&gt;
      x :: take (n - 1) xs
  | _, _ -&gt; []

let get_versions ~limit path =
    let open Rresult in
    let package = Fpath.basename path in
    let mk_pkg v =
      Printf.sprintf "%s.%s" package v
    in
    Bos.OS.Dir.contents path
    &gt;&gt;| (fun versions -&gt;
    versions
    |&gt; List.map (fun path -&gt;
            { opam = Fpath.basename path |&gt; mk_pkg |&gt; OpamPackage.of_string;
              path = path })
            )
    |&gt; Result.get_ok
    |&gt; (fun v -&gt;
           v
           |&gt; List.sort (fun a b -&gt;
                  -OpamPackage.compare a.opam b.opam)
           |&gt; take limit)</code></pre></div>
<p>This gives us a sorted list of the versions for the package, and we can pick the first one to get the latest version. With the output from this we can then run <code>sherlodoc index</code> and we get a nice big (1.7 gig!) index file.</p>
<h3><a href="https://jon.recoil.org/atom.xml#serving-the-index" class="anchor"></a>Serving the index</h3>
<p>The next step is to serve this index file so that the MCP server can access it. The file format is a marshalled OCaml value, so we need to have an OCaml program to read it in and perform the search, and it'll have to be a server since the whole index needs to be unmarshalled into memory before any search can be performed, and it would be dumb to do this for every query.</p>
<p>Sherlodoc got partially integrated into odoc's code base before the 3.0 release with the exception of the server, which was left out to avoid pulling a load of new dependencies to odoc. Unfortunately, we didn't expose the sherlodoc libraries publicly, so we'll need to do that in order to make anything useful with sherlodoc. In addition, odoc embeds the version of odoc used into the odocl files, and since right now the docs CI is building with a version of odoc that <em>doesn't</em> expose the libraries, we might have to hack around that in order to use those odocl files. Obviously the longer term solution is just to make a new release of odoc with this change and update the docs CI to use that.</p>
<h2><a href="https://jon.recoil.org/atom.xml#package-to-library-map" class="anchor"></a>Package to Library map</h2>
<p>A related quest was to assemble a map of opam package to ocamlfind library names. It's a quirk of history that the library names that an opam package provides are not necessarily related to the name of the package. That means that tools like <code>dune</code> have a hard time linting projects to check that the libraries they're using are mentioned in the opam files. Dune, of course, has resolved this be ensuring that it's an error to build a package using dune where the library names don't start with the package name, but as dune is just one of many OCaml build systems, the problem remains.</p>
<p>Since docs CI has built every version of every package, and because the Odoc 3 package layout includes the library names in the paths to the documentation, we should be able to produce a fairly definitive list of the libraries that each package provides, which tools like dune can then use for this sort of lint check. We can just tweak the code above slightly to get the library names and output a big JSON file with the mapping - or perhaps with the exceptions to dune's rule.</p>
<p>I thought this would be a neat first project to try Claude Code on - a 'starter for ten' - as it were, so I signed up to use Claude code and gave it a shot.</p>
<p>It handily produced a working program that did exactly what I wanted, including creating a test directory that it used to verify the code worked. One fascinating thing I noted as it scrolled past was that it tried to use <code>yojson</code> to write the output, but failed to get it to work and reverted back to <code>printf</code> output. I suspect this will be due to it finding it troublesome to figure out the various steps that need to be taken to use a new library in a dune project, so this is something to have a play with later.</p>
<p>After a couple of iterations with different heuristics to disambiguate between library names and other directories, I got a working program producing a JSON file with only the exceptions to dune's rule. I took a look through and almost immediately found <code>camlidl</code> suggesting it produces a library called <code>com</code>. This didn't look right at all so I installed it and found that the library is actually named <code>camlidl</code>. The <code>cma</code> file, though, is named <code>com.cma</code>, so it looks like <code>odoc_driver</code> has a bug. Interestingly, running <code>odoc_driver</code> locally gets the library name correct, so it's only an issue when running it in the docs CI. <a href="https://github.com/ocaml/odoc/issues/1351">Issue filed</a>.</p>
<h2><a href="https://jon.recoil.org/atom.xml#further-claude-code-experiments" class="anchor"></a>Further claude code experiments</h2>
<p>To see how well Claude Code could handle more complex tasks, I thought I'd give it a whirl on something more like its home territory, and somewhere where I was less familiar. I decided to ask it to write some code to make a nicer editor experience for the notebooks project. Since the <a href="https://github.com/jonludlam/jsoo-code-mirror">bindings to codemirror</a> I'm using are very minimal, any new features I want to use end up with needing to write a bunch of bindings first, and only then seeing if the feature works as I'd like. So instead I thought I'd get claude to write the editor code for me in javascript, and then I could make sure it works as I want and only then convert it to OCaml. This worked pretty nicely, and I've now got a neat <a href="https://jon.ludl.am/experiments/notebook-editor/notebook-editable.html">demonstration editor</a> that I can use to guide the OCaml implementation.</p>
<h2><a href="https://jon.recoil.org/atom.xml#more-notebook-work" class="anchor"></a>More notebook work</h2>
<p>The oxcaml project will be launching this week hopefully. I've been looking at Luke's <a href="https://github.com/ocaml-flambda/flambda-backend/pull/3886">Parallelism tutorial</a> and have been thinking about how this will work with the online notebooks. The parallel library works via effects, and the oxcaml branch of js_of_ocaml doesn't support effects yet, and it might be a while before it does. However, the blog post is mainly talking about the intricacies of the type system work that's been done to ensure the parallel library is safe, and as such perhaps we can get a lot out of doing this online with just Merlin.</p>
<p>Some early experimentation showed that the parallel library breaks the worker on load, so we need to do something a bit more sophisticated than just 'not call exec', so I did some work to have a mode of worker that doesn't load the cmas, just the cmis for Merlin. This is almost there.</p>
<h2><a href="https://jon.recoil.org/atom.xml#odoc-work" class="anchor"></a>Odoc work</h2>
<p>Ocaml 5.4 is just around the corner, and there's some odoc work to be done before the release. One of the main new features that will impact odoc is the new <a href="https://tyconmismatch.com/papers/ml2024_labeled_tuples.pdf">labelled tuples</a> feature. Fortunately <a href="https://github.com/lukemaurer">Luke Maurer</a> has already done a lot of work to plumb this into odoc, so this will save us a lot of work - thanks, Luke! There's likely to be a few other bits and pieces to do, but hopefully not too much.</p>
