---
title: Attempting overlayfs with macFuse
description: "It would be great if overlayFS or unionFS worked on macOS! Initially,
  I attempted to use DYLD_INTERPOSE, but I wasn\u2019t able to intercept enough system
  calls to get it to work. However, macFuse provides a way to implement our own userspace
  file systems. Patrick previously wrote obuilder-fs, which implemented a per-user
  filesystem redirection. It would be interesting to extend this concept to provide
  an overlayfs-style implementation."
url: https://www.tunbury.org/2025/10/06/overlayfs-macFuse/
date: 2025-10-06T06:00:00-00:00
preview_image: https://www.tunbury.org/images/macfuse-home.png
authors:
- Mark Elvers
source:
ignore:
---

<p>It would be great if overlayFS or unionFS worked on macOS! Initially, I attempted to use DYLD_INTERPOSE, but I wasn’t able to intercept enough system calls to get it to work. However, macFuse provides a way to implement our own userspace file systems. Patrick previously wrote <a href="https://github.com/ocurrent/obuilder-fs">obuilder-fs</a>, which implemented a per-user filesystem redirection. It would be interesting to extend this concept to provide an overlayfs-style implementation.</p>

<p>My approach was to use an environment variable to flag which process should have the I/O redirected. When the user space layer of Fuse is called, the context includes the UID of the calling process. It is then possible to query the process’s environment and check for the marker variables. If none are found, then we can check the parent process. This won’t work for a double <code class="language-plaintext highlighter-rouge">fork()</code>, but it’s good enough to traverse <code class="language-plaintext highlighter-rouge">sudo</code>. Processes without the environment marker will pass through to the existing path.</p>

<p>Passing through to the existing path is easier said than done. When the Fuse filesystem is mounted, the content of the underlying filesystem is completely hidden. The workaround was to move the existing files out of the way and redirect to requests to this temporary directory.</p>

<p>Initially, this showed promise as trivial commands like <code class="language-plaintext highlighter-rouge">stat</code> and <code class="language-plaintext highlighter-rouge">ls</code> worked. However, the excitement was short-lived as complex commands failed with “Device not configured”.</p>

<p>For example, with Fuse mounted on <code class="language-plaintext highlighter-rouge">/usr/local</code>, some files and directories were created in <code class="language-plaintext highlighter-rouge">/tmp/a</code>, but very few.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>% <span class="nv">WRAPPER</span><span class="o">=</span>/tmp/a git <span class="nt">-C</span> /usr/local clone https://github.com/ocaml/opam-repository
Cloning into <span class="s1">'opam-repository'</span>...
/System/Volumes/Data/usr/local/opam-repository/.git/hooks/: Device not configured
</code></pre></div></div>

<p>The log showed that <code class="language-plaintext highlighter-rouge">fseventsd</code> tried to query all the directories which <code class="language-plaintext highlighter-rouge">git</code> created, but since it didn’t have the environment variable set, it couldn’t find the files. After a few failures, <code class="language-plaintext highlighter-rouge">fseventsd</code> seem to mark the filesystem as bad and block access. The log snippet below shows a a typically request from <code class="language-plaintext highlighter-rouge">fseventsd</code></p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>unique: 8, opcode: GETATTR (3), nodeid: 21, insize: 56, pid: 522
getattr /opam-repository/.git
Searching for WRAPPER in process tree starting from PID 522:
    PID 522 has 1 args, checking environment...
    arg[0]: /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/FSEvents.framework/Versions/A/Support/fseventsd
    Checked 4 environment variables, no WRAPPER found
  PID 522 (fseventsd): no wrapper
No WRAPPER found in process tree
*** GETATTR PASSTHROUGH: /opam-repository/.git -&gt; /System/Volumes/Data/usr/local.fuse/opam-repository/.git ***
   unique: 8, error: -2 (No such file or directory), outsize: 16
unique: 6, opcode: LOOKUP (1), nodeid: 20, insize: 45, pid: 522
LOOKUP /opam-repository/.git
getattr /opam-repository/.git
Searching for WRAPPER in process tree starting from PID 522:
    PID 522 has 1 args, checking environment...
    arg[0]: /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/FSEvents.framework/Versions/A/Support/fseventsd
    Checked 4 environment variables, no WRAPPER found
  PID 522 (fseventsd): no wrapper
