---
title: Weeknotes 2026 week 9
description:
url: https://jon.recoil.org/blog/2026/03/weeknotes-2026-09.html
date: 2026-03-02T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#weeknotes-2026-week-9" class="anchor"></a>Weeknotes 2026 week 9</h1>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2026-03-02</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/weeknotes.html" title="weeknotes">weeknotes</a> <a href="https://jon.recoil.org/tags/odoc.html" title="odoc">odoc</a> <a href="https://jon.recoil.org/tags/plugins.html" title="plugins">plugins</a></p></li></ul>
<ul class="at-tags"><li class="notanotebook"><span class="at-tag">notanotebook</span> </li></ul>
<p>Let's make this really terse!</p>
<h2><a href="https://jon.recoil.org/atom.xml#what-did-i-do?" class="anchor"></a>What did I do?</h2>
<ul><li>Got docs working with github actions on Anil's oxmono monorepo. Results are <a href="https://jonludlam.github.io/oxmono/">here</a>. This includes experimental support for oxcaml modes/layouts.</li><li><p>Got markdown mode output into Sherlodoc's db so you can query it - great for agents!</p><div><a href="https://jon.recoil.org/search.png" class="img-link"><img src="https://jon.recoil.org/search.png" alt="search.png"></a></div></li><li><p>Widgets in the JS OCaml toplevels - using FRP for the interactions. The neat thing here is that using FRP via Daniel Bunzli's <a href="https://erratique.ch/software/note">note</a> library is that all the interactions are all purely functional, no refs or mutables in sight. You provide a little wrapper scripts that's run in the frontend and the interactions and send back and forth with the worker running the code where it's translated into Events and Signals. My proof-of-concept of this is a widget that works with the <a href="https://leafletjs.com/">leaflet.js</a> library:</p><div><video src="mapdemo.mov" controls="controls" aria-label="mapdemo.mov"></video></div><p>Demo coming soon!</p></li><li>Consolidating all of the Odoc toplevel bits and pieces into the one monorepo. Again, demo of this coming soon!</li></ul>
<h2><a href="https://jon.recoil.org/atom.xml#what-am-i-going-to-do?" class="anchor"></a>What am I going to do?</h2>
<ul><li>New website!</li><li>Odoc plugins showcase</li><li>Writing writing writing writing</li></ul>
