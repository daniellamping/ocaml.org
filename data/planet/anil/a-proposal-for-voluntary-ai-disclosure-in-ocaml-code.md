---
title: A Proposal for Voluntary AI Disclosure in OCaml Code
description: Proposing a voluntary, machine-readable AI content disclosure scheme
  for OCaml spanning opam packages, dune, and per-module attributes, aligned with
  the W3C AI Content Disclosure vocabulary.
url: https://anil.recoil.org/notes/opam-ai-disclosure
date: 2026-04-03T00:00:00-00:00
preview_image: https://anil.recoil.org/images/eu-ai-act-1.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>After my <a href="https://anil.recoil.org/notes/aoah-2025">December of agentic coding</a> sprint, I was left quite
<a href="https://marvinh.dev/blog/ddosing-the-human-brain/">frazzled</a> but also with a
practical problem. I've got two kinds of libraries: the ones I care about (and
handcraft), and the wild experiments that look perfectly formed but are in fact just
(well typed) slop. After <a href="https://anil.recoil.org/notes/claude-copilot-sandbox">a year</a> of doing this, it's obvious that the <em>quality</em> of generated code also varies dramatically as
models steadily improve and agentic harnesses improve context management.</p>
<p>This post is about an <strong><a href="https://github.com/avsm/ocaml-ai-disclosure">ocaml-ai-disclosure proposal</a></strong> I put together to help track this in OCaml using metadata and <a href="https://ocaml.org/manual/5.3/attributes.html">extension attributes</a> in source code.</p>
<h2><a href="https://anil.recoil.org/news.xml#the-eu-is-mandating-what-this-summer" class="anchor" aria-hidden="true"></a>The EU is mandating what this summer?!</h2>
<p>Toby Jaffey pointed
me to the <a href="https://www.w3.org/community/ai-content-disclosure/">W3C AI Content Disclosure</a>
<a href="https://anil.recoil.org/notes/2026w13">last week</a>. The bit that
properly surprised me was a legal snippet buried in their README:</p>
<blockquote>
<p>The EU AI Act Article 50 (effective August 2026) requires that AI-generated text content be "marked in a machine-readable format and detectable as artificially generated or manipulated."
<cite>-- <a href="https://github.com/dweekly/ai-content-disclosure?tab=readme-ov-file">ai-content-disclosure</a>, David E. Weekly, 2026</cite></p>
</blockquote>
<p>This summer!!! Whether source code falls under "text content" is an <a href="https://eur-lex.europa.eu/eli/reg/2024/1689/oj">open
question</a> that hasn't been
addressed in existing legal commentary as far as I can tell (nor can I read the
raw 300+ pages to figure it out for myself).  However, regardless of how lawyers eventually
parse this, voluntary disclosure for code seems like a sensible thing to do anyway.</p>
<p>I've therefore put together an <strong><a href="https://github.com/avsm/ocaml-ai-disclosure">ocaml-ai-disclosure</a></strong> repository contains a draft specification and OCaml reference tooling for voluntary, machine-readable AI content disclosure in OCaml code. I'm interested in both thoughts from the OCaml community but also from other language ecosystems. Weirdly, I can't find a single other programming language that's proposed anything for source code after some searching.</p>
<p><a href="https://eur-lex.europa.eu/eli/reg/2024/1689/oj"> <img src="https://anil.recoil.org/images/eu-ai-act-1.webp" alt="%c" title="Not even reading the AI Act in my mothertongue shed light on the matter. (Ok ok, it's about laying down harmonised rules on AI and amending existing Regulations)"> </a></p>
<h2><a href="https://anil.recoil.org/news.xml#ai-disclosure-for-ocaml-is-pretty-easy" class="anchor" aria-hidden="true"></a>AI Disclosure for OCaml is pretty easy</h2>
<p>The OCaml ecosystem's accumulating code with varying degrees of AI involvement, but currently no machine-readable way to signal it. We obviously need to be very careful about how we mix this code into the <a href="https://github.com/ocaml/opam-repository">commons</a>, because the usual social signals we use to review packages are basically useless now.</p>
<p>However a binary AI "yes/no" flag doesn't capture the reality of how people actually work with these tools. The code I wrote during <a href="https://anil.recoil.org/notes/aoah-2025">AoAH</a> ranged from a one-shot <em>"CC generated the whole module from a one-line prompt"</em> to <em>"I wrote the core logic by hand and Claude sorted the pretty-printer boilerplate"</em> or even <em>"<a href="https://toao.com/blog/check-with-gemini">I got CC to test with Gemini</a>"</em>.</p>
<p>My proposal is extremely simple, here's how it works...</p>
<h3><a href="https://anil.recoil.org/news.xml#package-disclosures" class="anchor" aria-hidden="true"></a>Package Disclosures</h3>
<p>An opam package can declare its disclosure using extension fields:</p>
<pre><code>x-ai-disclosure: "ai-assisted"
x-ai-model: "claude-opus-4-6"
x-ai-provider: "Anthropic"
</code></pre>
<p>Note: This may just become a list of values in the final proposal, but you get the idea.</p>
<h3><a href="https://anil.recoil.org/news.xml#ocaml-module-level" class="anchor" aria-hidden="true"></a>OCaml Module level</h3>
<p>OCaml supports extension attributes, which we use via a floating attribute that applies to the entire compilation unit:</p>
<pre><code class="language-ocaml">[@@@ai_disclosure "ai-generated"]
[@@@ai_model "claude-opus-4-6"]
[@@@ai_provider "Anthropic"]

