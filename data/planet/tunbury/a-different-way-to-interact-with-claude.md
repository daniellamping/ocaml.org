---
title: A different way to interact with Claude
description: "We\u2019ve all been using Claude via the prompt, and some have even
  ventured into running claude --dangerously-skip-permissions in a nice sandbox like
  avsm/claude-ocaml-devcontainer."
url: https://www.tunbury.org/2026/03/18/interact-with-claude/
date: 2026-03-18T15:20:00-00:00
preview_image: https://www.tunbury.org/images/anthropic-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>We’ve all been using Claude via the prompt, and some have even ventured into running <code class="language-plaintext highlighter-rouge">claude --dangerously-skip-permissions</code> in a nice sandbox like <a href="https://github.com/avsm/claude-ocaml-devcontainer">avsm/claude-ocaml-devcontainer</a>.</p>

<p>I have a number of <code class="language-plaintext highlighter-rouge">tmux</code> sessions running and periodically ask the running Claude to check on the free disk space or review the output of a diagnostic command in the context of the current session, but it’s still a prompt. I want Claude to do things regularly without prompting.</p>

<p>Claude accepts prompts on the command line <code class="language-plaintext highlighter-rouge">claude -p Hello</code>, so can this be extended to do something useful?</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>claude <span class="nt">-p</span> Hello
Hello! How can I <span class="nb">help </span>you today?
</code></pre></div></div>

<p>In your project directory (or globally), you can create <code class="language-plaintext highlighter-rouge">.claude/settings.local.json</code>. Perhaps as below. Note that the path <code class="language-plaintext highlighter-rouge">/</code> refers to the root of the project directory, and a double slash refers to the root of the file system. e.g. <code class="language-plaintext highlighter-rouge">//usr/bin</code>.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>{
  "permissions": {
    "allow": [
      "Bash(ssh foo.bar *)",
      "Write(/README.md)",
      "Bash(git *)"
    ]
  }
}
</code></pre></div></div>

<p>However, this pattern matching is tiresome, <code class="language-plaintext highlighter-rouge">ssh foo.bar *</code> doesn’t match <code class="language-plaintext highlighter-rouge">ssh -t foo.bar ...</code> as <code class="language-plaintext highlighter-rouge">-t</code> has invalidated the match, so you end up putting <code class="language-plaintext highlighter-rouge">ssh * foo.bar *</code> but then you <em>must</em> have a parameter. Also, <code class="language-plaintext highlighter-rouge">git commit -m "my commit"</code> matches, but <code class="language-plaintext highlighter-rouge">git commit -m "my\nmultiline\ncomment"</code> does not. It was very frustrating. Thus, I abandoned permissions and went for a container and <code class="language-plaintext highlighter-rouge">--dangerously-skip-permissions</code>.</p>

<p>You can get started with a simple Dockerfile, but you might want to augment it later. In mine, I change the default user from <code class="language-plaintext highlighter-rouge">node:node</code> to be <code class="language-plaintext highlighter-rouge">mtelvers:mtelvers</code> to match my local machine.</p>

<div class="language-dockerfile highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">FROM</span><span class="s"> node:22-trixie-slim</span>
<span class="k">RUN </span>apt update <span class="o">&amp;&amp;</span> <span class="nb">install </span>git <span class="nt">-y</span>
<span class="k">RUN </span>npm <span class="nb">install</span> <span class="nt">-g</span> @anthropic-ai/claude-code
<span class="k">RUN </span>usermod <span class="nt">-l</span> mtelvers <span class="nt">-d</span> /home/mtelvers <span class="nt">-m</span> node <span class="o">&amp;&amp;</span> <span class="se">\
</span>    groupmod <span class="nt">-n</span> mtelvers node <span class="o">&amp;&amp;</span> <span class="se">\
</span>    git config <span class="nt">--system</span> user.name <span class="s2">"mtelvers"</span> <span class="o">&amp;&amp;</span> <span class="se">\
</span>    git config <span class="nt">--system</span> user.email <span class="s2">"mtelvers@example.com"</span>
</code></pre></div></div>

