---
title: Getting Started with LLDB on OCaml
description:
url: https://lambdafoo.com/posts/2024-08-03-lldb-ocaml.html
date: 2024-08-03T00:00:00-00:00
preview_image:
authors:
- Tim McGilchrist
source:
ignore:
---

<div class="post">
  <h1 class="post-title">Getting Started with LLDB on OCaml</h1>
  <span class="post-date">August  3, 2024</span>
  <p>This post is a companion to KC’s excellent <a href="https://kcsrk.info/ocaml/gdb/2024/01/20/gdb-ocaml/">Getting Started with GDB on OCaml</a> that shows how to debug OCaml programs with GDB. I wanted to demonstrate the same functionality using LLDB on Linux ARM64. The aim is to show the beginnings of debugging OCaml programs with LLDB and highlight a few LLDB tricks I’ve found.</p>
<p>We will start with the same program:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb1-1" aria-hidden="true" tabindex="-1"></a><span class="co">(* fib.ml *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-2" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> <span class="kw">rec</span> fib n =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-3" aria-hidden="true" tabindex="-1"></a>  <span class="kw">if</span> n = <span class="dv">0</span> <span class="kw">then</span> <span class="dv">0</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-4" aria-hidden="true" tabindex="-1"></a>  <span class="kw">else</span> <span class="kw">if</span> n = <span class="dv">1</span> <span class="kw">then</span> <span class="dv">1</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-5" aria-hidden="true" tabindex="-1"></a>  <span class="kw">else</span> fib (n<span class="dv">-1</span>) + fib (n<span class="dv">-2</span>)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-6" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-7" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> main () =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-8" aria-hidden="true" tabindex="-1"></a>  <span class="kw">let</span> r = fib <span class="dv">20</span> <span class="kw">in</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-9" aria-hidden="true" tabindex="-1"></a>  <span class="dt">Printf</span>.printf <span class="st">"fib(20) = %d</span><span class="ch">\n</span><span class="st">"</span> r</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-10" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-11" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> _ = main ()</span></code></pre></div>
<p>Compiled with OCaml version 5.2.0.</p>
<pre class="shell"><code>$ ocamlopt --version
5.2.0
$ ocamlopt -g -o fib.exe fib.ml
$ ./fib.exe 20
fib(20) = 6765</code></pre>
<p>The program prints the 20th Fibonacci number, nothing special but interesting because it has recursion. Now start up an lldb session.</p>
<pre class="shell"><code>$ lldb ./fib.exe</code></pre>
<h2>Setting breakpoints</h2>
<p>We want to set breakpoints in the <code>fib</code> function. The first way to set breakpoints is based on OCaml function names, due to a process called name mangling, they look slightly different in the executable. Since we don’t know the exact names we can use tab completion to help us.</p>
<pre class="shell"><code>(lldb) br s -n camlFib.fib_ # press tab to show the possible matches
(lldb) br s -n camlFib.fib_270 # There is only one matching ending 270
Breakpoint 1: where = fib.exe`camlFib.fib_270 + 76, address = 0x0000000000051084</code></pre>
<p>You can also set break points using lldb’s file name and number combination. This time we will set a breakpoint in the <code>main</code> function, which starts at line 6 in <code>fib.ml</code>.</p>
<pre class="shell"><code>(lldb) br s -f fib.ml -l 6
Breakpoint 2: where = fib.exe`camlFib.main_271, address = 0x0000000000050f48
(lldb)</code></pre>
<p>Now we can run the program.</p>
<pre class="shell"><code>Breakpoint 2: where = fib.exe`camlFib.main_272, address = 0x00000000000510c8
(lldb) run
Process 11987 launched: '/home/tsmc/projects/fib.exe' (aarch64)
Process 11987 stopped
* thread #1, name = 'fib.exe', stop reason = breakpoint 2.1
    frame #0: 0x0000aaaaaaaf10c8 fib.exe`camlFib.main_272 at fib.ml:7
   4   	  else if n = 1 then 1
   5   	  else fib (n-1) + fib (n-2)
   6   	
-&gt; 7   	let main () =
   8   	  let r = fib 20 in
   9   	  Printf.printf "fib(20) = %d\n" r
   10  	</code></pre>
