---
title: Building OCaml from assembly
description:
url: https://lambdafoo.com/posts/2024-08-30-building-ocaml-from-assembly.html
date: 2024-08-30T00:00:00-00:00
preview_image:
authors:
- Tim McGilchrist
source:
ignore:
---

<div class="post">
  <h1 class="post-title">Building OCaml from assembly</h1>
  <span class="post-date">August 30, 2024</span>
  <p>At work I’ve been focusing on improving the debugging experience with OCaml.
As part of that I’ve discovered how some of the pieces fit together, that might
be obvious in retrospect, but are interesting to at least me so I’m going to
post details about them here.</p>
<p>The first nugget is you can hand compile an OCaml program into a final executable.
What do I mean? You can ask the OCaml compiler to output all the assembly generated
that goes into a library or executable. Then take that an call the assembler yourself
to build it. First lets review how the compiler works.</p>
<h2>Compilation Pipeline</h2>
<p>Here is a <em>grossly</em> simplified overview of the OCaml compiler. We feed in OCaml source code
in the form of ml/mli files, which flow through each stage and eventually end up
being emitted as either object files or textual assembly files. The first step from
OCaml Source to Parse Tree uses <a href="https://gallium.inria.fr/~fpottier/menhir/">menhir</a> to parse
and generate an untyped <a href="https://en.wikipedia.org/wiki/Abstract_syntax_tree">AST</a> representing
the code in the source file. This is then type
checked into a typed tree, this is where the type theory happens. After that, there are some stages
where the typed tree is transformed into representations more suitable for generating assembly.
The final stage traverses the CMM/Linear AST generating assembly code for a specific
family of CPUs (like x86_64 or ARM64).</p>
<pre><code>                                      
 ┌──────────────┐   ┌──────────────┐  
 │ OCaml Source │   │  Parse Tree  │  
 │              ┼───►              │  
 └──────────────┘   └──────┬───────┘  
                           │          
 ┌──────────────┐   ┌──────▼───────┐  
 │    Lambda    │   │  Typed Tree  │  
 │              ◄───┼              │  
 └──────┬───────┘   └──────────────┘  
        │                             
 ┌──────▼───────┐   ┌──────────────┐  
 │  CMM/Linear  │   │    Emit      │  
 │              ┼───►   Assembly   │  
 └──────────────┘   └──────────────┘  
                                      </code></pre>
