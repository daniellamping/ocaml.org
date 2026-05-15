---
title: Announcing New Wasm_of_ocaml Optimisations
description: Discover the latest optimisations we have brought to Wasm_of_ocaml in
  2025!
url: https://tarides.com/blog/2026-02-11-announcing-new-wasm-of-ocaml-optimisations
date: 2026-02-11T00:00:00-00:00
preview_image: https://tarides.com/blog/images/up-arrows-1360w.webp
authors:
- Tarides
source:
ignore:
---

<p>2025 was a good year for WASM support in OCaml! In February 2025, we announced the full release of Wasm_of_ocaml (also known as WSOO), a compiler translating OCaml bytecode to WebAssembly. Since then, our teams have been working on different improvements to the OCaml ecosystem, including boosting the performance of WSOO.</p>
<p>WSOO is already known for its speed. Users switching from Js_of_ocaml (JSOO), which translates OCaml bytecode to JavaScript, can expect significant performance improvements. Back in early 2025, Jane Street reported that they saw significant <a href="https://tarides.com/blog/2025-02-19-the-first-wasm-of-ocaml-release-is-out/">improvements</a> using WSOO in comparison to JSOO. With the most recent optimisations WSOO is even faster!</p>
<p>You can try Wasm_of_ocaml for your own projects by visiting the manual, which includes installation instructions, <a href="https://ocsigen.org/js_of_ocaml/latest/manual/wasm_overview">on the Ocsigen website</a>.</p>
<h2>How Have We Made Wasm_of_ocaml Faster?</h2>
<p>Almost as soon as the feature-complete release of Wasm_of_ocaml was out, a team at Tarides began working on optimisations for the library. Jérôme Vouillon has led the way, testing each change and measuring the resulting performance improvements. Let’s take a look at some of the PRs that have come out of this effort and what has changed:</p>
<ul>
<li>
<p>Inlining pass: Jérôme rewrote the inlining pass to adjust its behaviour in certain cases. The change avoids inlining code in a loop that WASM engines can’t optimise because V8 currently has no way to switch to a more efficient code while executing a loop. It is now also more assertive in inlining functors and functions, including <code>List.fold_left</code>. The PR is <a href="https://github.com/ocsigen/js_of_ocaml/pull/1935">#1935</a>.</p>
</li>
<li>
<p>The OCaml standard library contains a <code>Bigarray</code> module that offers an API to manipulate an array of numerical values (integers of floating point values of various sizes). When compiling a program to WebAssembly, <code>Bigarray</code> operations are translated behind the scenes as operations on <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Typed_arrays">Javascript “typed arrays”</a>.Up to now, these operations were implemented in terms of direct access to these arrays, but this method went through calls to Javascript functions that incurred some overhead.</p>
<p>Javascript offers an alternative API called <code>DataView</code>, whose operations are recognised and compiled as direct memory accesses by Wasm engines such as Google’s V8. We have modified the backend function supporting <code>Bigarray</code> to use <code>DataView</code> and benchmarked it with a program performing millions of <code>Bigarray</code> accesses, and we are observing an impressive 3.9x speedup.</p>
<p>Check out <a href="https://github.com/ocsigen/js_of_ocaml/pull/1979">PR #1979 to look more closely at the changes</a>.</p>
</li>
<li>
<p>Function call optimisations: There are a whole host of changes targeting function calls:</p>
<ul>
<li>PR <a href="https://github.com/ocsigen/js_of_ocaml/pull/2041">#2041</a> optimises the representation of closures using more precise types.</li>
<li>PR <a href="https://github.com/ocsigen/js_of_ocaml/pull/2044">#2044</a> optimises calls to a statically known function.</li>
<li>PR <a href="https://github.com/ocsigen/js_of_ocaml/pull/2059">#2059</a> omits the code pointer when it is not used (because of the previous optimisation). This reduces the amount of memory allocated by the program, but also allows <code>Binaryen</code> to perform some global optimisations.</li>
</ul>
</li>
<li>
<p>Integer optimisations:  Wasm_of_ocaml uses 31-bit integers to allow for a <a href="https://ocaml.org/docs/memory-representation#distinguishing-integers-and-pointers-at-runtime">uniform representation of integers and references</a>. The integers have to be converted to 32-bit integers to perform numeric operations. PR <a href="https://github.com/ocsigen/js_of_ocaml/pull/2032">#2032</a> optimises this workflow by avoiding unnecessary conversions between 31 and 32-bit integers, which speeds up performance.</p>
</li>
<li>
<p>Number unboxing: Avoids boxing numbers (both <a href="https://github.com/ocsigen/js_of_ocaml/pull/2069">within functions #2069</a> and <a href="https://github.com/ocsigen/js_of_ocaml/pull/2101">outside of functions #2101</a>) when the boxed value is not used. The change significantly improved the microbenchmarks <code>almabench</code> and <code>fft</code>.</p>
</li>
<li>
<p>Number comparisons and <code>bigarray</code> operations: Specialisation of number comparisons and <code>bigarray</code> operations in PR <a href="https://github.com/ocsigen/js_of_ocaml/pull/1954">#1954</a>, based on a type analysis, optimises the performance of functions. Future work to improve on this optimisation centres around using hints from the OCaml compiler.</p>
</li>
</ul>
<p>Hopefully, you now have a good sense of just how much has been tweaked and improved to make WSOO faster! But, you may ask, just <em>how much faster</em> are we talking about?</p>
<h2>Benchmarks</h2>
<p>Several of these changes have contributed to improving WSOO's performance, visualised in the graphs below. The first one is the <a href="https://github.com/linoscope/CAMLBOY">CAMLboy benchmark</a>, showing steady improvement over time until it was one third faster.</p>
<p><img src="https://tarides.com/blog/images/CAMLBoy-bench-1360w~ZKIIldzmpp7-z09HOdethg.webp" sizes="(min-width: 1360px) 1360px, (min-width: 680px) 680px, 100vw" srcset="/blog/images/CAMLBoy-bench-170w~g9I0gwHa3pnM8DMEcFaiWw.webp 170w, /blog/images/CAMLBoy-bench-340w~m1tiaF4quAJuoCFyc2Nh-Q.webp 340w, /blog/images/CAMLBoy-bench-680w~9UQYzMaokwWfb66CI0AliA.webp 680w, /blog/images/CAMLBoy-bench-1360w~ZKIIldzmpp7-z09HOdethg.webp 1360w" alt="A graph in blue showing a delta of performance improvement"></p>
<p>The second graph show the performance improvement for some selected microbenchmarks. Numerical benchmarks, like almabench, fft, nucleic, and raytrace, show a nice improvements. The integer optimisation helps for fib, quicksort, and fft. The raytrace microbenchmarks show the most improvement, thanks to better inlining and number unboxing across function.</p>
<p><img src="https://tarides.com/blog/images/microbench-wsoo-1360w~GMMAz3M8DNJMAhG6MjH0yQ.webp" sizes="(min-width: 1360px) 1360px, (min-width: 680px) 680px, 100vw" srcset="/blog/images/microbench-wsoo-170w~SJTc8JzfRvLzCrG0GO9qig.webp 170w, /blog/images/microbench-wsoo-340w~7SsE0DgRHCGO7nDuKUZzxQ.webp 340w, /blog/images/microbench-wsoo-680w~iMGeWnAvt6pkn5_kRmqRMw.webp 680w, /blog/images/microbench-wsoo-1360w~GMMAz3M8DNJMAhG6MjH0yQ.webp 1360w" alt="A graph of benchmark comparisons showing improvements"></p>
<h2>What’s Next for Wasm_of_ocaml?</h2>
<p>Look out for the next milestone for WSOO: adding WASI 0.1 support! The team have implemented an alternative runtime based on <a href="https://wasi.dev/">WASI</a>, a group of API specifications for software compiled to the <a href="https://www.w3.org/TR/wasm-core-2/">WebAssembly standard (W3C)</a>. WASI support will enable users to execute OCaml programs in new environments, from browsers to clouds to embedded devices. The <a href="https://github.com/ocsigen/js_of_ocaml/pull/1831">PR is still open</a> and feedback is always welcome.</p>
<p>Furthermore, another one of the team’s goals is to test an implementation of effect handlers based on the Stack Switching proposal, you can find the details in <a href="https://github.com/ocsigen/js_of_ocaml/pull/1832">#1882</a>.</p>
<p>Lastly, the team has more work planned to improve the performance of both Js_of_ocaml and Wasm_of_ocaml in 2026.</p>
<h2>Until Next Time</h2>
<p>Remember to visit the manual on Ocsigen’s website to learn more about <a href="https://ocsigen.org/js_of_ocaml/latest/manual/wasm_overview">Wasm_of_ocaml</a> and to get started with the compiler if you haven’t already. The WSOO team is always keen to hear feedback and figure out how they can improve the user experience. Please share your thoughts and experiences with the compiler on OCaml’s <a href="https://discuss.ocaml.org/">discussion forum</a>!</p>
<p>You can connect with us on <a href="https://bsky.app/profile/tarides.com">Bluesky</a>, <a href="https://mastodon.social/@tarides">Mastodon</a>, <a href="https://www.threads.net/@taridesltd">Threads</a>, and <a href="https://www.linkedin.com/company/tarides">LinkedIn</a> or sign up for our mailing list to stay updated on our latest projects. We look forward to hearing from you!</p>

