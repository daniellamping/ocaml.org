---
title: What Happened in January 2026?
description: My writeup about tinkerbell sparked some discussions, I finally gave
  in to the code agent hype (on my own terms), and this website got new RSS feeds.
url: https://soap.coffee/~lthms/posts/january-2026.html
date: 2026-01-30T00:00:00-00:00
preview_image: https://soap.coffee/~lthms/img/avatar2026.jpg
authors:
- "Thomas Letan\u2019s Blog"
source:
ignore:
---


        
        <h1>What Happened in January 2026?</h1><div><span class="icon"><svg><use href="/~lthms/img/icons.svg#tag"></use></svg></span>&nbsp;<a href="https://soap.coffee/~lthms/tags/meta.html" class="tag hover-sky" marked="">meta</a> <span class="icon"><svg><use href="/~lthms/img/icons.svg#tag"></use></svg></span>&nbsp;<a href="https://soap.coffee/~lthms/tags/vibecoding.html" class="tag hover-lavender" marked="">vibecoding</a> </div>
<p>This is my first retrospective in quite a while—I have yet to make writing
these logs a real habit of mine. That being the case, so much happened this
month that this article felt like the obvious thing to do.</p>
<h2><code class="hljs">tinkerbell</code>’s First Month</h2>
<p>On January 5th, I published <a href="https://soap.coffee/~lthms/posts/i-cannot-ssh-into-my-server-anymore.html" class="hover-rose" marked="">my account of migrating my website to a completely
new setup</a>. Not only am I very
proud of this article in and of itself, I am also quite happy that it has
sparked discussions in a few places such as <a href="https://lobste.rs/s/e7lpyy/i_cannot_ssh_into_my_server_anymore_s_fine" class="hover-coral" marked="">lobste.rs&nbsp;<span class="icon"><svg><use href="/~lthms/img/icons.svg#external-link"></use></svg></span></a> or the <a href="https://news.ycombinator.com/item?id=46524390" class="hover-mint" marked="">orange
website&nbsp;<span class="icon"><svg><use href="/~lthms/img/icons.svg#external-link"></use></svg></span></a>. It has also been shared in other places like the <a href="https://buttondown.com/devopsish/archive/devopsish-293/" class="hover-periwinkle" marked="">DevOps’ish
newsletter&nbsp;<span class="icon"><svg><use href="/~lthms/img/icons.svg#external-link"></use></svg></span></a><label for="fn1" class="sidenote-number margin-toggle"></label><input type="checkbox" class="margin-toggle"><span class="note-right sidenote note"><span class="footnote-p">I have also created a <a href="https://github.com/lthms/tinkerbell" class="hover-coral" marked="">GitHub repository&nbsp;<span class="icon"><svg><use href="/~lthms/img/icons.svg#github"></use></svg></span></a> to host <code class="hljs">tinkerbell</code>
configuration, and it is quietly farming stars as we speak. </span>
</span>.</p>
<p>I don’t write articles with the expectation that they reach a large audience,
but I will admit I had <em>some</em> ambitions for this one. It’s very fulfilling to
know my experiment report of sorts has caught the attention of my peers, if
only a little.</p>
<p>At first, I thought I would spend January experimenting with my new playground.
I have a handful of services I want to deploy. Little did I know I would be
sidetracked sharply before I had a chance to get to it.</p>
<h2>Meeting with Claude Code</h2>
<p>Somehow, I have managed to go through 2025 while staying afar from the code
agent hype. I gave “vibecoding” a try in May by <a href="https://soap.coffee/~lthms/posts/PeerProgrammingWithLLMs.html" class="hover-peach" marked="">chatting with ChatGPT and
Gemini while yak shaving my way towards transcribing YouTube videos in
OCaml</a>. Even I realized back then that the hype had already moved on from
chat UI towards agents.</p>
<p>Then, the craziest thing happened. I went on holidays on December 23rd, 2025. Two
weeks later, as I was going back to <code class="hljs language-bash"><span class="hljs-variable">$WORK</span></code>, Claude Code and its ilk
suddenly felt inevitable. It looks like agentic workflows got very good in a
matter of a few weeks, up to a point where there is a real opportunity cost in
ignoring them altogether. And as I am about to take out a mortgage, it feels
like a bad time to take the risk of falling out of relevance.</p>
<p>Looking back, I think it started with a simple, genuine suggestion—we were
discussing a fairly ambitious clean-up of our test-suite at <code class="hljs language-bash"><span class="hljs-variable">$WORK</span></code> when
someone mentioned Claude Code was quite relevant for this kind of tedious,
boilerplate-heavy task. This resonated with all the success stories I was
suddenly exposed to. Before I could realize what was happening, I was caught up
in <a href="https://soap.coffee/~lthms/posts/how-i-want-to-use-llms-in-2026.html" class="hover-lavender" marked="">an intense introspective journey</a>.</p>
<p>Part of my answer to the angst of LLMs being on the verge of reshaping the way
I work is <a href="https://lthms.github.io/cece/" class="hover-periwinkle" marked="">CeCe&nbsp;<span class="icon"><svg><use href="/~lthms/img/icons.svg#github"></use></svg></span></a><label for="fn2" class="sidenote-number margin-toggle"></label><input type="checkbox" class="margin-toggle"><span class="note-left sidenote note"><span class="footnote-p">CeCe being a nickname for Claude Code (CC). </span>
</span>.</p>
<p>CeCe is a Claude Code plugin <s>I have been working</s> Claude Code and I have
been working on for the better part of the month, and it has become my sandbox
to experiment with and understand agentic workflows. Configuring Claude Code in
a very opinionated way felt only natural—that’s the only starting point I know.
I am glad I did that, it enabled me to get a better understanding of how Claude
Code actually works.</p>
<p>Ironically, since CeCe has mostly been written by CeCe itself and because it
grew quite fast as a result, I do not really feel confident in modifying it
myself 😆. I am now trying to <a href="https://github.com/lthms/vee" class="hover-lemon" marked="">rewrite it&nbsp;<span class="icon"><svg><use href="/~lthms/img/icons.svg#github"></use></svg></span></a> on my
own, with more structure and “intention.” We will see how it goes.</p>
<h2>New RSS Feeds</h2>
<p>On a final note, I am glad to say that this website now features some new RSS
feeds that you may be interested in<label for="fn3" class="sidenote-number margin-toggle"></label><input type="checkbox" class="margin-toggle"><span class="note-right sidenote note"><span class="footnote-p">That’s actually the very first task I entrusted to Claude Code.
Suffice it to say, it didn't blink, and navigated through my quite unusual
setup like a champ. </span>
</span>: one per <a href="https://soap.coffee/~lthms/tags" class="hover-peach" marked="">tags</a>, and one per
<a href="https://soap.coffee/~lthms/series" class="hover-periwinkle" marked="">series</a>. I have been meaning to implement them since I’ve added my website to
a certain aggregator that now advertises every article I publish that cites one
particular programming language.</p>
<p>I am glad to know that camel aficionados learnt about my <code class="hljs">tinkerbell</code> setup
simply because I drew a parallel between infrastructure as code and functional
programming. Maybe they don’t share my enthusiasm, though. With this change, I
am one PR away from limiting this aggregator’s scope to simply the articles
featuring the appropriate tag. I think they will find it useful. Maybe you will
too!</p>
        
      