<p>Finally, this assembly is compiled by the system C compiler to produce object files or
executables to be run. So we could treat the OCaml compiler as a <em>fancy</em> way to
just generate assembly files, which we can then mess with to do things like add <a href="https://dwarfstd.org">DWARF
information</a> or optimise assembly routines, or just for pure fun.</p>
<h2>OCaml source</h2>
<p>Starting with an OCaml program taken from <a href="https://doi.org/10.1145/3453483.3454039">Retrofitting Effect Handlers onto OCaml</a>. This program doesn’t compute anything interesting but it does show how OCaml’s FFI to C works and how to pass control between the two. So it is interesting for what it does.</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb2-1" aria-hidden="true" tabindex="-1"></a>$ cat meander.ml</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-2" aria-hidden="true" tabindex="-1"></a><span class="kw">external</span> ocaml_to_c</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-3" aria-hidden="true" tabindex="-1"></a>         : <span class="dt">unit</span> -&gt; <span class="dt">int</span> = <span class="st">"ocaml_to_c"</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-4" aria-hidden="true" tabindex="-1"></a><span class="kw">exception</span> E1</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-5" aria-hidden="true" tabindex="-1"></a><span class="kw">exception</span> E2</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-6" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> c_to_ocaml () = <span class="dt">raise</span> E1</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-7" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> _ = <span class="dt">Callback</span>.register</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-8" aria-hidden="true" tabindex="-1"></a>          <span class="st">"c_to_ocaml"</span> c_to_ocaml</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-9" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> omain () =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-10" aria-hidden="true" tabindex="-1"></a>  <span class="kw">try</span> <span class="co">(* h1 *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-11" aria-hidden="true" tabindex="-1"></a>    <span class="kw">try</span> <span class="co">(* h2 *)</span> ocaml_to_c ()</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-12" aria-hidden="true" tabindex="-1"></a>    <span class="kw">with</span> E2 -&gt; <span class="dv">0</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-13" aria-hidden="true" tabindex="-1"></a>  <span class="kw">with</span> E1 -&gt; <span class="dv">42</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-14" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> _ = <span class="kw">assert</span> (omain () = <span class="dv">42</span>)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-15" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-16" aria-hidden="true" tabindex="-1"></a>$ cat meander_c.c</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-17" aria-hidden="true" tabindex="-1"></a><span class="ot">#include &lt;caml/mlvalues.h&gt;</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-18" aria-hidden="true" tabindex="-1"></a><span class="ot">#include &lt;caml/callback.h&gt;</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-19" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-20" aria-hidden="true" tabindex="-1"></a>value ocaml_to_c (value <span class="dt">unit</span>) {</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-21" aria-hidden="true" tabindex="-1"></a>    caml_callback<span class="co">(*caml_named_value</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-22" aria-hidden="true" tabindex="-1"></a><span class="co">                  ("c_to_ocaml"), Val_unit);</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-23" aria-hidden="true" tabindex="-1"></a><span class="co">    return Val_int(0);</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-24" aria-hidden="true" tabindex="-1"></a><span class="co">}</span></span></code></pre></div>
<p>Reading from the bottom of the file, <code>meander.ml</code> asserts that the function <code>omain</code>
returns the value <code>42</code>. It gets that value by calling <code>ocaml_to_c</code> which is actually an
external C function defined in <code>meander_c.c</code>, imported into OCaml using
<code>external</code> in the first line of <code>meander.ml</code>. The C function calls back into
OCaml using <code>caml_callback</code> which executes the <code>c_to_ocaml</code> function. An exception is
raised, unwinding everything back to <code>omain</code> with it’s try/with blocks.</p>
<p>To compile this program we use the OCaml 5.2 compiler.</p>
<pre class="shell"><code>$ ocamlopt --version
5.2.0
$ ocamlopt meander_c.c meander.ml -o meander.exe
$ ./meander.exe
$ echo $?
0</code></pre>
<p>Running the program under macOS gives a successful exit code, so it must have got
<code>42</code> and the assertion passed. Try changing the value 42 to something else to check.</p>
<p>Next we will pull apart what the compiler is doing to generate the final
executable. Run <code>ocamlopt</code> with these flags:</p>
<pre class="shell"><code> $ ocamlopt meander_c.c meander.ml -o meander.exe -S -g -dstartup -verbose

+ cc  -O2 -fno-strict-aliasing -fwrapv -pthread -pthread  -D_FILE_OFFSET_BITS=64 -c -g -I'/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml' 'meander_c.c'
+ cc -c -Wno-trigraphs  -o 'meander.o' 'meander.s'
+ cc -c -Wno-trigraphs  -o '/var/folders/z_/7yzlrkjn6pd441zs1qhzpjv00000gn/T/camlstartup9b503b.o' 'meander.exe.startup.s'
+ cc -O2 -fno-strict-aliasing -fwrapv -pthread  -pthread   -o 'meander.exe'  '-L/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml'  '/var/folders/z_/7yzlrkjn6pd441zs1qhzpjv00000gn/T/camlstartup9b503b.o' '/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml/std_exit.o' 'meander.o' '/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml/stdlib.a' 'meander_c.o' '/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml/libasmrun.a'     -lpthread</code></pre>
<p>Focusing on the <code>ocamlopt</code> command, the flag <code>-S</code> asks the compiler to generate the assembly
files for the OCaml source, <code>-g</code> asks for debug information to be included, <code>-dstartup</code>
generates the startup file that bridges between the C startup and OCaml (more on that later)
and <code>-verbose</code> tells <code>ocamlopt</code> to print out what commands it’s running.</p>
<p>So, what has been printed out? The first line is compiling the <code>meander_c.c</code> file into
an object file, the <code>meander_c.o</code> file in the current directory. Then we have a <code>meander.s</code>
file being compiled (assembled) into another object file. This is the output of compiling
the <code>meander.ml</code> OCaml source into assembly. The <code>--verbose</code> option doesn’t show how that
file gets created. The third line is compiling the startup file from <code>meander.exe.startup.s</code>
into another object file. The final step is calling the linker via <code>cc</code> to generate the final
<code>meander.exe</code> file. You can see all the object files from previous steps plus the OCaml stdlib
<code>_opam/lib/ocaml/stdlib.a</code> and <code>_opam/lib/ocaml/std_exit.o</code> from the local opam switch
plus the OCaml libraries being added to the search path as
<code>-L/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml</code>. It is not that dissimilar to building a
C program.</p>
<p>What about those assembly files? The <code>meander.s</code> is our ARM64 assembly file for <code>meander.ml</code>
open it up and search for <code>entry</code>. If you’re on Linux or another architecture like x86_64
the assembly will be different but the names will be the same. This is the entry point
called when executing the program, the OCaml runtime jumps to the symbol <code>_camlMeander.entry</code>.</p>
<pre class="assembly"><code>	.globl	_camlMeander.entry
L114:
	mov	x16, #34
	stp	x16, x30, [sp, #-16]!
	bl	_caml_call_realloc_stack
	ldp	x16, x30, [sp], #16
_camlMeander.entry:
	.cfi_startproc
	ldr	x16, [x28, #40]
	add	x16, x16, #328
	cmp	sp, x16
	bcc	L114
	sub	sp, sp, #16
	.cfi_adjust_cfa_offset	16
	.cfi_offset 30, -8
	str	x30, [sp, #8]</code></pre>
<p>Search for other symbols like <code>omain</code> and <code>c_to_ocaml</code></p>
<pre class="assembly"><code>	.globl	_camlMeander.omain_278
_camlMeander.omain_278:
	.loc	1	8
	.cfi_startproc
	sub	sp, sp, #16
	.cfi_adjust_cfa_offset	16
	.cfi_offset 30, -8
	str	x30, [sp, #8]
....
_camlMeander.c_to_ocaml_273:
	.file	1	"meander.ml"
	.loc	1	5
	.cfi_startproc
	sub	sp, sp, #16
	.cfi_adjust_cfa_offset	16
	.cfi_offset 30, -8
	str	x30, [sp, #8]</code></pre>
<p>All the code is there, we just need to assemble it. On my machine (macOS ARM64) running this
command will give me an executable <code>meander.exe</code> without even using <code>ocamlopt</code>.</p>
<pre class="shell"><code>$ gcc -O2 -fno-strict-aliasing -fwrapv -pthread -D_FILE_OFFSET_BITS=64 \
      -c -g -I'/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml' 'meander_c.c'
$ gcc -c -Wno-trigraphs -o 'meander.o' 'meander.s'
$ gcc -c -Wno-trigraphs -o meanderCamlStartup.o meander.exe.startup.s
$ gcc -o 'meander.exe' '-L/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml' 'meanderCamlStartup.o' \
       '/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml/std_exit.o' 'meander.o' \
       '/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml/stdlib.a' 'meander_c.o' \
       '/Users/tsmc/code/ocaml/owee/_opam/lib/ocaml/libasmrun.a' -lpthread</code></pre>
<p>Try it out, you’ll need to change <code>/Users/tsmc/code/ocaml/owee/_opam</code> to your local directory with
a local opam switch for OCaml 5.2.</p>
<h2>Startup file</h2>
<p>What about that startup file? <code>meander.exe.startup.s</code> What is that for?
Open the file and search for <code>_caml_program</code>, this is the entry point called by the
startup code written in C.</p>
<pre class="shell"><code>_caml_program:
	.cfi_startproc
	ldr	x16, [x28, #40]
	add	x16, x16, #328
	cmp	sp, x16
	bcc	L136
	sub	sp, sp, #16
	.cfi_adjust_cfa_offset	16
	.cfi_offset 30, -8
	str	x30, [sp, #8]
L135:
	bl	_camlCamlinternalFormatBasics$entry
L137:
	adrp	x0, _caml_globals_inited@GOTPAGE
	ldr	x0, [x0, _caml_globals_inited@GOTPAGEOFF]
	ldr	x2, [x0, #0]
	add	x3, x2, #1
	dmb	ishld
	str	x3, [x0, #0]
	bl	_camlStdlib$entry</code></pre>
<p>The code is responsible for calling the <code>entry</code> initialisation function for all
imported modules. In <code>meander.ml</code> we only include a couple of functions from the
standard library so we have <code>_camlStdlib$entry</code>, <code>_camlStdlib__Sys$entry</code> etc then
we finally call <code>_camlMeander$entry</code> which we saw earlier.</p>
<p>We need this assembly file to generate an object file for linking into the final executable.
If not the linker won’t have <code>_caml_program</code> symbol available and none of the OCaml Stdlib will
be initialised. A fun exercise is to re-write this file to not call all those <code>entry</code> functions
but still provide <code>_caml_program</code> and call into <code>_camlMeander$entry</code>.</p>
<p>I made small <a href="https://github.com/ocaml/ocaml/pull/13217">PR #13217</a> to improve this behaviour
to loop over a table of functions to call rather than generating large slabs of identical code.</p>
<h2>Bonus</h2>
<p>Now you we don’t need the OCaml compiler to write OCaml.</p>
<p>But seriously the purpose for discovering this was to investigate adding DWARF debugging
information to OCaml on macOS. That’s a different topic for next time.</p>
</div>

