---
title: Debugging OCaml with Emacs
description:
url: https://lambdafoo.com/posts/2024-03-25-ocaml-debugging-with-emacs.html
date: 2024-03-25T00:00:00-00:00
preview_image:
authors:
- Tim McGilchrist
source:
ignore:
---

<div class="post">
  <h1 class="post-title">Debugging OCaml with Emacs</h1>
  <span class="post-date">March 25, 2024</span>
  <p>This post started as a summary of my March Hacking Days effort at <a href="https://tarides.com">Tarides</a>.</p>
<p>I have been working on improving the debugging situation for OCaml and wanted to see how easily I could setup debug support in Emacs using DAP. Debug Adapter Protocol (DAP) is a wire protocol for communicating between an editor or IDE and a debug server like <a href="https://lldb.llvm.org">LLDB</a> or <a href="https://sourceware.org/gdb/">GDB</a>, providing an abstraction over debugging, similar to how <a href="https://microsoft.github.io/language-server-protocol/">Language Server Protocol (LSP)</a> provides language support for editors.</p>
<p>OCaml comes with support for debugging native programs with GDB and LLDB, and bytecode code programs using <a href="https://v2.ocaml.org/manual/debugger.html">ocamldebug</a> and <a href="https://github.com/hackwaly/ocamlearlybird">earlybird</a>. In this post we will cover setting up and debugging both kinds of programs. I am using an M3 Mac so all examples will show ARM64 assembly and macOS specific paths. The same setup should work on Linux. I use <a href="https://github.com/bbatsov/prelude">prelude</a> to configure my Emacs with my own customistations in <code>.emacs/personal</code>, adjust for your own personal Emacs setup.</p>
<p>Let’s start with the following program to compute Fibonacci sequence:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb1-1" aria-hidden="true" tabindex="-1"></a><span class="co">(* fib.ml *)</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-2" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> <span class="kw">rec</span> fib n =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-3" aria-hidden="true" tabindex="-1"></a>  <span class="kw">if</span> n = <span class="dv">0</span> <span class="kw">then</span> <span class="dv">0</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-4" aria-hidden="true" tabindex="-1"></a>  <span class="kw">else</span> <span class="kw">if</span> n = <span class="dv">1</span> <span class="kw">then</span> <span class="dv">1</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-5" aria-hidden="true" tabindex="-1"></a>  <span class="kw">else</span> fib (n<span class="dv">-1</span>) + fib (n<span class="dv">-2</span>)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-6" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-7" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> main () =</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-8" aria-hidden="true" tabindex="-1"></a>  <span class="kw">let</span> r = fib <span class="dv">20</span> <span class="kw">in</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-9" aria-hidden="true" tabindex="-1"></a>  <span class="dt">Printf</span>.printf <span class="st">"fib(20) = %d"</span> r</span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-10" aria-hidden="true" tabindex="-1"></a></span>
<span><a href="https://lambdafoo.com/rss.xml#cb1-11" aria-hidden="true" tabindex="-1"></a><span class="kw">let</span> _ = main ()</span></code></pre></div>
<p>And this <code>dune</code> configuration in the same directory:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb2-1" aria-hidden="true" tabindex="-1"></a>(executable</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-2" aria-hidden="true" tabindex="-1"></a> (name fib)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-3" aria-hidden="true" tabindex="-1"></a> (modules fib)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb2-4" aria-hidden="true" tabindex="-1"></a> (modes exe byte))</span></code></pre></div>
<p>And this <code>dune-project</code> configuration in the same directory:</p>
<div class="sourceCode"><pre class="sourceCode ocaml"><code class="sourceCode ocaml"><span><a href="https://lambdafoo.com/rss.xml#cb3-1" aria-hidden="true" tabindex="-1"></a>(lang dune <span class="fl">3.11</span>)</span>
<span><a href="https://lambdafoo.com/rss.xml#cb3-2" aria-hidden="true" tabindex="-1"></a>(map_workspace_root <span class="kw">false</span>)</span></code></pre></div>
<p>Create an empty <code>opam</code> switch in same directory and install dune:</p>
<pre class="shell"><code>$ opam switch create . 5.1.1 --no-install
$ opam install dune</code></pre>
<p>This gives us everything we need to try out all the different debuggers.</p>
<h2>Emacs configuration</h2>
<p>Emacs has <a href="https://github.com/emacs-lsp/dap-mode">dap-mode</a> that provides everything we need for DAP integration. Install it using <code>M-x package-install</code> and choose the <code>dap-mode</code> package. I have the following lines in my <code>.emacs/personal/init.el</code> that will require the packages we need and setup some convenient key bindings:</p>
<pre class="emacs-lisp"><code>; Require dap-mode plus the two extra files we need
(require 'dap-mode)
(require 'dap-codelldb)
(require 'dap-ocaml)

; Setup key bindings using use-package.
(use-package dap-mode
  :bind (("C-c M-n" . dap-next)
         ("C-c M-s" . dap-step-in)
         ("C-c M-a" . dap-step-out)
         ("C-c M-w" . dap-continue)))</code></pre>
<p>Save and restart Emacs, then we can move onto setting up bytecode debugging.</p>
<h2>Bytecode debugging</h2>
<p>The <a href="https://github.com/hackwaly/ocamlearlybird">earlybird</a> project provides DAP support for debugging OCaml bytecode. OCaml has a bytecode compiler that produces portable bytecode executables which can be run with <code>ocamlrun</code>, the interpreter for OCaml bytecode. Earlybird uses the (undocumented) protocol of <code>ocamldebug</code> to communicate with a bytecode executable, inheriting the same <a href="https://v2.ocaml.org/manual/debugger.html">functionality as ocamldebug</a>.</p>
<p>Start by installing the <code>earlybird</code> package:</p>
<pre class="shell"><code>opam install earlybird</code></pre>
<p>Then create a file in <code>.vscode/launch.json</code> with this configuration:</p>
<div class="sourceCode"><pre class="sourceCode json"><code class="sourceCode json"><span><a href="https://lambdafoo.com/rss.xml#cb7-1" aria-hidden="true" tabindex="-1"></a><span class="fu">{</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-2" aria-hidden="true" tabindex="-1"></a>    <span class="dt">"version"</span><span class="fu">:</span> <span class="st">"0.2.0"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-3" aria-hidden="true" tabindex="-1"></a>    <span class="dt">"configurations"</span><span class="fu">:</span> <span class="ot">[</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-4" aria-hidden="true" tabindex="-1"></a>        <span class="fu">{</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-5" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"name"</span><span class="fu">:</span> <span class="st">"OCaml earlybird (experimental)"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-6" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"type"</span><span class="fu">:</span> <span class="st">"ocaml.earlybird"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-7" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"request"</span><span class="fu">:</span> <span class="st">"launch"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-8" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"program"</span><span class="fu">:</span> <span class="st">"./_build/default/fib.bc"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-9" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"stopOnEntry"</span><span class="fu">:</span> <span class="kw">true</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-10" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"cwd"</span><span class="fu">:</span> <span class="st">"${workspaceFolder}"</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-11" aria-hidden="true" tabindex="-1"></a>        <span class="fu">}</span><span class="ot">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-12" aria-hidden="true" tabindex="-1"></a>    <span class="ot">]</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb7-13" aria-hidden="true" tabindex="-1"></a><span class="fu">}</span></span></code></pre></div>
<p>Build the project with <code>dune build</code> to create the <code>fib.bc</code> bytecode file. Finally start a debugger by running <code>M-x dap-debug</code>. It will prompt you to choose a session, we want <code>OCaml earlybird (experimental)</code> from the named configuration above. It will start earlybird and immediately stop it before executing any OCaml code.</p>
<figure>
<img src="https://lambdafoo.com/images/earlybird-dap-template.png" alt="Starting earlybird from Emacs">
<figcaption aria-hidden="true">Starting earlybird from Emacs</figcaption>
</figure>
<p>To set breakpoints you need to open the OCaml source file in <code>_build/default/fib.ml</code> and click on the source lines you want to stop at. Here is what it looks like after a few recursions. Use the buttons to control the debugger or use the keybindings we added to step through the execution. Curiously they are not pre-defined but here I’ve tried to reuse mappings from <a href="https://v2.ocaml.org/manual/debugger.html#s:inf-debugger">ocamldebug</a>.</p>
<figure>
<img src="https://lambdafoo.com/images/earlybird-dap-startup.png" alt="Running earlybird through fibonacci">
<figcaption aria-hidden="true">Running earlybird through fibonacci</figcaption>
</figure>
<h2>Native debugging</h2>
<p>OCaml can also produce native binaries that can be debugged using native debuggers like GDB or LLDB, depending on your platform. Here we will use LLDB on macOS, but Linux LLDB works too – just change the name of the program you want to debug.</p>
<p>Add another section to <code>.vscode/launch.json</code> for starting lldb.</p>
<div class="sourceCode"><pre class="sourceCode json"><code class="sourceCode json"><span><a href="https://lambdafoo.com/rss.xml#cb8-1" aria-hidden="true" tabindex="-1"></a>        <span class="fu">{</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb8-2" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"type"</span><span class="fu">:</span> <span class="st">"lldb"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb8-3" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"request"</span><span class="fu">:</span> <span class="st">"launch"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb8-4" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"name"</span><span class="fu">:</span> <span class="st">"LLDB with ocamlopt"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb8-5" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"program"</span><span class="fu">:</span> <span class="st">"./fib.exe"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb8-6" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"args"</span><span class="fu">:</span> <span class="ot">[]</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb8-7" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"stopOnEntry"</span><span class="fu">:</span> <span class="kw">true</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb8-8" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"cwd"</span><span class="fu">:</span> <span class="st">"${workspaceFolder}"</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb8-9" aria-hidden="true" tabindex="-1"></a>        <span class="fu">}</span><span class="er">,</span></span></code></pre></div>
<p>Run <code>M-x dap-codelldb-setup</code> which will download the <code>codelldb</code> DAP program that we are using to communicate with LLDB. This gets installed into <code>.extension/vscode/codelldb</code>. Now compile the fib program with <code>ocamlopt -g -o fib.exe fib.ml</code> and startup a debugger session with <code>M-x dap-debug</code> choose the <code>LLDB with ocamlopt</code> option. You should see something similar to:</p>
<figure>
<img src="https://lambdafoo.com/images/lldb-dap-startup.png" alt="codelldb dap startup">
<figcaption aria-hidden="true">codelldb dap startup</figcaption>
</figure>
<p>Now DAP as setup with LLDB and macOS, is a little broken and is missing support for setting breakpoints on symbols and line numbers in source code. Fixes for both will be comming soon. Linux LLDB works better in this scenario. Setting breakpoints using line numbers in source code requires fixes to the OCaml compiler, while setting breakpoints on symbols is supported in <code>codelldb</code> but not exposed into <code>dap-mode</code>.</p>
<p>The second option is debugging native binaries built with Dune, this is slightly different for two reasons. First Dune places the executable into <code>_build/default/fib.exe</code> and second Dune produces slightly different symbols. Start by adding a new section in <code>.vscode/launch.json</code> for Dune built executables:</p>
<div class="sourceCode"><pre class="sourceCode json"><code class="sourceCode json"><span><a href="https://lambdafoo.com/rss.xml#cb9-1" aria-hidden="true" tabindex="-1"></a>        <span class="fu">{</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-2" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"type"</span><span class="fu">:</span> <span class="st">"lldb"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-3" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"request"</span><span class="fu">:</span> <span class="st">"launch"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-4" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"name"</span><span class="fu">:</span> <span class="st">"LLDB with Dune"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-5" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"program"</span><span class="fu">:</span> <span class="st">"./_build/default/fib.exe"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-6" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"args"</span><span class="fu">:</span> <span class="ot">[]</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-7" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"stopOnEntry"</span><span class="fu">:</span> <span class="kw">true</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-8" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"cwd"</span><span class="fu">:</span> <span class="st">"${workspaceFolder}"</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb9-9" aria-hidden="true" tabindex="-1"></a>        <span class="fu">}</span><span class="er">,</span></span></code></pre></div>
<p>Remove the old <code>fib.exe</code> in the project directory (dune will complain if you don’t) and run <code>dune build</code>. Startup a new DAP session with <code>M-x dap-debug</code> and choose <code>LLDB with Dune</code>. You should see the same debugger session as before.</p>
<h2>Conclusion</h2>
<p>Debugging OCaml with DAP inside Emacs is possible. There are working options for both bytecode programs and native programs which work reasonably well.</p>
<p>Use <code>dap-mode</code> with:</p>
<pre class="emacs-lisp"><code>(require 'dap-mode)
(require 'dap-codelldb)
(require 'dap-ocaml)

(use-package dap-mode
  :bind (("C-c M-n" . dap-next)
         ("C-c M-s" . dap-step-in)
         ("C-c M-a" . dap-step-out)
         ("C-c M-w" . dap-continue)))
</code></pre>
<p>and a <code>launch.json</code> of</p>
<div class="sourceCode"><pre class="sourceCode json"><code class="sourceCode json"><span><a href="https://lambdafoo.com/rss.xml#cb11-1" aria-hidden="true" tabindex="-1"></a><span class="fu">{</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-2" aria-hidden="true" tabindex="-1"></a>    <span class="dt">"version"</span><span class="fu">:</span> <span class="st">"0.2.0"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-3" aria-hidden="true" tabindex="-1"></a>    <span class="dt">"configurations"</span><span class="fu">:</span> <span class="ot">[</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-4" aria-hidden="true" tabindex="-1"></a>        <span class="fu">{</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-5" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"name"</span><span class="fu">:</span> <span class="st">"OCaml earlybird (experimental)"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-6" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"type"</span><span class="fu">:</span> <span class="st">"ocaml.earlybird"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-7" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"request"</span><span class="fu">:</span> <span class="st">"launch"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-8" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"program"</span><span class="fu">:</span> <span class="st">"./_build/default/fib.bc"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-9" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"stopOnEntry"</span><span class="fu">:</span> <span class="kw">true</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-10" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"cwd"</span><span class="fu">:</span> <span class="st">"${workspaceFolder}"</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-11" aria-hidden="true" tabindex="-1"></a>        <span class="fu">}</span><span class="ot">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-12" aria-hidden="true" tabindex="-1"></a>        <span class="fu">{</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-13" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"type"</span><span class="fu">:</span> <span class="st">"lldb"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-14" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"request"</span><span class="fu">:</span> <span class="st">"launch"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-15" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"name"</span><span class="fu">:</span> <span class="st">"LLDB with Dune"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-16" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"program"</span><span class="fu">:</span> <span class="st">"./_build/default/fib.exe"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-17" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"args"</span><span class="fu">:</span> <span class="ot">[]</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-18" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"stopOnEntry"</span><span class="fu">:</span> <span class="kw">true</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-19" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"cwd"</span><span class="fu">:</span> <span class="st">"${workspaceFolder}"</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-20" aria-hidden="true" tabindex="-1"></a>        <span class="fu">}</span><span class="ot">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-21" aria-hidden="true" tabindex="-1"></a>        <span class="fu">{</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-22" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"type"</span><span class="fu">:</span> <span class="st">"lldb"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-23" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"request"</span><span class="fu">:</span> <span class="st">"launch"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-24" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"name"</span><span class="fu">:</span> <span class="st">"LLDB with ocamlopt"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-25" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"program"</span><span class="fu">:</span> <span class="st">"./fib.exe"</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-26" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"args"</span><span class="fu">:</span> <span class="ot">[]</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-27" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"stopOnEntry"</span><span class="fu">:</span> <span class="kw">true</span><span class="fu">,</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-28" aria-hidden="true" tabindex="-1"></a>            <span class="dt">"cwd"</span><span class="fu">:</span> <span class="st">"${workspaceFolder}"</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-29" aria-hidden="true" tabindex="-1"></a>        <span class="fu">}</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-30" aria-hidden="true" tabindex="-1"></a>    <span class="ot">]</span></span>
<span><a href="https://lambdafoo.com/rss.xml#cb11-31" aria-hidden="true" tabindex="-1"></a><span class="fu">}</span></span></code></pre></div>
<p>The same setup will work under VSCode with the <code>CodeLLDB</code> and <code>OCaml Platform</code> extensions installed. Happy Emacs debugging.</p>
<h2>Future Work</h2>
<p>I’m working on improving the OCaml debugging experience on macOS and Linux. Currently the macOS LLDB experience is behind that on Linux LLDB, so that is the first goal. Then I want to improve the DWARF encodings for OCaml and generally improve the native debugger experience.</p>
</div>