<p>I build the <code class="language-plaintext highlighter-rouge">Dockerfile</code> with <code class="language-plaintext highlighter-rouge">docker build -t bot .</code>, and execute it using a shell script <code class="language-plaintext highlighter-rouge">run.sh</code> which is a wrapper around <code class="language-plaintext highlighter-rouge">docker run</code>, setting my user uid and gid to match my local machine and mapping my <code class="language-plaintext highlighter-rouge">~/.claude</code> directory. I run Claude with <code class="language-plaintext highlighter-rouge">--output-format stream-json</code>, which outputs the JSON “thinking” in real time, which I filter through <code class="language-plaintext highlighter-rouge">jq</code>.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">#!/bin/bash</span>
<span class="nb">set</span> <span class="nt">-euo</span> pipefail

docker run <span class="nt">--rm</span> <span class="se">\</span>
  <span class="nt">--user</span> <span class="s2">"</span><span class="si">$(</span><span class="nb">id</span> <span class="nt">-u</span><span class="si">)</span><span class="s2">:</span><span class="si">$(</span><span class="nb">id</span> <span class="nt">-g</span><span class="si">)</span><span class="s2">"</span> <span class="se">\</span>
  <span class="nt">-e</span> <span class="nv">HOME</span><span class="o">=</span>/home/mtelvers <span class="se">\</span>
  <span class="nt">-v</span> <span class="s2">"</span><span class="nv">$HOME</span><span class="s2">/.claude:/home/mtelvers/.claude"</span> <span class="se">\</span>
  <span class="nt">-v</span> <span class="s2">"</span><span class="nv">$HOME</span><span class="s2">/.claude/.claude.json:/home/mtelvers/.claude.json"</span> <span class="se">\</span>
  <span class="nt">-v</span> <span class="s2">"</span><span class="nv">$PWD</span><span class="s2">:/work"</span> <span class="se">\</span>
  <span class="nt">-w</span> /work <span class="se">\</span>
  test-bot claude <span class="nt">--dangerously-skip-permissions</span> <span class="nt">--verbose</span> <span class="nt">--output-format</span> stream-json <span class="nt">-p</span> <span class="s2">"</span><span class="nv">$@</span><span class="s2">"</span> 2&gt;/dev/null | jq <span class="nt">--unbuffered</span> <span class="nt">-r</span> <span class="s1">'
    if .type == "assistant" then
      .message.content[]? |
      if .type == "text" then "💬 \(.text)"
      elif .type == "tool_use" then "🔧 \(.name): \(.input | tostring | .[0:200])"
      else empty end
    elif .type == "result" then
      "📊 Cost: $\(.total_cost_usd // "?") | Duration: \(.duration_ms // "?")ms"
    else empty end
  '</span>
</code></pre></div></div>

<p>I sync the uid/gid and username to avoid file permission issues.</p>

<p>Claude is now running in a container, but essentially operating in the same way as before:-</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>./run.sh <span class="nt">-p</span> <span class="s2">"What is the value of pi"</span>
💬 

π ≈ 3.14159265358979323846…
📊 Cost: <span class="nv">$0</span>.017534 | Duration: 2691ms
</code></pre></div></div>

<p>The prompt is now the rest of the work. As an example, make the current directory a git repo with <code class="language-plaintext highlighter-rouge">git init .</code>, then create <code class="language-plaintext highlighter-rouge">NOTES.md</code> and run with <code class="language-plaintext highlighter-rouge">./run.sh -p "@NOTES.md"</code>.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>This is a git repo containing our working notes. Follow the checklist below **in order**, completing every step. Do not skip any steps.

# Steps (complete ALL of these, in order, every run)

1. **Gather data** -- check the status of this machine - processes, load, disk etc
2. **Update NOTES.md** -- Rewrite this file with the latest status. Rules:
   - Keep the file well structured. Don't mix up knowledge with history.
   - Ask questions in the relevant section.
   - Summarise question answers into knowledge
3. **Git commit** -- Stage and commit NOTES.md with a descriptive commit message summarising what changed.

# Log

# Knowledge

# Questions
</code></pre></div></div>

