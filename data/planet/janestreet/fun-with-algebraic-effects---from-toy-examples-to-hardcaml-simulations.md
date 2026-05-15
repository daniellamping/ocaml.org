---
title: Fun with Algebraic Effects - from Toy Examples to Hardcaml Simulations
description: I recently ported the Hardcaml_step_testbench library, one of the libraries
  that we use at Jane Street for Hardcaml simulations, from using monads to using
  a...
url: https://blog.janestreet.com/fun-with-algebraic-effects-hardcaml/
date: 2026-01-06T00:00:00-00:00
preview_image: https://blog.janestreet.com/fun-with-algebraic-effects-hardcaml/banner.png
authors:
- Jane Street Tech Blog
source:
ignore:
---

<p>I recently ported the <a href="https://github.com/janestreet/hardcaml_step_testbench"><code class="highlighter-rouge">Hardcaml_step_testbench</code>
library</a>, one of the libraries that
we use at Jane Street for Hardcaml simulations, from using monads to using algebraic
effects, a new OCaml 5 feature. This blog post walks through what algebraic effects are,
why you should consider using them in lieu of monads, and how to actually work with them
using the <a href="https://github.com/janestreet/handled_effect"><code class="highlighter-rouge">Handled_effect</code> library</a>. One
thing I’ve come to believe is that most of what can be done with monads can be done with
algebraic effects in a much more elegant way.</p>