No WRAPPER found in process tree
*** GETATTR PASSTHROUGH: /opam-repository/.git -&gt; /System/Volumes/Data/usr/local.fuse/opam-repository/.git ***
   unique: 6, error: -2 (No such file or directory), outsize: 16
</code></pre></div></div>

<p>Searching online suggested that <code class="language-plaintext highlighter-rouge">fseventsd</code> could be blocked by creating a file named <code class="language-plaintext highlighter-rouge">/.fseventsd/no_log</code> on the filesystem. This didn’t work. Since the incoming request always came from <code class="language-plaintext highlighter-rouge">fseventsd</code> could it be blocked at the Fuse level?  As a quick test, I tried returning <code class="language-plaintext highlighter-rouge">ENOTSUP</code> based on the PID, and that worked! I replaced the static PID with a call to <code class="language-plaintext highlighter-rouge">proc_pidpath()</code> and matched the name against <code class="language-plaintext highlighter-rouge">fseventsd</code>.</p>

<div class="language-c highlighter-rouge"><div class="highlight"><pre class="highlight"><code>    <span class="k">if</span> <span class="p">(</span><span class="n">context</span><span class="o">-&gt;</span><span class="n">pid</span> <span class="o">==</span> <span class="mi">522</span><span class="p">)</span> <span class="p">{</span>
        <span class="k">return</span> <span class="o">-</span><span class="n">ENOTSUP</span><span class="p">;</span>
    <span class="p">}</span>
</code></pre></div></div>

<p>With this working, I implemented an overlayfs-style semantics using environment variables <code class="language-plaintext highlighter-rouge">WRAPPER_UPPER</code> and <code class="language-plaintext highlighter-rouge">WRAPPER_LOWER</code>. Deletions are handled by creating a whiteout directory, <code class="language-plaintext highlighter-rouge">.deleted</code>, at the root, which is populated with empty files that reflect the files/directories which have been deleted. If a file <code class="language-plaintext highlighter-rouge">bar</code> is deleted from directory <code class="language-plaintext highlighter-rouge">foo</code>, then <code class="language-plaintext highlighter-rouge">/.delete/foo/bar</code> would be created. Later, if <code class="language-plaintext highlighter-rouge">foo</code> was removed, the directory <code class="language-plaintext highlighter-rouge">foo</code> would be removed from the whiteout directory and be replaced with a file instead. <code class="language-plaintext highlighter-rouge">/.deleted/foo</code></p>

<p>opendir()/readdir() were the most complex functions to implement, as they needed to scan the upper directory and merge in the lower directory, taking account of any deleted files and hide the <code class="language-plaintext highlighter-rouge">/.deleted</code> directory.</p>

<p>The redirection worked. For example, given these steps, <code class="language-plaintext highlighter-rouge">/tmp/a</code> would be empty, <code class="language-plaintext highlighter-rouge">/tmp/b</code> contains the vanilla checkout of opam-repository, and <code class="language-plaintext highlighter-rouge">/tmp/c</code> contains the difference: <code class="language-plaintext highlighter-rouge">/tmp/c/.deleted</code> with the files removed, <code class="language-plaintext highlighter-rouge">/tmp/c/opam-repository/...</code> and <code class="language-plaintext highlighter-rouge">/tmp/c/opam-repository/.git</code> with just the files which contain differences.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>% <span class="nb">mkdir</span> /tmp/a /tmp/b /tmp/c
% <span class="nv">WRAPPER_LOWER</span><span class="o">=</span>/tmp/a <span class="nv">WRAPPER_UPPER</span><span class="o">=</span>/tmp/b git <span class="nt">-C</span> /usr/local clone https://github.com/ocaml/opam-repository
% <span class="nv">WRAPPER_LOWER</span><span class="o">=</span>/tmp/b <span class="nv">WRAPPER_UPPER</span><span class="o">=</span>/tmp/c git <span class="nt">-C</span> /usr/local/opam-repository checkout c35a0314d6c7c7260c978f490fb8f7109f4e9766
</code></pre></div></div>

