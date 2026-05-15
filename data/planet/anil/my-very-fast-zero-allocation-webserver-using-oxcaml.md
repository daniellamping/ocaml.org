---
title: My (very) fast zero-allocation webserver using OxCaml
description: Building httpz, a high-performance HTTP/1.1 parser with zero heap allocation
  using OxCaml's unboxed types, local allocations, and mutable local variables.
url: https://anil.recoil.org/notes/oxcaml-httpz
date: 2026-02-01T00:00:00-00:00
preview_image: https://anil.recoil.org/images/claude-oxlocal-1.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>Since helping with the <a href="https://anil.recoil.org/notes/icfp25-oxcaml">OxCaml tutorial</a> last year at ICFP,
I've been chomping at the bit to use it for real in our research infrastructure
for <a href="https://anil.recoil.org/projects/plancomp">planetary computing</a> to manage the petabytes of <a href="https://anil.recoil.org/notes/geotessera-python">TESSERA embeddings</a> we've been generating.</p>
<p>The reason for my eagerness is that OxCaml has a number of language extensions
that give giant leaps in performance for systems-oriented programs, while
retaining the familiar OCaml functional style of programming. And unlike Rust,
there's a garbage collector available for 'normal' code. I am also deeply sick
and tired of maintaining large Python scripts recently, and crave the modularity and
type safety of OCaml.</p>
<p>The traditional way I learn a new technology is by replacing my <a href="https://anil.recoil.org/notes/bushel-lives">website infrastructure</a> with the latest hotness. I switched my live site
over to building with OxCaml last year, but never got around to deeply
integrating the new extensions. Therefore, what I'll talk about next is a new
webserver I've been building called
<strong><a href="https://github.com/avsm/oxmono/tree/e0b061c0f6621c80e3a990d02867e3302fd7ce16/avsm/httpz">httpz</a></strong>
which goes all in on performance in OCaml!</p>
<p><em>(Many thanks to <a href="https://tyconmismatch.com/code.html">Chris Casinghino</a>, <a href="https://thenumb.at/">Max Slater</a>, <a href="https://richarde.dev/">Richard Eisenberg</a>, <a href="https://github.com/yminsky">Yaron Minsky</a>, <a href="https://github.com/mshinwell">Mark Shinwell</a>, <a href="https://www.dra27.uk">David Allsopp</a> and the rest of the Jane Street
tools and compilers team for answering many questions while I got started on all this!)</em></p>
<h2><a href="https://anil.recoil.org/news.xml#why-zero-allocation-for-http11" class="anchor" aria-hidden="true"></a>Why Zero Allocation for HTTP/1.1?</h2>
<p><a href="https://github.com/avsm/oxmono/tree/e0b061c0f6621c80e3a990d02867e3302fd7ce16/avsm/httpz">httpz</a> is a high-performance HTTP/1.1 parser that aims to have no major heap allocation, and very minimal minor heap allocation, by using OxCaml's <a href="https://oxcaml.org/documentation/unboxed-types/01-intro/">unboxed types</a> and <a href="https://oxcaml.org/documentation/stack-allocation/intro/">local allocations</a>.</p>
<p>Why is this useful?  It means that the entire lifetime of an HTTP connection
can be handled in the callstack alone, so freeing up a connection is just a
matter of returning from the function that handles it. In the steady state, a
webserver would have almost no garbage collector activity. When combined with
<a href="https://anil.recoil.org/papers/2021-pldi-retroeff">direct style effects</a>, it can also be written without
looking like callback soup!</p>
<p>I decided to specialise this library for HTTP/1.1 for now, and so settled on
the input being a simple 32KB bytes value. This represents an HTTP request with
the header portion (HTTP body handling is relatively straightforward for POST
requests, and not covered in this post).</p>
<p>Given an input buffer like this, what can we do with OxCaml <em>vs</em> vanilla OCaml
to make this go fast?</p>
<h3><a href="https://anil.recoil.org/news.xml#unboxed-types-and-records" class="anchor" aria-hidden="true"></a>Unboxed Types and Records</h3>
<p>The first port of call is to figure out the core types we're going to use for our
parser. If you need to get familiar with OCaml's upstream memory representation then
<a href="https://dev.realworldocaml.org/runtime-memory-layout.html">head over to Real World OCaml</a>.</p>
<p>In my usual OCaml code, I use libraries like <a href="https://github.com/mirage/ocaml-cstruct">cstruct</a>
that I <a href="https://anil.recoil.org/projects/unikernels">originally</a> wrote back in 2012 to manage non-copying views into bytes buffers. Cstruct
defines a record that has four words (the box, and three words for the fields):</p>
<pre><code class="language-ocaml">type buffer = (char, Bigarray.int8_unsigned_elt, Bigarray.c_layout) Bigarray.Array1.t
type Cstruct.t = private {
  buffer: buffer;
  off   : int;
  len   : int;
}
</code></pre>
<p>The idea is to use the record to get narrow views into a larger buffer, and that these
small views can just live on the minor heap of the runtime which is fast to collect.
OxCaml advances this by providing unboxed versions of <a href="https://oxcaml.org/documentation/miscellaneous-extensions/small-numbers/">small
numbers</a>
that live in registers or on the stack, via a new syntax <code>int16#</code>.</p>
<p>Instead of Bigarrays, we're now going to switch to use <code>bytes</code> instead, but the
basic idea is the same.  Since httpz's buffer is a max of 32KB, 16-bit integers
also suffice for all positions and lengths!</p>
<pre><code class="language-ocaml">type Httpz.t = #{ off : int16# ; len : int16# }
</code></pre>
<p>There are actually two new features here: the first is that records can be unboxed with the <code>#{}</code>
syntax, and the contents themselves are of a smaller width.  Let's have a closer look
at the difference between the Cstruct boxed version and this new OxCaml one:</p>
<h4><a href="https://anil.recoil.org/news.xml#inspect-unboxing-in-utop" class="anchor" aria-hidden="true"></a>Inspect unboxing in utop</h4>
<p>My first port-of-call is usually to use utop interactively to poke around
using the <code>Obj</code> module.  This isn't quite so easy in OxCaml since the unboxed
records use a special <a href="https://oxcaml.org/documentation/unboxed-types/01-intro/">layout</a>:</p>
<pre><code># type t = #{ off : int16# ; len : int16# };;
type t = #{ off : int16#; len : int16#; }

