---
title: Deriving QCheck Generators for External Types in OCaml
description:
url: https://lambdafoo.com/posts/2026-05-01-qcheck-deriving-ocaml.html
date: 2026-05-01T00:00:00-00:00
preview_image:
authors:
- Tim McGilchrist
source:
ignore:
---

<div class="post">
  <h1 class="post-title">Deriving QCheck Generators for External Types in OCaml</h1>
  <span class="post-date">May  1, 2026</span>
  <p>Recently I’ve been working on <a href="https://github.com/tmcgilchrist/durin">durin</a>, a DWARF library for OCaml. I won’t go into the details here but I wanted to share a property testing technique I’ve been using in durin.</p>
<p>The <a href="https://dwarfstd.org">DWARF spec</a> is huge (version 5 is just short of 500 pages) and includes many large variant types that can be combined in different ways. To test the serialisation and deserialisation code I’m using property testing, and in particular a technique I picked up working with <a href="https://github.com/nick8325/quickcheck">QuickCheck</a> in Haskell that I haven’t seen written up for OCaml. Let’s look at how to derive generators for our types using <a href="https://github.com/c-cube/qcheck">QCheck</a>, OCaml’s QuickCheck-inspired property-based testing library.</p>
<h2>Deriving QCheck Generators</h2>
<p>When writing property-based tests in OCaml, you need QCheck generators for the library’s types. These generators then get used when you write your tests. The <a href="https://github.com/c-cube/qcheck#an-introduction-to-the-library">QCheck documentation</a> is a good introduction to the library.</p>
<p>You have a couple of options for where you add these generators. The obvious approach is adding <code>[@@deriving qcheck]</code> directly to the library’s type definition. This uses a PPX (which you can think of as a macro that generates new code before it gets compiled), which pulls <code>qcheck-core</code> and <code>ppx_deriving_qcheck</code> into the library’s dependency tree. This is unfortunate as every consumer of the library picks up a QCheck dependency whether they need it or not.</p>
<p>The second approach would be deriving something like <code>[@@deriving enum]</code> to generate <code>to_enum</code>/<code>of_enum</code> conversions for the variants you want to test, and then writing the QCheck generators by hand in the test code using those conversions. This improves on the previous approach as you avoid library consumers needing QCheck, but you still have the downside of needing a PPX dependency on your library. This might be totally fine if you’re already using PPX elsewhere in the library.</p>
<p>Finally you can write the generator manually, which you often want to do with types that have properties that are hard to encode directly in the type. For example, when generating a date you might want to bias the generation to use modern dates between 1990 and 2030. For now let’s assume you mostly want automatically derived generators.</p>
<p>This post shows a technique that keeps the library free of PPX and QCheck dependencies while still getting derived generators in test code.</p>
<h2>The problem</h2>
<p>Say you have a library with variant types like these from durin:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb1-1" aria-hidden="true" tabindex="-1"></a><span class="co">(* lib/dwarf_constants.ml *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-2" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> accessibility =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-3" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_public</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-4" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_protected</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-5" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_private</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-6" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-7" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> endianity =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-8" aria-hidden="true" tabindex="-1"></a>  | DW_END_default</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-9" aria-hidden="true" tabindex="-1"></a>  | DW_END_big</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-10" aria-hidden="true" tabindex="-1"></a>  | DW_END_little</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-11" aria-hidden="true" tabindex="-1"></a>  | DW_END_lo_user</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-12" aria-hidden="true" tabindex="-1"></a>  | DW_END_hi_user</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-13" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-14" aria-hidden="true" tabindex="-1"></a><span class="co">(* lib/dwarf_form.ml *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-15" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> attribute_form_encoding =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-16" aria-hidden="true" tabindex="-1"></a>  | DW_FORM_addr</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-17" aria-hidden="true" tabindex="-1"></a>  | DW_FORM_block2</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-18" aria-hidden="true" tabindex="-1"></a>  | DW_FORM_data1</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-19" aria-hidden="true" tabindex="-1"></a>  <span class="co">(* ... 45 more constructors ... *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-20" aria-hidden="true" tabindex="-1"></a>  | DW_FORM_unknown <span class="kw">of</span> <span class="dt">int</span>  <span class="co">(** Unknown or vendor-specific forms *)</span></span></code></pre></div>
<p>Now you want to apply the roundtrip test pattern like <code>decode(encode(v)) = v</code> for every variant. For durin I want to ensure I get the same value back that I passed to my serialisation function.</p>
<p>In this case you need a QCheck generator for each type. You might even need a generator for types that you didn’t define and are coming from other libraries. How can we handle this?</p>
<h3>Library provides generators</h3>
<p>The library depends on <code>qcheck-core</code> and ships generators alongside each type:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb2-1" aria-hidden="true" tabindex="-1"></a><span class="co">(* lib/dwarf_constants.ml *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-2" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> accessibility =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-3" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_public</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-4" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_protected</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-5" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_private</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-6" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-7" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> gen_accessibility =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-8" aria-hidden="true" tabindex="-1"></a>  QCheck.Gen.oneofl</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-9" aria-hidden="true" tabindex="-1"></a>    [DW_ACCESS_public; DW_ACCESS_protected; DW_ACCESS_private]</span></code></pre></div>
<p>This gets the job done, but every user of the library now transitively depends on <code>qcheck-core</code>. The library’s <code>.opam</code> file grows and build times increase for everyone, not just people running tests. Often in Haskell I would define a second library called <code>library-test</code> to ship alongside the main library, bundling the generators and any associated test utilities. This is fine for code you aren’t shipping to opam (maybe it’s even fine there but doesn’t seem to be the done thing). What else can we try?</p>
<h3>Library uses ppx_deriving_qcheck</h3>
<p>Rather than writing and shipping generators directly, the library adds the PPX deriver annotation <code>[@@deriving qcheck]</code>.</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb3-1" aria-hidden="true" tabindex="-1"></a><span class="co">(* lib/dwarf_constants.ml *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb3-2" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> accessibility =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb3-3" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_public</span>
<span><a href="https://lambdafoo.com/rss.xml#cb3-4" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_protected</span>
<span><a href="https://lambdafoo.com/rss.xml#cb3-5" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_private</span>
<span><a href="https://lambdafoo.com/rss.xml#cb3-6" aria-hidden="true" tabindex="-1"></a>[@@deriving qcheck]</span></code></pre></div>
<p>This is a variation of the previous approach where the deriver writes the generators for you. It saves developer time, and you get generators even for types you might not have written by hand. The library now depends on both <code>qcheck-core</code> and <code>ppx_deriving_qcheck</code> at build time. PPX dependencies tend to be quite heavy, the PPX machinery for deriving is shipped separately from the compiler even though it’s using the OCaml AST. By using the OCaml AST your project needs to be more careful about the OCaml versions it uses, and it can be difficult to use your code on pre-release versions of OCaml. Which I tend to do a lot as I work on the compiler. So it makes sense to avoid a PPX dependency in certain cases.</p>
<h3>Tests derive generators externally</h3>
<p>What I want is for the test code to only be required when testing. The main library knows nothing about QCheck, while the test executable imports the type definitions and derives generators locally. The PPX dependency is confined to the test executable. This approach also leaves the door open to shipping the generators as a separate <code>library-test</code> package. Before getting into how to wire this up in OCaml, let’s look at the same problem in Haskell.</p>
<h2>The Haskell analogy</h2>
<p>If you’ve used QuickCheck in Haskell, this problem is familiar. When a library defines a type but doesn’t provide an <code>Arbitrary</code> instance (the equivalent to a generator), you can’t write an Arbitrary instance for it directly in test code. GHC warns about orphan instance and the ecosystem generally discourages orphan instances. The standard workaround for this problem is a newtype wrapper.</p>
<div class="sourceCode"><pre class="sourceCode haskell"><code class="sourceCode haskell"><span><a href="https://lambdafoo.com/rss.xml#cb4-1" aria-hidden="true" tabindex="-1"></a><span class="co">-- Library defines:</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-2" aria-hidden="true" tabindex="-1"></a><span class="kw">data</span> <span class="dt">Color</span> <span class="ot">=</span> <span class="dt">Red</span> <span class="op">|</span> <span class="dt">Green</span> <span class="op">|</span> <span class="dt">Blue</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-3" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-4" aria-hidden="true" tabindex="-1"></a><span class="co">-- Test code:</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-5" aria-hidden="true" tabindex="-1"></a><span class="kw">newtype</span> <span class="dt">TestColor</span> <span class="ot">=</span> <span class="dt">TestColor</span> <span class="dt">Color</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-6" aria-hidden="true" tabindex="-1"></a>  <span class="kw">deriving</span> <span class="kw">newtype</span> (<span class="dt">Eq</span>, <span class="dt">Show</span>)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-7" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-8" aria-hidden="true" tabindex="-1"></a><span class="kw">instance</span> <span class="dt">Arbitrary</span> <span class="dt">TestColor</span> <span class="kw">where</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-9" aria-hidden="true" tabindex="-1"></a>  arbitrary <span class="ot">=</span> <span class="dt">TestColor</span> <span class="op">&lt;$&gt;</span> elements [<span class="dt">Red</span>, <span class="dt">Green</span>, <span class="dt">Blue</span>]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-10" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-11" aria-hidden="true" tabindex="-1"></a><span class="ot">prop_roundtrip ::</span> <span class="dt">TestColor</span> <span class="ot">-&gt;</span> <span class="dt">Bool</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb4-12" aria-hidden="true" tabindex="-1"></a>prop_roundtrip (<span class="dt">TestColor</span> c) <span class="ot">=</span> decode (encode c) <span class="op">==</span> c</span></code></pre></div>
<p>The newtype is zero-cost at runtime but gives you a place to hang the instance without orphan warnings. The key insight is the type definition is structurally duplicated in test scope so the typeclass machinery can operate on it. Haskell also has a built-in deriving mechanism similar to OCaml’s PPX.</p>
<h2>Solving with QCheck</h2>
<p>OCaml doesn’t have typeclasses or orphan instances, but the problem is structurally identical. A PPX deriver needs to see the type definition in the file where it runs. If the definition is in the library and the library doesn’t use the PPX, the deriver never sees it.</p>
<p>Enter <code>ppx_import</code> with a clever solution. It copies a type definition from a compiled module interface file (<code>.cmi</code>) into the current file at pre-processing time. A <code>.cmi</code> is the OCaml compiler’s binary form of an <code>.mli</code> interface, holding type information and module signatures but no implementation. If you want the bigger picture of what the compiler produces, the <a href="https://ocaml.org/docs/using-the-ocaml-compiler-toolchain">OCaml compiler toolchain docs</a> cover the other artifacts (<code>.cmo</code>, <code>.cmx</code>, <code>.cma</code>, <code>.cmxa</code>, and friends).</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb5-1" aria-hidden="true" tabindex="-1"></a><span class="co">(* In test code, NOT in the library *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb5-2" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> accessibility = [%import: Durin.Dwarf.accessibility] [@@deriving qcheck]</span></code></pre></div>
<p>After preprocessing, this expands to:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb6-1" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> accessibility =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb6-2" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_public</span>
<span><a href="https://lambdafoo.com/rss.xml#cb6-3" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_protected</span>
<span><a href="https://lambdafoo.com/rss.xml#cb6-4" aria-hidden="true" tabindex="-1"></a>  | DW_ACCESS_private</span>
<span><a href="https://lambdafoo.com/rss.xml#cb6-5" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb6-6" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> gen_accessibility =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb6-7" aria-hidden="true" tabindex="-1"></a>  QCheck.Gen.oneof</span>
<span><a href="https://lambdafoo.com/rss.xml#cb6-8" aria-hidden="true" tabindex="-1"></a>    [ QCheck.Gen.pure DW_ACCESS_public;</span>
<span><a href="https://lambdafoo.com/rss.xml#cb6-9" aria-hidden="true" tabindex="-1"></a>      QCheck.Gen.pure DW_ACCESS_protected;</span>
<span><a href="https://lambdafoo.com/rss.xml#cb6-10" aria-hidden="true" tabindex="-1"></a>      QCheck.Gen.pure DW_ACCESS_private ]</span></code></pre></div>
<p>The imported type is structurally identical to <code>Durin.Dwarf.accessibility</code> which OCaml can unify. No newtype wrapper, no coercion, no runtime cost. Unlike Haskell, there’s no orphan instance concern because OCaml generators are plain values, not typeclass instances. We get all the benefits of QCheck and PPX without making our library users depend on them.</p>
<p>One caveat worth flagging. <code>ppx_import</code> can only copy type definitions that are actually exposed in the library’s <code>.mli</code>. If a type is abstract (declared as <code>type foo</code> with no right-hand side), there is nothing for import to copy and you’ll need to fall back to a hand-written generator in the test code. At that point you would be questioning why you don’t have access to the type and if it should be present in the <code>.mli</code> file.</p>
<h2>Step-by-step setup</h2>
<p>This is how I’ve been setting up this pattern in durin.</p>
<h3>Add test-only dependencies</h3>
<p>In your <code>dune-project</code>, add the PPX packages scoped to tests:</p>
<pre><code>(package
 (name durin)
 (depends
  (ocaml (&gt;= 5.3))
  ;; ... your library dependencies ...
  (ppx_import :with-test)
  (ppx_deriving_qcheck :with-test)
  (qcheck-core :with-test)
  (qcheck-alcotest :with-test)))</code></pre>
<p>The <code>:with-test</code> scope means these packages are never required by library consumers. They’re only installed when running tests or installed with <code>opam install --with-test</code>.</p>
<h3>Configure the test stanza with staged_pps</h3>
<p>In your <code>test/dune</code>:</p>
<pre><code>(test
 (name test_roundtrip)
 (libraries durin alcotest qcheck-core qcheck-alcotest)
 (preprocess (staged_pps ppx_import ppx_deriving_qcheck)))</code></pre>
<p>I’m using <code>staged_pps</code> rather than plain <code>pps</code> because <code>ppx_import</code> requires two passes. The first pass runs through ocamldep to work out which modules the file depends on, and only the second pass has the <code>.cmi</code> files available to copy the type definitions from. Regular <code>pps</code> runs in a single pass where <code>ppx_import</code> won’t have the <code>.cmi</code> files available yet.</p>
<h3>Import types and derive generators</h3>
<p>Import the types using ppx_import and derive the qcheck generators:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb9-1" aria-hidden="true" tabindex="-1"></a><span class="co">(* test_roundtrip.ml *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-2" aria-hidden="true" tabindex="-1"></a><span class="kw">open</span> Durin.Dwarf</span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-3" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-4" aria-hidden="true" tabindex="-1"></a><span class="co">(* ppx_import copies the type definition from the .mli.</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-5" aria-hidden="true" tabindex="-1"></a><span class="co">   ppx_deriving_qcheck generates gen_&lt;name&gt; : &lt;name&gt; QCheck.Gen.t</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-6" aria-hidden="true" tabindex="-1"></a><span class="co">   for each type. *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-7" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> accessibility = [%import: Durin.Dwarf.accessibility]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-8" aria-hidden="true" tabindex="-1"></a>[@@deriving qcheck]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-9" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-10" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> endianity = [%import: Durin.Dwarf.endianity]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-11" aria-hidden="true" tabindex="-1"></a>[@@deriving qcheck]</span></code></pre></div>
<p>This gives you <code>gen_accessibility : accessibility QCheck.Gen.t</code> and <code>gen_endianity : endianity QCheck.Gen.t</code> to use in your tests.</p>
<h3>Customise types with payloads</h3>
<p>For variants carrying data, <code>ppx_deriving_qcheck</code> generates sub-generators automatically for standard OCaml types (<code>int</code>, <code>string</code>, <code>float</code>, <code>bool</code>, etc.):</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb10-1" aria-hidden="true" tabindex="-1"></a><span class="co">(* Library defines:</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-2" aria-hidden="true" tabindex="-1"></a><span class="co">   type attribute_form_encoding =</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-3" aria-hidden="true" tabindex="-1"></a><span class="co">     | DW_FORM_addr</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-4" aria-hidden="true" tabindex="-1"></a><span class="co">     | ...</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-5" aria-hidden="true" tabindex="-1"></a><span class="co">     | DW_FORM_unknown of int *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-6" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> attribute_form_encoding = [%import: Durin.Dwarf.attribute_form_encoding]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-7" aria-hidden="true" tabindex="-1"></a>[@@deriving qcheck]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-8" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-9" aria-hidden="true" tabindex="-1"></a><span class="co">(* Generates a generator that picks one of the constructors,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-10" aria-hidden="true" tabindex="-1"></a><span class="co">   passing a generated int to DW_FORM_unknown:</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-11" aria-hidden="true" tabindex="-1"></a><span class="co">   let gen_attribute_form_encoding =</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-12" aria-hidden="true" tabindex="-1"></a><span class="co">     QCheck.Gen.oneof</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-13" aria-hidden="true" tabindex="-1"></a><span class="co">       [ QCheck.Gen.pure DW_FORM_addr;</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-14" aria-hidden="true" tabindex="-1"></a><span class="co">         ...</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb10-15" aria-hidden="true" tabindex="-1"></a><span class="co">         QCheck.Gen.map (fun i -&gt; DW_FORM_unknown i) QCheck.Gen.int ] *)</span></span></code></pre></div>
<p>If the payload is another type from your library, import and derive the inner type first. The outer generator will then reference it by name:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb11-1" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> base_type = [%import: Durin.Dwarf.base_type] [@@deriving qcheck]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-2" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> dwarf_language = [%import: Durin.Dwarf.dwarf_language] [@@deriving qcheck]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-3" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> attribute_value = [%import: Durin.Dwarf.DIE.attribute_value] [@@deriving qcheck]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-4" aria-hidden="true" tabindex="-1"></a><span class="co">(* gen_attribute_value uses gen_base_type for the Encoding constructor</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-5" aria-hidden="true" tabindex="-1"></a><span class="co">   and gen_dwarf_language for the Language constructor *)</span></span></code></pre></div>
<p>Sometimes the default generator for a field is wrong, for example a DWARF version that must be 2 to 5. You can override it with a <code>[@gen ...]</code> attribute, but attributes don’t survive the <code>[%import: ...]</code> expansion, so restate the record definition in test scope using OCaml’s type sharing:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb12-1" aria-hidden="true" tabindex="-1"></a><span class="co">(* Library: type encoding = {</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb12-2" aria-hidden="true" tabindex="-1"></a><span class="co">     format : dwarf_format;</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb12-3" aria-hidden="true" tabindex="-1"></a><span class="co">     address_size : u8;</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb12-4" aria-hidden="true" tabindex="-1"></a><span class="co">     version : u16;</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb12-5" aria-hidden="true" tabindex="-1"></a><span class="co">   } *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb12-6" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> encoding = Durin.Dwarf.encoding = {</span>
<span><a href="https://lambdafoo.com/rss.xml#cb12-7" aria-hidden="true" tabindex="-1"></a>  <span class="dt">format</span> : dwarf_format;</span>
<span><a href="https://lambdafoo.com/rss.xml#cb12-8" aria-hidden="true" tabindex="-1"></a>  address_size : u8;</span>
<span><a href="https://lambdafoo.com/rss.xml#cb12-9" aria-hidden="true" tabindex="-1"></a>  version : u16 [@gen QCheck.Gen.map Unsigned.UInt16.of_int (QCheck.Gen.int_range <span class="dv">2</span> <span class="dv">5</span>)];</span>
<span><a href="https://lambdafoo.com/rss.xml#cb12-10" aria-hidden="true" tabindex="-1"></a>}</span>
<span><a href="https://lambdafoo.com/rss.xml#cb12-11" aria-hidden="true" tabindex="-1"></a>[@@deriving qcheck]</span></code></pre></div>
<p>The <code>type encoding = Durin.Dwarf.encoding = { ... }</code> form tells OCaml that <code>encoding</code> is the same type as <code>Durin.Dwarf.encoding</code>, so values flow between them without coercion. For simple type aliases where restating the definition doesn’t help, skip the deriver and write the generator by hand:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb13-1" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> register = [%import: Durin.Dwarf.register]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb13-2" aria-hidden="true" tabindex="-1"></a><span class="co">(* AArch64 has 31 general-purpose registers *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb13-3" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> gen_register =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb13-4" aria-hidden="true" tabindex="-1"></a>  QCheck.Gen.map (<span class="kw">fun</span> n -&gt; Register n) (QCheck.Gen.int_range <span class="dv">0</span> <span class="dv">30</span>)</span></code></pre></div>
<h3>Write the actual tests</h3>
<p><code>ppx_deriving_qcheck</code> produces <code>QCheck.Gen.t</code>, not <code>QCheck.arbitrary</code>. Wrap with <code>QCheck.make</code> for use with <code>QCheck.Test.make</code> like so:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb14-1" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> arb gen = QCheck.make gen</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-2" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-3" aria-hidden="true" tabindex="-1"></a><span class="co">(* A generic roundtrip helper *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-4" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> roundtrip name gen encode decode =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-5" aria-hidden="true" tabindex="-1"></a>  QCheck.Test.make ~name:(name ^ <span class="st">" roundtrip"</span>)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-6" aria-hidden="true" tabindex="-1"></a>    (arb gen)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-7" aria-hidden="true" tabindex="-1"></a>    (<span class="kw">fun</span> v -&gt; decode (encode v) = v)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-8" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-9" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> tests =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-10" aria-hidden="true" tabindex="-1"></a>  [</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-11" aria-hidden="true" tabindex="-1"></a>    roundtrip <span class="st">"accessibility"</span> gen_accessibility</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-12" aria-hidden="true" tabindex="-1"></a>      int_of_accessibility accessibility;</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-13" aria-hidden="true" tabindex="-1"></a>    roundtrip <span class="st">"endianity"</span> gen_endianity</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-14" aria-hidden="true" tabindex="-1"></a>      int_of_endianity endianity;</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-15" aria-hidden="true" tabindex="-1"></a>  ]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-16" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-17" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> () =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-18" aria-hidden="true" tabindex="-1"></a>  <span class="kw">let</span> <span class="kw">open</span> Alcotest <span class="kw">in</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-19" aria-hidden="true" tabindex="-1"></a>  run <span class="st">"roundtrip tests"</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-20" aria-hidden="true" tabindex="-1"></a>    [</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-21" aria-hidden="true" tabindex="-1"></a>      ( <span class="st">"roundtrip"</span>,</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-22" aria-hidden="true" tabindex="-1"></a>        <span class="dt">List</span>.map QCheck_alcotest.to_alcotest tests );</span>
<span><a href="https://lambdafoo.com/rss.xml#cb14-23" aria-hidden="true" tabindex="-1"></a>    ]</span></code></pre></div>
<h2>A real-world example</h2>
<p>Here’s the complete test file from <a href="https://github.com/tmcgilchrist/durin">durin</a>, a DWARF debugging format library. It tests roundtrip properties for 26 DWARF constant types, each with from 2 to over 170 variants:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb15-1" aria-hidden="true" tabindex="-1"></a><span class="kw">open</span> Durin.Dwarf</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-2" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-3" aria-hidden="true" tabindex="-1"></a><span class="co">(* Import and derive generators for every type under test.</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-4" aria-hidden="true" tabindex="-1"></a><span class="co">   When a variant is added to any type in the library, the</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-5" aria-hidden="true" tabindex="-1"></a><span class="co">   generator updates automatically. *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-6" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> abbreviation_tag =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-7" aria-hidden="true" tabindex="-1"></a>  [%import: Durin.Dwarf.abbreviation_tag]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-8" aria-hidden="true" tabindex="-1"></a>[@@deriving qcheck]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-9" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-10" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> attribute_encoding =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-11" aria-hidden="true" tabindex="-1"></a>  [%import: Durin.Dwarf.attribute_encoding]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-12" aria-hidden="true" tabindex="-1"></a>[@@deriving qcheck]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-13" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-14" aria-hidden="true" tabindex="-1"></a><span class="kw">type</span> calling_convention =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-15" aria-hidden="true" tabindex="-1"></a>  [%import: Durin.Dwarf.calling_convention]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-16" aria-hidden="true" tabindex="-1"></a>[@@deriving qcheck]</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-17" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-18" aria-hidden="true" tabindex="-1"></a><span class="co">(* ... 23 more types ... *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-19" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-20" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> arb gen = QCheck.make gen</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-21" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-22" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> roundtrip name gen encode decode =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-23" aria-hidden="true" tabindex="-1"></a>  QCheck.Test.make ~name:(name ^ <span class="st">" roundtrip"</span>)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-24" aria-hidden="true" tabindex="-1"></a>    (arb gen)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-25" aria-hidden="true" tabindex="-1"></a>    (<span class="kw">fun</span> v -&gt; decode (encode v) = v)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-26" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-27" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> tests =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-28" aria-hidden="true" tabindex="-1"></a>  [</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-29" aria-hidden="true" tabindex="-1"></a>    roundtrip <span class="st">"abbreviation_tag"</span> gen_abbreviation_tag</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-30" aria-hidden="true" tabindex="-1"></a>      uint64_of_abbreviation_tag abbreviation_tag_of_int;</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-31" aria-hidden="true" tabindex="-1"></a>    roundtrip <span class="st">"attribute_encoding"</span> gen_attribute_encoding</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-32" aria-hidden="true" tabindex="-1"></a>      u64_of_attribute_encoding attribute_encoding;</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-33" aria-hidden="true" tabindex="-1"></a>    roundtrip <span class="st">"calling_convention"</span> gen_calling_convention</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-34" aria-hidden="true" tabindex="-1"></a>      int_of_calling_convention calling_convention;</span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-35" aria-hidden="true" tabindex="-1"></a>    <span class="co">(* ... *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb15-36" aria-hidden="true" tabindex="-1"></a>  ]</span></code></pre></div>
<h2>Conclusion</h2>
<p>This is a useful technique to add to your OCaml property-testing toolkit. It is self-maintaining, so when you add a variant to a type in the library, <code>ppx_import</code> picks up the new definition on the next build and <code>ppx_deriving_qcheck</code> generates a generator that includes it.</p>
<p>The library side stays clean. No PPX dependencies, no QCheck dependency, no generated code in its build artifacts. The <code>.opam</code> file and build are unchanged, and consumers who never run tests never see these test packages. The imported type is also the same OCaml type as the library’s, so there are no type-safety issues to worry about. It keeps open the option for defining a test library that just includes useful generators.</p>
<p>As a bonus, the same approach lets you define generators for types outside of your control, which is handy when you want to test round-trips against data from another library. I’ve found this pattern very useful and I hope you do too.</p>
</div>