<p>Extending this further allows <code class="language-plaintext highlighter-rouge">/tmp/d</code> to be created with a different delta.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>% <span class="nb">mkdir</span> /tmp/d
% <span class="nv">WRAPPER_LOWER</span><span class="o">=</span>/tmp/b <span class="nv">WRAPPER_UPPER</span><span class="o">=</span>/tmp/d git <span class="nt">-C</span> /usr/local/opam-repository checkout f33f62ebff75cd03620d09d46a4540340f5564a6
</code></pre></div></div>

<p>Annoyingly, this revealed a significant issue: running <code class="language-plaintext highlighter-rouge">git status</code> on <code class="language-plaintext highlighter-rouge">/tmp/c</code> showed that files had changed. I presumed there was a flaw in my code which was corrupting the files, but I couldn’t find it. Examining the files on disk showed that they were correct, but when reading them through Fuse, gave different data:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>% <span class="k">for </span>x <span class="k">in </span>c d <span class="p">;</span> <span class="k">do </span><span class="nb">cat</span> /tmp/<span class="nv">$x</span>/opam-repository/.git/HEAD <span class="p">;</span> <span class="nv">WRAPPER_LOWER</span><span class="o">=</span>/tmp/b <span class="nv">WRAPPER_UPPER</span><span class="o">=</span>/tmp/<span class="nv">$x</span> <span class="nb">cat</span> /usr/local/opam-repository/.git/HEAD <span class="p">;</span> <span class="k">done
</span>c35a0314d6c7c7260c978f490fb8f7109f4e9766
c35a0314d6c7c7260c978f490fb8f7109f4e9766
f33f62ebff75cd03620d09d46a4540340f5564a6
c35a0314d6c7c7260c978f490fb8f7109f4e9766
</code></pre></div></div>

<p>The log showed the root cause - two OPEN calls, but only a single READ. The kernel is caching the reads.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>% <span class="nb">grep </span>OPEN log5.txt
unique: 2, opcode: OPEN <span class="o">(</span>14<span class="o">)</span>, nodeid: 4, insize: 48, pid: 52976
<span class="k">***</span> OPEN: /opam-repository/.git/HEAD from UPPER: /tmp/c/opam-repository/.git/HEAD <span class="k">***</span>
unique: 3, opcode: OPEN <span class="o">(</span>14<span class="o">)</span>, nodeid: 4, insize: 48, pid: 52980
<span class="k">***</span> OPEN: /opam-repository/.git/HEAD from UPPER: /tmp/d/opam-repository/.git/HEAD <span class="k">***</span>

% <span class="nb">grep </span>READ log5.txt
unique: 3, opcode: READ <span class="o">(</span>15<span class="o">)</span>, nodeid: 4, insize: 80, pid: 52976
</code></pre></div></div>

<p>You can disable attribute caching with <code class="language-plaintext highlighter-rouge">-o attr_timeout=0 -o entry_timeout=0</code>, and you can circumvent the cache by specifying <code class="language-plaintext highlighter-rouge">-o direct_io</code>. Setting <code class="language-plaintext highlighter-rouge">direct_io</code> is sufficient to resolve the issue in a simple <code class="language-plaintext highlighter-rouge">cat</code> test, but it has the side effect of disabling <code class="language-plaintext highlighter-rouge">mmap()</code>, which causes <code class="language-plaintext highlighter-rouge">git</code> to crash with a <code class="language-plaintext highlighter-rouge">bus error</code>. Setting <code class="language-plaintext highlighter-rouge">fi-&gt;keep_cache = 0</code> doesn’t prevent the cache.</p>

<p>The kernel asks Fuse to allocate a node ID for a path. The node ID number is passed as a parameter to GETATTR, OPEN and READ. Even though GETATTR returns different mtime values at the second call, the kernel still sees a cache hit and returns the file content from the cache.</p>

<p>To control the node ID allocation process this needs to be rewritten using the Fuse low level API. This would allow full control over the allocation process and gives access to calls such as <code class="language-plaintext highlighter-rouge">fuse_lowlevel_notify_inval_inode()</code>.</p>

<p>My work-in-progress code is available on GitHub <a href="https://github.com/mtelvers/macfuse/blob/master/LoopbackFS-C/loopback/loopback.c">mtelvers/macfuse</a>.</p>