# let x = #{ off=#1S; len=#2S };;
val x : t = #{off = &lt;abstr&gt;; len = &lt;abstr&gt;}

# Obj.repr x;;
Error: This expression has type t but an expression was expected of type
         ('a : value)
       The layout of t is bits16 &amp; bits16
         because of the definition of t at line 1, characters 0-41.
       But the layout of t must be a sublayout of value.

</code></pre>
<p>That failed, but it did reveal that we have this intriguing int16 pair layout
instead of the normal OCaml flat value representation!  Let's use the compiler
to figure this out...</p>
<h4><a href="https://anil.recoil.org/news.xml#inspect-unboxing-in-lambda" class="anchor" aria-hidden="true"></a>Inspect unboxing in lambda</h4>
<p>I next built a small
test program and inspected the <a href="https://dev.realworldocaml.org/compiler-backend.html">lambda intermediate language</a> from the compiler. To avoid dependencies, I just bound the raw compiler internals directly by checking out the oxcaml source code.</p>
<pre><code class="language-ocaml">external add_int16 : int16# -&gt; int16# -&gt; int16# = "%int16#_add"
external int16_to_int : int16# -&gt; int = "%int_of_int16#"

type span = #{ off : int16#; len : int16# }

let[@inline never] add_spans (x : span) (y : span) : span =
  #{ off = add_int16 x.#off y.#off; len = add_int16 x.#len y.#len }

let () =
  let x = Sys.opaque_identity #{ off = #1S; len = #2S } in
  let y = Sys.opaque_identity #{ off = #100S; len = #200S } in
  let z = add_spans x y in
  Printf.printf "off=%d len=%d\n" (int16_to_int z.#off) (int16_to_int z.#len)
</code></pre>
<p>This introduces enough compiler optimisation barriers such that
the addition is not optimised away at compile time. We can compile this
with <code>ocaml -dlambda src.ml</code> and see the intermediate form after type checking:</p>
<pre><code>(let
  (add_spans/290 =
     (function {nlocal = 0} x/292[#(int16, int16)] y/293[#(int16, int16)]
       never_inline : #(int16, int16)
       (funct-body add_spans ./x.ml(6)&lt;ghost&gt;:196-294
         (before add_spans ./x.ml(7):229-294
           (make_unboxed_product #(int16, int16)
             (%int16#_add (unboxed_product_field 0 #(int16, int16) x/292)
               (unboxed_product_field 0 #(int16, int16) y/293))
             (%int16#_add (unboxed_product_field 1 #(int16, int16) x/292)
               (unboxed_product_field 1 #(int16, int16) y/293)))))))
</code></pre>
<p>You can see the unboxing propagating nicely here through the intermediate code!</p>
<h4><a href="https://anil.recoil.org/news.xml#inspect-unboxing-in-native-code" class="anchor" aria-hidden="true"></a>Inspect unboxing in native code</h4>
<p>The next step is to verify what this looks like when compiled as optimised native
code. I used <code>ocamlopt -O3 -S</code> on my arm64 machine which emits the assembly code
after all the compiler passes, and found:</p>
<pre><code>In the entry point:
  orr   x0, xzr, #1      ; x.#off = 1
  orr   x1, xzr, #2      ; x.#len = 2
  movz  x2, #100, lsl #0 ; y.#off = 100
  movz  x3, #200, lsl #0 ; y.#len = 200
  bl    _camlX__add_spans_0_1_code

_camlX__add_spans_0_1_code:
  add   x1, x1, x3       ; len: x.#len + y.#len
  sbfm  x1, x1, #0, #15  ; sign-extend to 16 bits (int16# semantics)
  add   x0, x0, x2       ; off: x.#off + y.#off
  sbfm  x0, x0, #0, #15  ; sign-extend to 16 bits
  ret

</code></pre>
<p>We can see from the assembly that there's no boxing, and no heap allocations,
and the <a href="https://finkmartin.com/aarch64-morello/sbfm.html">sbfm instruction</a> maintains
the 16-bit semantics via sign extension.</p>
<p>Let's double check that the normal boxed OCaml does do more work and that isn't
just the flambda2 compiler doing its magic.  Here's a boxed version of the benchmark using
plain OCaml:</p>
<pre><code>type span = { off : int; len : int }

let[@inline never] add_spans (x : span) (y : span) : span =
  { off = x.off + y.off; len = x.len + y.len }

let () =
  let x = Sys.opaque_identity { off = 1; len = 2 } in
  let y = Sys.opaque_identity { off = 100; len = 200 } in
  let z = add_spans x y in
  Printf.printf "off=%d len=%d\n" z.off z.len
</code></pre>
<p>Compiling this boxed version with <code>ocamlopt -O3 -S</code> and looking at the assembly shows
much more minor heap activity:</p>
<pre><code>_camlY__add_spans_0_1_code:
      sub   sp, sp, #16
      str   x30, [sp, #8]
      mov   x2, x0
      ldr   x16, [x28, #0]        ; load young_limit
      sub   x27, x27, #24         ; bump allocator: reserve 24 bytes (3 words)
      cmp   x27, x16              ; check if GC needed
      b.cc  L114                  ; branch to GC if out of space
  L113:
      add   x0, x27, #8           ; x0 = pointer to new block
      orr   x3, xzr, #2048        ; header word (tag 0, size 2)
      str   x3, [x0, #-8]         ; write header
      ldr   x3, [x1, #0]          ; load y.off from heap
      ldr   x4, [x2, #0]          ; load x.off from heap
      add   x3, x4, x3            ; add them
      sub   x3, x3, #1            ; adjust for tagged int
      str   x3, [x0, #0]          ; store result.off to heap
      ldr   x1, [x1, #8]          ; load y.len from heap
      ldr   x2, [x2, #8]          ; load x.len from heap
      add   x1, x2, x1            ; add them
      sub   x1, x1, #1            ; adjust for tagged int
      str   x1, [x0, #8]          ; store result.len to heap
      ...
      ret
  L114:
      bl    _caml_call_gc         ; GC call if needed
</code></pre>
<p>The OCaml minor heap is really fast, but it's nowhere near as
fast as just passing values around in registers and doing
direct operations, which the unboxed version lets us do!</p>
<p>My benchmark above used direct external calls to compiler primitives,
but OxCaml exposes normal modules for all these special types so
we can just open them and gain access to the usual integer operations:</p>
<pre><code>module I16 = Stdlib_stable.Int16_u

let[@inline always] i16 x = I16.of_int x
let[@inline always] to_int x = I16.to_int x

let pos : int16# = i16 0
let next : int16# = I16.add pos #1S
</code></pre>
<h3><a href="https://anil.recoil.org/news.xml#unboxed-characters" class="anchor" aria-hidden="true"></a>Unboxed characters</h3>
<p>There's more than just integer operations in OxCaml. Hot off the press in the
past few weeks have been unboxed character operations as well, so we don't need
to use an OCaml int (this is unboxed as well, but I presume the compiler can
optimise and pack 8-bit operations much more effectively if it knows that we're
operating on a char instead of a full word).</p>
<p>The httpz parser tries to use these, but the support for untagged ints <a href="https://github.com/oxcaml/oxcaml/pull/4779">isn't fully
complete yet</a> (thanks <a href="https://thenumb.at/">Max Slater</a> for
the <a href="https://bsky.app/profile/thenumb.at/post/3mdevcomw2k2d">pointer</a>).</p>
<p>HTTP <a href="https://github.com/avsm/oxmono/blob/e0b061c0f6621c80e3a990d02867e3302fd7ce16/avsm/httpz/core/date.ml#L107-L137">date timestamps</a> use unboxed floats as well.</p>
<h3><a href="https://anil.recoil.org/news.xml#returning-unboxed-records-and-tuples" class="anchor" aria-hidden="true"></a>Returning unboxed records and tuples</h3>
<p>Once we've declared these unboxed records, they're fully nestable within other unboxed records.
For example, <a href="https://github.com/avsm/oxmono/blob/e0b061c0f6621c80e3a990d02867e3302fd7ce16/avsm/httpz/core/req.ml#L12-L21">HTTP requests with multiple fields</a> remain unboxed:</p>
<pre><code class="language-ocaml">type request =
  #{ meth : method_
   ; target : span           (* Nested unboxed record *)
   ; version : version
   ; body_off : int16#
   ; content_length : int64#
   ; is_chunked : bool
   ; keep_alive : bool
   ; expect_continue: bool
   }
</code></pre>
<p>Functions can therefore naturally return multiple values without allocation by using unboxed tuples in the return value of a function:</p>
<pre><code class="language-ocaml">let take_while predicate buf ~(pos : int16#) ~(len : int16#)
    : #(span * int16#) =
  let start = pos in
  let mutable p = pos in
  while (* ... *) do p &lt;- I16.add p #1S done;
  #(#{ off = start; len = I16.sub p start }, p)

let #(result_span, new_pos) = take_while is_token buf ~pos ~len
</code></pre>
<p>Vanilla OCaml did some unboxing of this use of tuples, but not with
records (which would land up on the minor heap).  With this OxCaml code,
it's all just passed directly on the stack through function call traces.</p>
<h3><a href="https://anil.recoil.org/news.xml#local-allocations-and-exclaves" class="anchor" aria-hidden="true"></a>Local allocations and exclaves</h3>
<p>We can then also mark parameters to demand that they won't escape a function, enabling stack
allocation more explicitly:</p>
<pre><code class="language-ocaml">(* Buffer is borrowed, won't be stored anywhere *)
let[@inline] equal (local_ buf) (sp : span) (s : string) : bool =
  let sp_len = I16.to_int sp.#len in
  if sp_len &lt;&gt; String.length s then false
  else Bigstring.memcmp_string buf ~pos:(I16.to_int sp.#off) s = 0
</code></pre>
<p>If a function needs to return a local value, then it uses a new <code>exclave_</code> keyword. For example, in the <a href="https://github.com/avsm/oxmono/blob/e0b061c0f6621c80e3a990d02867e3302fd7ce16/avsm/httpz/core/header.mli">HTTP request parsing</a> we look up a stack allocated list of headers:</p>
<pre><code class="language-ocaml">val find : t list @ local -&gt; Name.t -&gt; t option @ local

let rec find_string (buf : bytes) (headers : t list @ local) name = exclave_
  match headers with
  | [] -&gt; None
  | hdr :: rest -&gt;
    let matches =
      match hdr.name with
      | Name.Other -&gt; Span.equal_caseless buf hdr.name_span name
      | known -&gt;
        let canonical = Name.lowercase known in
        String.( = ) (String.lowercase name) canonical
    in
    if matches then Some hdr else find_string buf rest name
;;
</code></pre>
<p>Notice that it's a recursive function as well, so this is a fairly natural way
to write something that remains heap allocated.  You can learn more about this
from <a href="https://gavinleroy.com/">Gavin Gray</a>'s <a href="https://gavinleroy.com/oxcaml-tutorial-icfp25/">OxCaml tutorial slides</a>.</p>
<h2><a href="https://anil.recoil.org/news.xml#mutable-local-variables-with-let-mutable" class="anchor" aria-hidden="true"></a>Mutable Local Variables with "let mutable"</h2>
<p>A nice quality of life improvement is that OxCaml allows stack-allocated
mutable variables in loops, eliminating the need to allocate <code>ref</code> values. This
allows parsing code to have local mutability:</p>
<pre><code class="language-ocaml">let parse_int64 (local_ buf) (sp : span) : int64# =
  let mutable acc : int64# = #0L in
  let mutable i = 0 in
  let mutable valid = true in
  while valid &amp;&amp; i &lt; I16.to_int sp.#len do
    let c = Bytes.get buf (I16.to_int sp.#off + i) in
    match c with
    | '0' .. '9' -&gt;
      acc &lt;- I64.add (I64.mul acc #10L) (I64.of_int (Char.code c - 48));
      i &lt;- i + 1
    | _ -&gt; valid &lt;- false
  done;
  acc
</code></pre>
<p>Whereas in conventional OCaml there might be a minor heap allocation for the
reference:</p>
<pre><code class="language-ocaml">let parse_int64 buf sp =
  let acc = ref 0L in           (* Heap-allocated ref *)
  let i = ref 0 in              (* Heap-allocated ref *)
  let valid = ref true in       (* Heap-allocated ref *)
  while !valid &amp;&amp; !i &lt; sp.len do
    let c = Bytes.get buf (sp.off + !i) in
    match c with
    | '0' .. '9' -&gt;
      acc := Int64.add (Int64.mul !acc 10L) (Int64.of_int (Char.code c - 48));
      i := !i + 1
    | _ -&gt; valid := false
  done;
  !acc
</code></pre>
<h3><a href="https://anil.recoil.org/news.xml#putting-the-parser-together" class="anchor" aria-hidden="true"></a>Putting the parser together</h3>
<p>The toplevel <a href="https://github.com/avsm/oxmono/blob/e0b061c0f6621c80e3a990d02867e3302fd7ce16/avsm/httpz/core/httpz.mli#L182">Httpz.parse function</a> has a pretty simple signature from a user's perspective:</p>
<pre><code>val parse : bytes -&gt; len:int16# -&gt; limits:limits -&gt;
  #(Buf_read.status * Req.t * Header.t list) @ local
</code></pre>
<p>This function receives some a bytebuffer and resource limits and returns an unboxed local tuple of the connection status, parsed (unboxed) request and a stack-local list of header spans that represent the offsets within the input buffer of what was passed.</p>
<p>I should probably make the input buffer local too; one nice aspect of OxCaml is how easy it is to incrementally add type and kind annotations and lean on the compiler type inference to help guide where to fixup callsites.</p>
<h3><a href="https://anil.recoil.org/news.xml#caveats-and-limitations" class="anchor" aria-hidden="true"></a>Caveats and limitations</h3>
<p>There are lots and lots of other new features in OxCaml which I've started integrating, but require careful planning of layouts.
For example, I wanted to use <a href="https://oxcaml.org/documentation/unboxed-types/02-or-null/">or_null</a> to have a non-allocating
version of option, but you often end up with long compiler errors about value inference failures, so I ended up just allocating
a local type instead. Something to investigate more in the future as I get familiar with OxCaml.</p>
<p>I also ran into issues using mutable fields in unboxed records and found this is <a href="https://oxcaml.org/documentation/unboxed-types/01-intro/">documented</a>:</p>
<blockquote>
<p>We plan to allow mutating unboxed records within boxed records (the design
will differ from boxed record mutability, as unboxed types don’t have the
same notion of identity).</p>
</blockquote>
<p>It's also difficult right now to strip away the OxCaml extensions and go back
to normal OCaml syntax. <a href="https://tyconmismatch.com/code.html">Chris Casinghino</a> pointed me to the OxCaml ocamlformat fork which
has a <code>--erase-jane-syntax</code>, but it requires some build system work to
integrate and seems to lag a little behind the new features (like unboxed small
literals). For now, I've decided to just focus on using OxCaml exclusively and
see how it goes for a while.</p>
<p>Finally, the tooling is still a fluid story. <a href="https://github.com/art-w">Arthur Wendling</a> and <a href="https://jon.recoil.org">Jon Ludlam</a> are making
fast progress on getting <a href="https://github.com/ocaml/odoc/pull/1399">odoc working</a> in the
mainline tool, but it's not quite there today.</p>
<h3><a href="https://anil.recoil.org/news.xml#claude-skills-for-oxcaml" class="anchor" aria-hidden="true"></a>Claude skills for OxCaml</h3>
<p>While I built small scale examples to test out the architecture, I leaned heavily
on Claude code to build out the majority of the parser so I could rapidly experiment.
To do this, I synthesised a set of <a href="https://github.com/avsm/ocaml-claude-marketplace/tree/main/plugins/ocaml-dev/skills/oxcaml">OxCaml specific Claude skills</a>
in my <a href="https://anil.recoil.org/notes/aoah-2025-25">Claude OCaml marketplace</a> which you can add to your own projects as well. Browsing the skills is a pretty nice way of getting familiar with the different features.</p>
<p>I generated those skills via a combination of summarising the OxCaml source trees and cribbing from the <a href="https://anil.recoil.org/notes/icfp25-oxcaml">ICFP 2025 tutorial</a>, and then getting CC to verify that the example code actually compiled. All automated and very easy to refresh every time a new compiler drops from Jane Street.</p>
<p><img src="https://anil.recoil.org/images/claude-oxlocal-1.webp" alt="%c" title="The OxCaml compiler errors are really descriptive in the latest drop, which greatly helps coding agents figure out the new types"></p>
<h2><a href="https://anil.recoil.org/news.xml#performance-results" class="anchor" aria-hidden="true"></a>Performance Results</h2>
<p>Ultimately, none of this matters if the  runtime performance isn't there!
Luckily, the HTTPz parser is incredible in a synthetic benchmark (just passing
buffers around) as opposed to a network benchmark, using Core_bench to measure
performance. What's impressive isn't the straightline throughput, but the
massive drop in heap activity which greatly increased the predictability and
tail latency of the service. And with all the extra typing information, I
expect that straightline performance will only increase (and this is before
I've looked at the <a href="https://oxcaml.org/documentation/simd/intro/">SIMD
support</a>).</p>
<div role="region"><table>
<tbody><tr>
<th>Metric</th>
<th>httpz (OxCaml)</th>
<th>Traditional Parser</th>
</tr>
<tr>
<td>Small request (35B)</td>
<td>154 ns</td>
<td>300+ ns</td>
</tr>
<tr>
<td>Medium request (439B)</td>
<td>1,150 ns</td>
<td>2,000+ ns</td>
</tr>
<tr>
<td>Heap allocations</td>
<td>0</td>
<td>100-800 words</td>
</tr>
<tr>
<td>Throughput</td>
<td>6.5M req/sec</td>
<td>3M req/sec</td>
</tr>
</tbody></table></div><h2><a href="https://anil.recoil.org/news.xml#putting-my-new-site-live" class="anchor" aria-hidden="true"></a>Putting my new site live</h2>
<p>I then glued this together using Eio into a <a href="https://github.com/avsm/oxmono/blob/e0b061c0f6621c80e3a990d02867e3302fd7ce16/avsm/httpz/eio/httpz_eio.mli">full
webserver</a>.
It works, and serves traffic just fine and in fact you are reading this web page via it right now!</p>
<h3><a href="https://anil.recoil.org/news.xml#what-next-caml_alloc_local-for-c-bindings" class="anchor" aria-hidden="true"></a>What next: caml_alloc_local for C bindings</h3>
<p>The current Eio/OxCaml does a data copy right now since Eio uses Bigarray, but I had a catchup coffee
with <a href="https://github.com/https://roscidus.com">Thomas Leonard</a> and <a href="https://patrick.sirref.org">Patrick Ferris</a> where I agreed to treesmash my local eio into
switching entirely to bytes from the io-uring layer up. <a href="https://toao.com">Sadiq Jaffer</a> informs me
that his compactor doesn't trigger automatically, so any bytes above a 4KB
threshold are allocated using mmap and so are fine to pass to the kernel for
zero copy receive.</p>
<p>The key OxCaml feature to make this <code>io_uring</code> integration awesome is a new FFI
function that allocates an OCaml value directly into the caller's OxCaml stack
rather than the heap. This means that we <em>should</em> be able to come up with a scheme
by which io_uring requests are routed directly to an OCaml continuation that's woken
up directly with a buffer available to it on the stack. True zero-copy to the kernel
awaits, which should also help speed up <a href="https://anil.recoil.org/papers/2025-docker-icfp">Docker's VPNKit</a> hugely
as well.</p>
<h3><a href="https://anil.recoil.org/news.xml#making-it-easier-to-develop-in-oxcaml-in-the-open" class="anchor" aria-hidden="true"></a>Making it easier to develop in OxCaml in the open</h3>
<p>Keen readers may note that my OxCaml repo links here go to a new <a href="https://github.com/avsm/oxmono">monorepo</a> I've
setup for the purpose of hacking on real code in production outside of Jane Street's
walls.</p>
<p>I'll blog more about this next week, but for now I hope you've enjoyed a little
taste of what the OxCaml extensions offer in real world code.  Stay tuned also for
even more performance improvements, and for native TLS with an OxCaml port of
<a href="https://github.com/mirleft/ocaml-tls">ocaml-tls</a> from <a href="https://github.com/hannesm">Hannes Mehnert</a> soon!</p><h1>References</h1><ul><li>Madhavapeddy et al (2025). Functional Networking for Millions of Docker Desktops. <a href="https://doi.org/10.1145/3747525" target="_blank"><i>10.1145/3747525</i></a></li>
<li>Madhavapeddy (2025). Holding an OxCaml tutorial at ICFP/SPLASH 2025. <a href="https://doi.org/10.59350/55bc5-x4p75" target="_blank"><i>10.59350/55bc5-x4p75</i></a></li>
<li>Sivaramakrishnan et al (2021). Retrofitting effect handlers onto OCaml. ACM. <a href="https://doi.org/10.1145/3453483.3454039" target="_blank"><i>10.1145/3453483.3454039</i></a></li>
<li>Madhavapeddy (2025). Arise Bushel, my sixth generation oxidised website. <a href="https://doi.org/10.59350/0r62w-c8g63" target="_blank"><i>10.59350/0r62w-c8g63</i></a></li>
<li>Madhavapeddy (2025). GeoTessera Python library released for geospatial embeddings. <a href="https://doi.org/10.59350/7hy6m-1rq76" target="_blank"><i>10.59350/7hy6m-1rq76</i></a></li></ul>
