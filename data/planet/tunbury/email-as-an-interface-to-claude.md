---
title: Email as an interface to Claude
description: In my previous post, I described running Claude Code as a non-interactive
  agent by feeding it a runbook via NOTES.md, letting it SSH into workers, diagnose
  problems, and commit its findings back to git.
url: https://www.tunbury.org/2026/03/26/email-autoresponder/
date: 2026-03-26T20:00:00-00:00
preview_image: https://www.tunbury.org/images/anthropic-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>In my <a href="https://www.tunbury.org/2026/03/18/interact-with-claude/">previous post</a>, I described running Claude Code as a non-interactive agent by feeding it a runbook via <code class="language-plaintext highlighter-rouge">NOTES.md</code>, letting it SSH into workers, diagnose problems, and commit its findings back to git.</p>

<p>That works well for scheduled tasks, but what if you are out shopping and someone sends a message which requires urgent attention? You now want to be at your desk with all your normal tools available. So I built an email autoresponder backed by Claude Code: send an email to <code class="language-plaintext highlighter-rouge">your-claude-bot@gmail.com</code> with a question like “check disk space on server-1”, and Claude processes it, runs the commands, and emails you back.</p>

<h2>The architecture</h2>

<p>The autoresponder is a single OCaml binary that polls an IMAP mailbox, processes new messages through Claude Code running in Docker, and sends replies via SMTP. No mail server infrastructure required beyond an email account. It works with Gmail, Fastmail, or any provider with IMAP/SMTP.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>  Inbound email
       |
       v
  IMAP client (poll for UNSEEN)
       |
       v
  S/MIME verification --&gt; reject unsigned/untrusted
       |
       v
  Strip noise (quotes, signatures, session tokens)
       |
       v
  Session lookup (resume or create)
       |
       v
  Claude Code (docker run ... claude --output-format json -p "...")
       |
       v
  SMTP client (send reply with session token)
</code></pre></div></div>

<p>Session continuity is handled by embedding a <code class="language-plaintext highlighter-rouge">[Session: uuid.hmac]</code> token in the reply. When you reply to that email, the token routes your follow-up to the same Claude session via <code class="language-plaintext highlighter-rouge">--resume</code>, so context carries across the conversation. Thus, you can ask about a server in one message and have the context carry over, so a follow-up question doesn’t need to reference the server a second time.</p>

<h2>The security problem</h2>

<p>An email autoresponder creates a serious security risk, and the more access keys you provide to Claude, the higher the risk, but the more useful the service would be.</p>

<p>Sender allow lists are trivially bypassed as spoofing an email From header is trivial and offers no authentication whatsoever. SPF and DKIM help at the domain level, but don’t prevent a compromised account or a determined attacker.</p>

<p>A possible solution is S/MIME. Apple Mail has built-in support for signing emails with X.509 certificates. The autoresponder can verify the PKCS#7 signature against a pinned certificate before processing. Emails with an invalid signature are silently dropped. I’ve used <code class="language-plaintext highlighter-rouge">openssl cms -verify</code> with <code class="language-plaintext highlighter-rouge">-partial_chain</code> for self-signed certificate support:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">verify_raw_message</span> <span class="o">~</span><span class="n">trusted_certs</span> <span class="n">raw_message</span> <span class="o">=</span>
  <span class="c">(* ... write to temp file ... *)</span>
  <span class="k">let</span> <span class="n">cert_args</span> <span class="o">=</span>
    <span class="nn">List</span><span class="p">.</span><span class="n">concat_map</span>
      <span class="p">(</span><span class="k">fun</span> <span class="n">cert</span> <span class="o">-&gt;</span> <span class="p">[</span> <span class="s2">"-certfile"</span><span class="p">;</span> <span class="n">cert</span><span class="p">;</span> <span class="s2">"-CAfile"</span><span class="p">;</span> <span class="n">cert</span> <span class="p">])</span>
      <span class="n">trusted_certs</span>
  <span class="k">in</span>
  <span class="k">let</span> <span class="n">args</span> <span class="o">=</span> <span class="nn">Array</span><span class="p">.</span><span class="n">of_list</span>
    <span class="p">([</span> <span class="s2">"openssl"</span><span class="p">;</span> <span class="s2">"cms"</span><span class="p">;</span> <span class="s2">"-verify"</span><span class="p">;</span> <span class="s2">"-in"</span><span class="p">;</span> <span class="n">msg_file</span><span class="p">;</span>
       <span class="s2">"-inform"</span><span class="p">;</span> <span class="s2">"SMIME"</span><span class="p">;</span> <span class="s2">"-purpose"</span><span class="p">;</span> <span class="s2">"any"</span><span class="p">;</span>
       <span class="s2">"-partial_chain"</span> <span class="p">]</span> <span class="o">@</span> <span class="n">cert_args</span><span class="p">)</span>
  <span class="k">in</span>
  <span class="c">(* ... *)</span>
</code></pre></div></div>

<p>This is certificate pinning, not CA chain validation. Only the specific certificate in <code class="language-plaintext highlighter-rouge">trusted_certs</code> is accepted. An attacker generating their own self-signed cert with the same email address is rejected. The verification requires that the signature be made with the private key corresponding to the pinned certificate.</p>

