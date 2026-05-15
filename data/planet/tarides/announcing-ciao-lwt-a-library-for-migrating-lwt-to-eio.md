---
title: 'Announcing `ciao-lwt`: A Library for Migrating Lwt to Eio'
description: Announcing a new collection of tools to automate the migration from Lwt
  to Eio!
url: https://tarides.com/blog/2026-03-05-announcing-ciao-lwt-a-library-for-migrating-lwt-to-eio
date: 2026-03-05T00:00:00-00:00
preview_image: https://tarides.com/blog/images/ciao-lwt-1360w.webp
authors:
- Tarides
source:
ignore:
---

<p>The I/O library <a href="https://github.com/ocaml-multicore/eio">Eio</a>, which uses effects and direct-style concurrency, was released in 2024. Since then, users have seized the opportunity to <a href="https://tarides.com/blog/2024-09-19-eio-from-a-user-s-perspective-an-interview-with-simon-grondin/">test it in their own projects</a>, and several OCaml devs have ported applications to Eio.</p>
<p>Now, with a new library called <a href="https://github.com/tarides/ciao-lwt">ciao-lwt</a>, users can automate part of the migration process from Lwt to Eio. One of our engineers, Jules Aguillon, has been developing the library and using it for the <a href="https://ocsigen.org/home/intro.html">Ocsigen</a> project. This post shares his work and will introduce you to <code>ciao-lwt</code>, how to try it, and what limitations you should expect.</p>
<p>The project is made possible thanks to a grant from the <a href="https://nlnet.nl">NLnet Foundation</a>, which funds research and development projects furthering internet technologies and the open internet, and the <a href="https://nlnet.nl/core/">NGI Zero Core fund</a> of the European commission.</p>
<h2>Why Would I Switch to Eio?</h2>
<p>Ultimately, the concurrency library you choose comes down to a matter of taste, but Eio has some nice characteristics that you may find worth the switch. Since it is direct-style, Eio does not require you use a monad for concurrency, which gets rid of the so-called ‘<a href="https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/">function colouring problem</a>’. The resulting code is faster, less complex, and has some nice security capabilities.</p>
<p>You can read <a href="https://tarides.com/blog/2024-03-20-eio-1-0-release-introducing-a-new-effects-based-i-o-library-for-ocaml/">our blog post on Eio 1.0</a> for more context.</p>
<h2>Ciao-Lwt</h2>
<p>The library contains a collection of tools for translating an Lwt library into Eio code. Lwt marks concurrent code and non-concurrent (sync and async) code using bind operators or bindings, and functions must be explicitly marked as sync or async for the program to run. These are the ‘bind’ and ‘map’ operators, including <code>Lwt.bind</code>, <code>Lwt.map</code>, <code>let*</code>, <code>let+</code>, as well as the infix operators  <code>&gt;&gt;=</code> and <code>&gt;&gt;|</code>.</p>
<p>The first step in turning Lwt into Eio code is to get rid of the bind operators. They weave through every part of Lwt code and are time-intensive to remove one by one. <code>Ciao-lwt</code> can automate the process and remove the bindings for you.  However, the library has some limitations, which will be explained in more detail below.</p>
<p>Lastly, something to bear in mind is that <code>ciao-lwt</code> uses <a href="https://github.com/ocaml/merlin">Merlin</a>'s index to locate every use of <code>Lwt</code>.</p>
<h2>How Do I Try It?</h2>
<p>To get started with <code>ciao-lwt</code>, the first thing to do is <a href="https://github.com/tarides/ciao-lwt">visit the repo</a> and install the tools in the <code>opam</code> switch you’re using to build your projects using the command:</p>
<pre><code>opam install ciao_lwt
</code></pre>
<p>To make reviewing the change easier, make sure your code is formatted. The tool will entirely reformat the file it touches, which may make actual changes harder to see.</p>
<p>The first step is to remove any use of <code>lwt_ppx</code> (for example the <code>let%lwt</code> syntax):</p>
<pre><code>ciao-lwt lwt-ppx-to-let-syntax .
dune fmt # Remove formatting changes created by the tool
</code></pre>
<p>This operation is purely syntactical, the tool simply walks the given directory tree and parses every <code>.ml</code> files it finds, updating the files that contain usages of <code>lwt_ppx</code>.</p>
<p>Before running the next tool, try eliminating common causes of implicit forks:</p>
<pre><code>ciao-lwt lint .
</code></pre>
<p>This operation is also purely syntactical. The tool warns about every occurrence of <code>let _ = ..</code> and <code>ignore</code> that doesn’t have a type annotation. This helps you find cases where an Lwt promise is disregarded. To silence each warning, add a type annotation, for example: <code>let _ : my_t = ..</code> and <code>ignore (.. :my_t)</code>.</p>
<p>If you use <code>Lwt_log</code>, you can migrate to <code>Logs</code> easily with:</p>
<pre><code>dune build @ocaml-index # Build the index (required)
ciao-lwt to-logs --migrate .
dune fmt # Remove formatting changes created by the tool
</code></pre>
<p>This tool works similarly to <code>ciao-lwt to-eio</code> described below. It is provided as a separate command because your program will likely work as before but it lets you review this step independently and it simplifies the next step.</p>
<p>Finally, migrate to Eio:</p>
<pre><code>dune build @ocaml-index # Build the index (required)
ciao-lwt to-eio --migrate .
dune fmt # Remove formatting changes created by the tool
</code></pre>
<p>This operation migrates the common uses of Lwt, but the transition is not yet complete.</p>
<h2>Limitations &amp; Considerations</h2>
<p><code>Ciao-lwt</code> is still considered experimental and a work-in-progress, which you should bear in mind when you try it. Your feedback and input is very welcome and will help the team improve the tools.</p>
<p>It sounds obvious, but as a promise-based concurrency library, Lwt creates a lot of promises. Everything that is concurrent in Lwt is a promise; it specifies actions that will happen at a later time. Some promises are so-called ‘implicit forks’, which do not use the bindings we mentioned earlier.</p>
<p>Let's look at an implicit fork:</p>
<pre><code>let _ =
  let a = operation_1 () in
  let* b = operation_2 () in
  let* a = a in
  Lwt.return (a + b)