<p>The program execution starts in the lldb session and we stop at the breakpoint at <code>main</code>. LLDB has a terminal UI mode for stepping through the file. This can be started up typing <code>gui</code> into the <code>lldb</code> prompt, it should look similar to this.</p>
<figure>
<img src="https://lambdafoo.com/images/lldb-aarch64-fib.png" alt="Terminal image of lldb running fib.ml showing gui">
<figcaption aria-hidden="true">Terminal image of lldb running fib.ml showing gui</figcaption>
</figure>
<p>Note that we can see both breakpoints highlighted on the line numbers, the backtrace of how we got here and the current line is highlighted. Use <code>Esc</code> to exit the terminal UI mode and go back to the lldb prompt. We will use the lldb prompt for the rest of the post.</p>
<h2>Examining the stack</h2>
<p>You can step through the OCaml program with lldb commands <code>n</code> and <code>s</code>. After a few <code>n</code>’s, examine the backtrace using the <code>bt</code> command.</p>
<pre class="shell"><code>(lldb) bt
* thread #1, name = 'fib.exe', stop reason = breakpoint 1.1
  * frame #0: 0x0000aaaaaaaf1084 fib.exe`camlFib.fib_270 at fib.ml:5
    frame #1: 0x0000aaaaaaaf108c fib.exe`camlFib.fib_270 at fib.ml:5
    frame #2: 0x0000aaaaaaaf108c fib.exe`camlFib.fib_270 at fib.ml:5
    frame #3: 0x0000aaaaaaaf108c fib.exe`camlFib.fib_270 at fib.ml:5
    frame #4: 0x0000aaaaaaaf10f4 fib.exe`camlFib.main_272 at fib.ml:8
    frame #5: 0x0000aaaaaaaf11bc fib.exe`camlFib.entry at fib.ml:11
    frame #6: 0x0000aaaaaaaee684 fib.exe`caml_program + 476
    frame #7: 0x0000aaaaaab46b48 fib.exe`caml_start_program + 132
    frame #8: 0x0000aaaaaab46640 fib.exe`caml_main [inlined] caml_startup(argv=&lt;unavailable&gt;) at startup_nat.c:145:7
    frame #9: 0x0000aaaaaab4663c fib.exe`caml_main(argv=&lt;unavailable&gt;) at startup_nat.c:151:3
    frame #10: 0x0000aaaaaaaee310 fib.exe`main(argc=&lt;unavailable&gt;, argv=&lt;unavailable&gt;) at main.c:37:3
    frame #11: 0x0000fffff7d784c4 libc.so.6`__libc_start_call_main(main=(fib.exe`main at main.c:31:1), argc=1, argv=0x0000fffffffffb58) at libc_start_call_main.h:58:16
    frame #12: 0x0000fffff7d78598 libc.so.6`__libc_start_main_impl(main=0x0000aaaaaaba0de0, argc=16, argv=0x000000000000000f, init=&lt;unavailable&gt;, fini=&lt;unavailable&gt;, rtld_fini=&lt;unavailable&gt;, stack_end=&lt;unavailable&gt;) at libc-start.c:360:3
    frame #13: 0x0000aaaaaaaee3b0 fib.exe`_start + 48</code></pre>
<p>You can see the backtrace includes the recursive calls to <code>fib</code> function, the <code>main</code> function in <code>fib.ml</code>, followed by some assembly functions and a number of functions from the OCaml runtime. In between frame #8 and #5 is where the runtime, written in C, switches into assembly to setup the environment to execute the OCaml program. Then we actually enter the OCaml program at frame #5 via <code>camlFib.entry</code>. This function calls initialisation functions for the program and any dependencies like Stdlib that get used.</p>
<h2>Examining values</h2>
<p>The support for examining OCaml values in LLDB, as you would for say C, is a bit lacking. Not enough information is being emitted by the OCaml compiler to do this yet. So we need to understand how OCaml represents values at runtime and what the OCaml calling conventions are. First we will look at examining values.</p>
<p>Here we are on ARM64 so our registers are named <code>x0-x30</code> with <code>sp</code> representing the stack pointer.
The first <a href="https://github.com/ocaml/ocaml/blob/5.2.0/asmcomp/arm64/proc.ml#L168-L172">16 arguments are passed in registers</a>, starting from register x0. So the arguments to the <code>fib</code> function should be in the <code>x0</code> register. We also know that the argument to fib is an integer. OCaml uses 63-bit tagged integers (on 64-bit machines) with the least-significant bit is 1. Given a machine word or a register holding an OCaml integer, the integer value is obtained by right shifting the value by 1.</p>
<p>Putting that all together, we can examine the arguments to <code>fib</code> at the breakpoint in <code>fib</code> like so.</p>
<pre class="shell"><code>(lldb) p $x0 &gt;&gt; 1
(unsigned long) 5</code></pre>
<p>Given we have a recursive fib function this printing corresponds to <code>fib(5)</code>. Have a go at moving up and down the recursive fib calls using <code>up</code> or <code>down</code> and print out the arguments. You can also examine the evaluation order of arguments in <code>fib</code>, noting that the evaluation order of arguments in OCaml is unspecified but 5.2.0 evaluates right-to-left.</p>
<h2>Advanced printing</h2>
<p>Examining values using bit shifting is tedious. We can do better by writing our own printing functions in Python. The OCaml compiler distribution comes with some scripts to make examining OCaml values in LLDB easier. Note they have historically been used by OCaml maintainers to develop the compiler, so they might be a little rough or missing features (PRs to improve this situation are welcome). With that lets see what we can do.</p>
<p>Since we are using OCaml 5.2.0, we need to get that source code.</p>
<pre class="shell"><code># I'm working within ~/projects directory on my machine
$ git clone https://github.com/ocaml/ocaml --branch 5.2.0</code></pre>
<p>Startup a new lldb session, load the lldb script, and get to a breakpoint in the recursive fib calls</p>
<pre class="shell"><code>lldb ./fib.exe
(lldb) command script import ../ocaml/tools/lldb.py
(lldb) br s -f fib.ml -l 1
Process 12014 launched: '/home/tsmc/projects/fib.exe' (aarch64)
Process 12014 stopped
* thread #1, name = 'fib.exe', stop reason = breakpoint 4.1
    frame #0: 0x0000aaaaaaaf1038 fib.exe`camlFib.fib_270 at fib.ml:2
   1   	(* fib.ml *)
