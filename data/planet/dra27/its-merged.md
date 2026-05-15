---
title: "It\u2019s merged!!"
description: As I was very happy to announce on Discuss on 12 December, OCaml is Relocatable!
  Today, the final piece of the puzzle was merged, which is the necessary support
  to allow opam to take advantage of all this to be able to clone switches instead
  of recompiling them. Before this, you could rename a local switch, for example,
  but opam would still be compiling them from scratch. The fruits of this continue
  to be available through my opam-repository fork, as announced in September, and
  will be available out-of-the-box for the first alpha release of OCaml 5.5 early
  next year.
url: https://www.dra27.uk/blog/platform/2025/12/17/its-merged.html
date: 2025-12-17T00:00:00-00:00
preview_image:
authors:
- ""
source:
ignore:
---

<p>As I was very happy to <a href="https://discuss.ocaml.org/t/volunteers-to-review-the-relocatable-ocaml-work/16667/11">announce on Discuss</a>
on 12 December, <a href="https://github.com/ocaml/ocaml/commit/7e71861a1ac1ea472104f40168f42133da02096c">OCaml is Relocatable</a>!
Today, the final piece of the puzzle was merged, which is the necessary support
to allow opam to take advantage of all this to be able to clone switches instead
of recompiling them. Before this, you could <em>rename</em> a local switch, for
example, but opam would still be compiling them from scratch. The fruits of this
continue to be available <a href="https://discuss.ocaml.org/t/relocatable-ocaml/17253">through my opam-repository fork</a>,
as <a href="https://www.dra27.uk/blog/platform/2025/09/15/relocatable-ocaml.html">announced in September</a>, and
will be available out-of-the-box for the first alpha release of OCaml 5.5 early
next year.</p>

<p>I am particularly thrilled, and just a little surprised, that Relocatable OCaml
has happened as I envisioned, with nothing needing to be dropped. There were of
course many useful suggestions for re-working to improve clarity and so forth
during review, but the test harnesses, RFCs, talks, and so forth appear to have
provided me with the required knife to work out correctly what was <em>necessary</em>
to achieve all this!</p>

<p>For posterity, here’s the timeline! On 4 May, <a href="https://github.com/ocaml/ocaml/pull/14014">ocaml/ocaml#14014</a>
introduced a comprehensive test harness for the <a href="https://github.com/ocaml/RFCs/pull/53">Relocatable OCaml RFC</a>,
which was reviewed by <a href="https://github.com/nojb">Nicolás Ojeda Bär</a> and
<a href="https://github.com/OlivierNicole">Olivier Nicole</a> and merged on 21 June.</p>

<p>Finalising the PRs took me a little while longer than I’d expected, and the set
of 4 PRs, along with a fifth “combined” demonstration <a href="https://github.com/ocaml/ocaml/pulls?q=is:pr%20author:dra27%20created:2025-09-15">were opened on
15 September</a>.
Over the coming weeks, <a href="https://github.com/jonahbeckford">Jonah Beckford</a>,
<a href="https://github.com/MisterDA">Antonin Décimo</a>, <a href="https://github.com/hhugo">Hugo Heuzard</a>,
<a href="https://github.com/shym">Samuel Hym</a> and <a href="https://github.com/lthls">Vincent Laviron</a>
pored over the details. Their invaluable review then set the scene for a
marathon synchronous <a href="https://bsky.app/profile/dra27.uk/post/3m6htzdhxok2f">defence of the branches</a>
in Paris with <a href="https://github.com/damiendoligez">Damien Doligez</a> on 25 November.
I’m very grateful to everyone involved in these reviews: a lot of code; and a
lot of gnarly details!</p>

<p>That left a small todo list for each PR. I managed to coerce a different core
maintainer for each, with <a href="https://github.com/Octachron">Florian Angeletti</a>
merging <a href="https://github.com/ocaml/ocaml/pull/14243">ocaml/ocaml#14243</a> on
27 November, <a href="https://github.com/gasche">Gabriel Scherer</a> merging
<a href="https://github.com/ocaml/ocaml/pull/14244">ocaml/ocaml#14244</a> on 8 December,
Nicolás merging <a href="https://github.com/ocaml/ocaml/pull/14245">ocaml/ocaml#14245</a>
on 12 December and then finally <a href="https://github.com/NickBarnes">Nick Barnes</a>
merging <a href="https://github.com/ocaml/ocaml/pull/14246">ocaml/ocaml#14246</a> today.</p>

<p>That means that OCaml’s trunk branch now matches the bulk of <a href="https://github.com/ocaml/ocaml/pull/14247">ocaml/ocaml#14247</a>.
What’s next? There’s a small amount of work still to do on the scripts which
plumb the ocaml opam package together. Then, having now got everything merged,
it’ll be time to finalise the backports for proposed re-releases of the older
compilers.</p>

<p>But, for now, the future’s very definitely relocatable! 🥳</p>