<p>S/MIME is enabled by default. You can disable it for testing with <code class="language-plaintext highlighter-rouge">"require_smime": false</code>, but the autoresponder logs a warning at startup if you do.</p>

<p>As in my previous Git solution, Claude is running in a Docker container. This allows hard limits on what Claude can do, even with <code class="language-plaintext highlighter-rouge">--dangerously-skip-permissions</code>; however, you might choose to allow limited SSH access to hosts or create a limited GitHub account.</p>

<p>Other mitigations are layered but imperfect:</p>

<ul>
  <li>Email noise is stripped before prompting; things like the quoted replies, signature separators, and session tokens are removed, so Claude only sees the new text</li>
  <li>Prompt length is capped at 100k characters</li>
  <li>Claude Code’s own safety mechanisms provide some resistance</li>
  <li>A <code class="language-plaintext highlighter-rouge">CLAUDE.md</code> in the working directory can establish operational boundaries</li>
</ul>

<h2>Session management</h2>

<p>Sessions are persisted to a JSON file and keyed by HMAC-signed tokens. The HMAC binds each token to the sender’s email address, so a token extracted from one conversation can’t be used by a different sender. Sessions expire after a configurable TTL (default 24 hours).</p>

<p>The session token’s primary purpose is session resumption, not authentication. It’s the mechanism by which a reply-to-reply chains back to the same Claude context.</p>

<h2>Running it</h2>

<p>The configuration is a single JSON file pointing at your email provider and Claude Code Docker image:</p>

<div class="language-json highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">{</span><span class="w">
  </span><span class="nl">"imap"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w">
    </span><span class="nl">"host"</span><span class="p">:</span><span class="w"> </span><span class="s2">"imap.example.com"</span><span class="p">,</span><span class="w">
    </span><span class="nl">"port"</span><span class="p">:</span><span class="w"> </span><span class="mi">993</span><span class="p">,</span><span class="w">
    </span><span class="nl">"username"</span><span class="p">:</span><span class="w"> </span><span class="s2">"claude@example.com"</span><span class="p">,</span><span class="w">
    </span><span class="nl">"password"</span><span class="p">:</span><span class="w"> </span><span class="s2">"app-password"</span><span class="w">
  </span><span class="p">},</span><span class="w">
  </span><span class="nl">"smtp"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w">
    </span><span class="nl">"host"</span><span class="p">:</span><span class="w"> </span><span class="s2">"smtp.example.com"</span><span class="p">,</span><span class="w">
    </span><span class="nl">"port"</span><span class="p">:</span><span class="w"> </span><span class="mi">465</span><span class="p">,</span><span class="w">
    </span><span class="nl">"username"</span><span class="p">:</span><span class="w"> </span><span class="s2">"claude@example.com"</span><span class="p">,</span><span class="w">
    </span><span class="nl">"password"</span><span class="p">:</span><span class="w"> </span><span class="s2">"app-password"</span><span class="w">
  </span><span class="p">},</span><span class="w">
  </span><span class="nl">"claude"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w">
    </span><span class="nl">"docker_image"</span><span class="p">:</span><span class="w"> </span><span class="s2">"bot"</span><span class="p">,</span><span class="w">
    </span><span class="nl">"work_dir"</span><span class="p">:</span><span class="w"> </span><span class="s2">"/path/to/workdir"</span><span class="p">,</span><span class="w">
    </span><span class="nl">"claude_dir"</span><span class="p">:</span><span class="w"> </span><span class="s2">"/home/you/.claude"</span><span class="p">,</span><span class="w">
    </span><span class="nl">"docker_args"</span><span class="p">:</span><span class="w"> </span><span class="p">[],</span><span class="w">
    </span><span class="nl">"extra_args"</span><span class="p">:</span><span class="w"> </span><span class="p">[]</span><span class="w">
  </span><span class="p">},</span><span class="w">
  </span><span class="nl">"hmac_secret"</span><span class="p">:</span><span class="w"> </span><span class="s2">"generate-a-random-hex-string"</span><span class="p">,</span><span class="w">
  </span><span class="nl">"allowed_senders"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="s2">"you@example.com"</span><span class="p">],</span><span class="w">
  </span><span class="nl">"reply_from"</span><span class="p">:</span><span class="w"> </span><span class="s2">"claude@example.com"</span><span class="p">,</span><span class="w">
  </span><span class="nl">"require_smime"</span><span class="p">:</span><span class="w"> </span><span class="kc">true</span><span class="p">,</span><span class="w">
  </span><span class="nl">"trusted_certs"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="s2">"/path/to/your/cert.pem"</span><span class="p">]</span><span class="w">
</span><span class="p">}</span><span class="w">
</span></code></pre></div></div>

<p>Generate a self-signed S/MIME certificate, import the <code class="language-plaintext highlighter-rouge">.p12</code> into Apple Mail (or your client of choice), and sign your outgoing emails. The autoresponder rejects anything unsigned.</p>

<p>The code is available on GitHub <a href="https://github.com/mtelvers/claude-autoresponder">mtelvers/claude-autoresponder</a>. Use at your own risk!</p>