let foo = ...
let bar = ...
</code></pre>
<p>These can also be scoped more finely via declaration attributes that apply to a single binding:</p>
<pre><code class="language-ocaml">[@@@ai_disclosure "ai-assisted"]

let human_written x = ...

let ai_helper y =
  ...
[@@ai_disclosure "ai-generated"]
</code></pre>
<p>Disclosure follows a nearest-ancestor inheritance model like the W3C HTML proposal, whereby an explicit annotation overrides the inherited value.</p>
<p>One detail I'm quite pleased with is that <code>.mli</code> and <code>.ml</code> files are annotated independently, which means that one workflow I use quite a bit of writing the interface files first can be tracked separately from the implementations themselves.</p>
<h3><a href="https://anil.recoil.org/news.xml#the-disclosure-vocabulary" class="anchor" aria-hidden="true"></a>The disclosure vocabulary</h3>
<p>I use the same four levels as the W3C vocabulary, which works well enough for HTML:</p>
<div role="region"><table>
<tbody><tr>
<th>Value</th>
<th>Meaning</th>
</tr>
<tr>
<td><code>none</code></td>
<td>No AI involvement</td>
</tr>
<tr>
<td><code>ai-assisted</code></td>
<td>Human-authored, AI edited or refined</td>
</tr>
<tr>
<td><code>ai-generated</code></td>
<td>AI-generated with human prompting and review</td>
</tr>
<tr>
<td><code>autonomous</code></td>
<td>AI-generated without human oversight</td>
</tr>
</tbody></table></div><p>I treat the absence of annotation as "unknown", not "none". The <code>none</code> value exists for authors who <em>want</em> to positively assert human authorship, perhaps because their project's policy requires it or because they want reviewers to know this particular module was deliberately hand-written. Tools may also choose to spelunk back through pre-2022 code and add <code>none</code> automatically where it's obvious.</p>
<p>If a module contains both human-written and AI-generated bits, you can annotate
at the package level and add overrides directly in code.  OCaml's module system
and attributes gives us a natural hierarchy for this.</p>
<h3><a href="https://anil.recoil.org/news.xml#model-provenance" class="anchor" aria-hidden="true"></a>Model provenance</h3>
<p>Each annotation can also optionally carry provenance metadata:</p>
<ul>
<li><code>ai_model</code> (the API model identifier, like <code>claude-opus-4-6</code> or <code>gpt-4o</code>)</li>
<li><code>ai_provider</code> (like <code>Anthropic</code> or <code>OpenAI</code>).</li>
</ul>
<p><a href="https://mynameismwd.org">Michael Dales</a> pointed out it's quite common to use multiple models (e.g. to cross
test), so these attributes can be repeated when multiple models contributed.</p>
<h2><a href="https://anil.recoil.org/news.xml#the-programmer-burden-is-minimal" class="anchor" aria-hidden="true"></a>The programmer burden is minimal</h2>
<p>The nice thing about this proposal is that there's <em>no</em> overhead to a programmer that chooses not to use AI assistance.</p>
<p>For those that do, I've got a <a href="https://github.com/avsm/ocaml-claude-marketplace/blob/main/plugins/ocaml-dev/skills/ai-disclosure/SKILL.md">Claude Skill ocaml-dev:ai-disclosure</a>
that instructs the agent to add the right annotations in.  So when Claude
generates OCaml code in my sessions, it now inserts the attributes and also
maintains the <code>.opam.template</code> files.</p>
<p>During code review, I read the AI-generated code and edit away to (hopefully) improve it, and downgrade <code>ai-generated</code> to <code>ai-assisted</code> on the way.  If I've substantially rewritten the code then I just remove the annotation and fully claim it.</p>
<p>The key principle is that disclosure reflects the <em>current state of the code</em> to make it easier for a human to claim responsibility. A human who has thoroughly reviewed, understood, and rewritten a piece of code may reasonably call it their own. This is not my legal opinion, just a moral, informal and pragmatic one!</p>
<h2><a href="https://anil.recoil.org/news.xml#what-this-isnt" class="anchor" aria-hidden="true"></a>What this isn't</h2>
<p>A few things worth being explicit about after discussions around <a href="https://anil.recoil.org/projects/oxcaml">my group</a> on the matter:</p>
<ul>
<li>
<p>It's not a judgement on whether AI code is good or bad. The goal is a transparent, machine-readable signal so that consumers of the code (be they humans, puppies, licence checkers, package managers, CI systems, whatever) can apply their own policies.</p>
</li>
<li>
<p>We don't use git for this. A human may commit AI-generated code, or an AI agent may commit code that was human-reviewed and hacked and slashed enough to be considered rewritten before the commit. Rebases and squash also destroy attribution based on commits. Source-level attributes survive all these operations.</p>
</li>
<li>
<p>It's not mandatory. The whole point is voluntary adoption. I have noticed a vague reluctance from the people I've talked to to declare, as they'll feel they're being judged. If the OCaml community decides this is useful, adoption will happen naturally. If not, then it'll just be me using it and I'm fine with that!</p>
</li>
</ul>
<h2><a href="https://anil.recoil.org/news.xml#whats-next" class="anchor" aria-hidden="true"></a>What's next</h2>
<p>I'm starting by integrating this into my own <a href="https://anil.recoil.org/notes/aoah-2025">libraries</a> as a test bed. The Claude Code <a href="https://github.com/avsm/ocaml-claude-marketplace">marketplace skill</a> is already available if you want to try the automated annotation in your own sessions.</p>
<p>On the tooling side, there are several integration points I'd like to see if this idea has legs:</p>
<ul>
<li>odoc could render disclosure metadata alongside module documentation, perhaps using <a href="https://jon.recoil.org/blog/2026/03/weeknotes-2026-13.html">the odoc plugin</a> system that <a href="https://jon.recoil.org">Jon Ludlam</a> has been designing.</li>
<li>merlin or ocaml-lsp could surface disclosure attributes in hover information in the IDE, giving you a quick 'trust signal' while reading other people's code.</li>
<li>dune could gain native support for the <code>(ai_disclosure)</code> stanza to make the opam file generation easier.</li>
<li>opam could eventually use disclosure fields during version solving. I think it'd be useful to have a solver constraint that prefers packages with human-reviewed code where available, and only fall back to AI if nothing else works.</li>
</ul>
<p>The full draft specification, FAQ, and reference implementation are at <strong><a href="https://github.com/avsm/ocaml-ai-disclosure">github.com/avsm/ocaml-ai-disclosure</a></strong>.
I'd love feedback on the spec. File issues on the repo or in the <a href="https://discuss.ocaml.org/t/a-proposal-for-voluntary-ai-disclosure-in-ocaml-code/17950">OCaml Discussion thread</a>.</p><h1>References</h1><ul><li>Madhavapeddy (2026). .plan-26-13: Oxidised, standardised, and syndicated. <a href="https://doi.org/10.59350/ddx61-wd948" target="_blank"><i>10.59350/ddx61-wd948</i></a></li>
<li>Madhavapeddy (2025). Oh my Claude, we need agentic copilot sandboxing right now. <a href="https://doi.org/10.59350/aecmt-k3h39" target="_blank"><i>10.59350/aecmt-k3h39</i></a></li></ul>