<p>Claude dutifully follows things through, updates and commits <code class="language-plaintext highlighter-rouge">NOTES.md</code>.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>This is a git repo containing our working notes. Follow the checklist below **in order**, completing every step. Do not skip any steps.

# Steps (complete ALL of these, in order, every run)

1. **Gather data** -- check the status of this machine - processes, load, disk etc
2. **Update NOTES.md** -- Rewrite this file with the latest status. Rules:
   - Keep the file well structured. Don't mix up knowledge with history.
   - Ask questions in the relevant section.
   - Summarise question answers into knowledge
3. **Git commit** -- Stage and commit NOTES.md with a descriptive commit message summarising what changed.

# Log

## 2026-03-18

- **Load average:** 4.23 / 8.31 / 9.60 (1/5/15 min)
- **Uptime:** ~35.2 days
- **Memory:** 189 GB total, ~88 GB available (MemFree: 1.3 GB, Buffers: 65 GB, Cached: 6.5 GB)
- **Disk (/):** 1.8 TB total, 1.1 TB used, 640 GB available (62% used)
- **CPU:** 40 cores — Intel Xeon E5-2640 v4 @ 2.40GHz
- **Kernel:** 6.8.0-100-generic
- **Hostname:** 119c0548ec30 (container)
- **Processes:** `ps` unavailable in this environment; /proc shows 1332 total threads, 5 currently running

# Knowledge

- This is a containerised environment (overlay filesystem, short hex hostname).
- The machine has 40 Xeon cores and ~189 GB RAM — a large server or VM.
- `uptime`, `free`, `ps`, and other common utilities are not installed; status must be gathered from `/proc` and `df`.
- Load is moderate relative to core count (4.2 / 8.3 / 9.6 on 40 cores).
- Disk is at 62% — not critical but worth monitoring.

# Questions

- What workloads are driving the 15-min load average of 9.6? (No `ps` available to inspect.)
- Is the low MemFree (1.3 GB) expected given the large buffer/cache usage, or is there memory pressure?
- What services run in this container? No process listing was available to determine this.
</code></pre></div></div>

<p>The exact issue (procps is missing from the container) and questions don’t matter, but we have a workflow which can be invoked via cron. We could even push to a (private) GH repo so you can interact via GitHub.</p>

<p>I look after a number of CI workers… could this be extended to monitor those machines? We should acknowledge the risk here, but CI workers are ephemeral, and their cache is typically in tmpfs, so there is very little to lose and I auto reinstallation over the network.</p>

<p>I made the Dockerfile multistage by first building the <a href="https://github.com/ocurrent/ocluster">ocurrent/ocluster</a> admin tools, and then copying them from the final stage.</p>

<p>I prepopulated the Knowledge section with details on how to use <code class="language-plaintext highlighter-rouge">ocluster-admin</code> to list pools and query them, and how to interpret the results. Once I was happy that this was working and that sufficient knowledge had been gained through Knowledge and Questions. For example, how to pause a worker before performing actions!</p>

<p>I took things to the next stage. I generated a new SSH key in the project directory and deployed it to a few workers. I mounted the key into the container by adding <code class="language-plaintext highlighter-rouge">-v "$PWD/ssh:/home/mtelvers/.ssh"</code> along with an <code class="language-plaintext highlighter-rouge">ssh_config</code> file. Then I added an additional step to the instructions: “SSH into workers or run commands to diagnose any problems found. Action outstanding items wherever possible – fix anything you can.”</p>

<p>Now, when a run finds a new issue, Claude actually debugs it and attempts to resolve it. Questions are still raised and answered in the same way.</p>

<p>This is an interesting experiment about how to interact with an AI agent beyond the prompt. The agent isn’t answering questions or generating code for a human to review. It’s executing a runbook against live systems, making judgment calls about what to fix, and maintaining its own operational documentation.</p>

<p>The git repo is the interface. I push instructions by editing NOTES.md. The agent pushes results by committing updates. It’s version-controlled, auditable, asynchronous communication between a human and an AI agent about shared infrastructure.</p>

<p>Full disclosure: I manually invoke the script and monitor the output. I haven’t yet let it run via cron, but I also haven’t had to Control-C it either.</p>