</code></pre>
<p>Here, <code>let a = operation_1 () in</code> 'forks', meaning it creates a concurrent thread. Since there are no binding operators or <code>Lwt</code> function calls, <code>ciao-lwt</code> can't detect this fork syntactically or with Merlin's index.</p>
<p>As a result, while Lwt would run <code>operation_1</code> and <code>operation_2</code> concurrently, after <code>ciao-lwt</code> converts it to Eio it would instead run sequentially:</p>
<pre><code>let _ =
  let a = operation_1 () in
  let b = operation_2 () in
  let a = Promise.await a in
  a + b
</code></pre>
<p>The added call to <code>Promise.await</code> generates a typing error, helping to locate the issue.
Users need to be aware of how <code>ciao-lwt</code> handles 'implicit forks' so they can fix bugs introduced in the migration.</p>
<p>The most helpful tool to verify your new code is the OCaml compiler. Your resulting code will likely not typecheck, and OCaml's typechecker can guide you towards the manual changes you will need to make. It's not foolproof, and you will still need to be on the lookout for concurrency bugs, but <code>ciao-lwt</code>'s tools in combination with the OCaml compiler will give you a nice head start on your journey from Lwt to Eio.</p>
<h2>Until Next Time</h2>
<p>Tarides remains committed to creating new tools that make new and old workflows easier. We hope <code>ciao-lwt</code> proves useful to you, and appreciate any feedback you have to share.</p>
<p>You can connect with us on <a href="https://bsky.app/profile/tarides.com">Bluesky</a>, <a href="https://mastodon.social/@tarides">Mastodon</a>, <a href="https://www.threads.net/@taridesltd">Threads</a>, and <a href="https://www.linkedin.com/company/tarides">LinkedIn</a> or sign up for our mailing list to stay updated on our latest projects. We look forward to hearing from you!</p>
<h2>Updates</h2>
<p>2026-04-28 The blog post was changed to reflect the changes made in version <code>0.2</code>.</p>