-&gt; 2   	let rec fib n =
   3   	  if n = 0 then 0
   4   	  else if n = 1 then 1
   5   	  else fib (n-1) + fib (n-2)
   6   	
   7   	let main () =
</code></pre>
<p>As earlier, the first argument is in <code>x0</code> register. We can examine the value now with the python script.</p>
<pre class="shell"><code>(lldb) p (value)$x0
(value) 41 caml:20</code></pre>
<p><code>value</code> is the type of OCaml values defined in the OCaml runtime. The script <code>tools/lldb.py</code> installs a pretty printer for the values of type <code>value</code>. Here is pretty prints the first argument which is <code>20</code></p>
<p>We can also print other kinds of OCaml values. Create this file with some interesting OCaml values:</p>
<pre class="shell"><code>$ cat test_blocks.ml
(* test_blocks.ml *)

type t = {s : string; i : int}

let main a b =
  print_endline "Hello, world!";
  print_endline a;
  print_endline b.s

let _ = main "foo" {s = "bar"; i = 42}</code></pre>
<p>Now we need to compile it, start an lldb session and break on the main function.</p>
<pre class="shell"><code>$ ocamlopt -g -o test_blocks.exe test_blocks.ml
$ lldb ./test_blocks.exe
(lldb) target create "./test_blocks.exe"
Current executable set to '/home/tsmc/projects/test_blocks.exe' (aarch64).
(lldb) command script import ../ocaml/tools/lldb.py
OCaml support module loaded. Values of type 'value' will now
print as OCaml values, and an 'ocaml' command is available for
heap exploration (see 'help ocaml' for more information).
(lldb) br s -n camlTest_blocks.main_273
Breakpoint 1: where = test_blocks.exe`camlTest_blocks.main_273 + 40, address = 0x0000000000019ab0
(lldb) run
Process 12043 launched: '/home/tsmc/projects/test_blocks.exe' (aarch64)
Process 12043 stopped
* thread #1, name = 'test_blocks.exe', stop reason = breakpoint 1.1
    frame #0: 0x0000aaaaaaab9ab0 test_blocks.exe`camlTest_blocks.main_273 at test_blocks.ml:4
   1   	type t = {s : string; i : int}
   2   	
   3   	let main a b =
-&gt; 4   	  print_endline "Hello, world!";
   5   	  print_endline a;
   6   	  print_endline b.s
   7   	
(lldb)</code></pre>
<p>Let’s examine the two arguments to main</p>
<pre class="shell"><code>(lldb) p (value)$x0
(value) 187649984891864 caml(-):'Hello, world!'&lt;13&gt;
(lldb) p (value)$x1
(value) 187649984891808 caml(-):('bar', 42)</code></pre>
<p>What is going on here, didn’t we say the first argument is in <code>x0</code>? What has happened here is our breakpoint has been set a little after we have entered the function and the original value for <code>x0</code> has been stored on the stack and <code>x0</code> register has been reused to store arguments to <code>print_endline "Hello, world!";</code>. The second argument in <code>x1</code> is as expected.</p>
<p>To find the original <code>x0</code> value we need to look at assembly (don’t worry too much about the specifics of ARM assembly).</p>
<pre class="shell"><code>(lldb) dis
test_blocks.exe`camlTest_blocks.main_273:
    0xaaaaaaab9a88 &lt;+0&gt;:  ldr    x16, [x28, #0x28]
    0xaaaaaaab9a8c &lt;+4&gt;:  add    x16, x16, #0x158
    0xaaaaaaab9a90 &lt;+8&gt;:  cmp    sp, x16
    0xaaaaaaab9a94 &lt;+12&gt;: b.lo   0xaaaaaaab9a78 ; camlStd_exit.code_end
    0xaaaaaaab9a98 &lt;+16&gt;: sub    sp, sp, #0x20
    0xaaaaaaab9a9c &lt;+20&gt;: str    x30, [sp, #0x18]
    0xaaaaaaab9aa0 &lt;+24&gt;: str    x0, [sp]
    0xaaaaaaab9aa4 &lt;+28&gt;: str    x1, [sp, #0x8]
(lldb) reg r sp
      sp = 0x0000aaaaaab3d160
(lldb) memory read -s8 -fx -l2 0x0000aaaaaab3d160
0xaaaaaab3d160: 0x0000aaaaaab10bc8 0x0000aaaaaab10ba0
0xaaaaaab3d170: 0x0000fffffffff8e0 0x0000aaaaaaab9b38
0xaaaaaab3d180: 0x0000000000000000 0x0000aaaaaaab94fc
0xaaaaaab3d190: 0x0000000000000000 0x0000aaaaaaae05c8
(lldb) p (value)0x0000aaaaaab10bc8
(value) 187649984891848 caml(-):'foo'&lt;3&gt;</code></pre>
<p>The disassembled code is the function prologue code, which is saving <code>x0</code> onto the stack using <code>str x0, [sp]</code>. To get the original value for <code>x0</code> we read sp (Stack Pointer), retrieve the data at that address and then print it using <code>value</code>. We get back to our argument passed to main, which was <code>foo</code> and can confirm that by looking at the source code.</p>
<h2>Extras</h2>
<p>A few useful extras for debugging OCaml programs.</p>
<p>You can set breakpoints based on addresses, this is useful when you know a specific instruction you want to break on. From the previous session, set a breakpoint on the <code>sub sp, sp, #0x20</code> address.</p>
<pre class="shell"><code>(lldb) br s -a 0xaaaaaaab9a98
Breakpoint 7: where = test_blocks.exe`camlTest_blocks.main_273 + 16, address = 0x0000aaaaaaab9a98
(lldb) run
There is a running process, kill it and restart?: [Y/n] y
Process 12070 exited with status = 9 (0x00000009) killed
Process 12078 launched: '/home/tsmc/projects/test_blocks.exe' (aarch64)
Process 12078 stopped
* thread #1, name = 'test_blocks.exe', stop reason = breakpoint 7.1
    frame #0: 0x0000aaaaaaab9a98 test_blocks.exe`camlTest_blocks.main_273 at test_blocks.ml:3
   1   	type t = {s : string; i : int}
   2   	
-&gt; 3   	let main a b =
   4   	  print_endline "Hello, world!";
   5   	  print_endline a;
   6   	  print_endline b.s
   7   	
(lldb) p (value)$x0
(value) 187649984891848 caml(-):'foo'&lt;3&gt;</code></pre>
<p>Now we can print out the value of <code>x0</code> before it gets saved on the stack.</p>
<p>We can also lookup symbols in the executable using <code>image lookup -r -n &lt;symbol_name&gt;</code> if we are not sure of the specific name we want.</p>
<pre class="shell"><code>(lldb) image lookup -r -n camlTest
4 matches found in /home/tsmc/projects/test_blocks.exe:
        Address: test_blocks.exe[0x0000000000019a78] (test_blocks.exe.PT_LOAD[0]..text + 1912)
        Summary: test_blocks.exe`camlStd_exit.code_end
        Address: test_blocks.exe[0x0000000000019b48] (test_blocks.exe.PT_LOAD[0]..text + 2120)
        Summary: test_blocks.exe`camlTest_blocks.code_end
        Address: test_blocks.exe[0x0000000000019ae0] (test_blocks.exe.PT_LOAD[0]..text + 2016)
        Summary: test_blocks.exe`camlTest_blocks.entry
        Address: test_blocks.exe[0x0000000000019a88] (test_blocks.exe.PT_LOAD[0]..text + 1928)
        Summary: test_blocks.exe`camlTest_blocks.main_273</code></pre>
<p>Finally setting breakpoints on macOS with LLDB is slightly broken so you need to lookup the symbol name and then set the breakpoint based on the address of the symbol. We can combine <code>image lookup</code> with setting breakpoints on addresses to debug on macOS.</p>
<h2>More for later</h2>
<p>There is a lot more to say about debugging OCaml programs using LLDB and there is ongoing work to improve debugger support in OCaml. Get in touch if you would like to be involved.</p>
</div>

