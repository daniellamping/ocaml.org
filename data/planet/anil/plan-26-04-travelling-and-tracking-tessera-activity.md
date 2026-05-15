---
title: '.plan-26-04: Travelling and tracking TESSERA activity'
description: Tracking TESSERA activity including a new preprint and podcast, wrestling
  with Zarr, and saying farewell to David Allsopp.
url: https://anil.recoil.org/notes/2026w4
date: 2026-01-25T00:00:00-00:00
preview_image: https://anil.recoil.org/images/faces/avsm.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>Just short weeknotes this week as I've been travelling back to Belfast for various matters and haven't had much computer time.</p>
<p>One exciting development in the week was that <a href="https://shaneweisz.com">Shane Weisz</a> continued the conversation with the <a href="https://www.shaneweisz.com/blog/presenting-ai-for-the-red-list-to-iucn">IUCN Red list team</a> about his developing dashboard, which went extremely well. There's so much excitement on both sides about how all this is going!</p>
<h2><a href="https://anil.recoil.org/news.xml#tessera-developments" class="anchor" aria-hidden="true"></a>TESSERA developments</h2>
<p>I also spent a chunk of time wrestling with understanding <a href="https://zarr-specs.readthedocs.io/en/latest/v3/core/index.html">Zarr</a> so I can port <a href="https://anil.recoil.org/notes/geotessera-python">TESSERA</a> to use this instead of Numpy arrays. There's been <a href="https://eeg.zulipchat.com/#narrow/channel/527258-Tessera/topic/zarr.20file.20format/with/571006960">a long and helpful thread</a> on our Zulip about this with a lot of people chiming in. <a href="https://mynameismwd.org">Michael Dales</a> has also been a <a href="https://digitalflapjack.com/weeknotes/2025-12-15">source of coordinate inspiration</a> from his experiences, so I'll put proper thoughts together on this soon.</p>
<p>On the OxCaml front, <a href="https://www.tunbury.org/">Mark Elvers</a> has knocked up some <a href="https://github.com/mtelvers/ocaml-zarr">OCaml Zarr</a> so I'll be porting those to OxCaml soon and taking the TESSERA support in OCaml for a spin.</p>
<h3><a href="https://anil.recoil.org/news.xml#tessera-activity-around-the-web" class="anchor" aria-hidden="true"></a>TESSERA activity around the web</h3>
<p>There was quite a lot of TESSERA things going on alongside this.</p>
<p>First a preprint <a href="https://arxiv.org/abs/2601.13134">Earth Embeddings as Products: Taxonomy, Ecosystem, and Standardized Access</a> that's a nice summary of how to use geoembeddings like TESSERA:</p>
<blockquote>
<p>Geospatial Foundation Models (GFMs) provide powerful representations, but
high compute costs hinder their widespread use. Pre-computed embedding data
products offer a practical "frozen" alternative, yet they currently exist in
a fragmented ecosystem of incompatible formats and resolutions. This lack of
standardization creates an engineering bottleneck that prevents meaningful
model comparison and reproducibility. We formalize this landscape through a
three-layer taxonomy: Data, Tools, and Value. We survey existing products to
identify interoperability barriers. To bridge this gap, we extend TorchGeo
with a unified API that standardizes the loading and querying of diverse
embedding products. By treating embeddings as first-class geospatial
datasets, we decouple downstream analysis from model-specific engineering,
providing a roadmap for more transparent and accessible Earth observation
workflows.</p>
</blockquote>
<p>Then <a href="https://toao.com">Sadiq Jaffer</a> and <a href="https://www.cst.cam.ac.uk/people/zf281">Frank Feng</a> did a great podcast on <a href="https://www.satellite-image-deep-learning.com/p/tessera-a-temporal-foundation-model">Satellite Image Deep learning</a> where they go through the journey of how we trained the model. I can't believe it's barely been a year!</p>
<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/10CBuGfrz6M?si=FAIWvnfIOPaEkGwn" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen=""></iframe>
<p>TESSERA support also got merged into <a href="https://github.com/torchgeo/torchgeo/pull/3243">Torchgeo</a> for those looking to work on customising the model itself. Most users don't have to use this as they can just use our pregenerated embeddings.</p>
<h2><a href="https://anil.recoil.org/news.xml#a-fond-farewell-to-dra27-from-the-lab" class="anchor" aria-hidden="true"></a>A fond farewell to dra27 from the Lab</h2>
<p>After almost a <a href="https://anil.recoil.org/papers/2017-oud-platform">decade</a> of working with <a href="https://www.dra27.uk">David Allsopp</a> in both the University and later <a href="https://anil.recoil.org/notes/founded-tarides">Tarides</a>, he finally "graduated" and went off to <a href="https://www.dra27.uk/blog/platform/2026/01/19/plotting-a-new-course.html">join Jane Street</a> where...we will continue to work together on OxCaml and OCaml.</p>
<p>Good luck to David as he no doubt enjoys the ridiculously nice Jane Street office, where I would overdose on the fresh fruit juice machine and be on a perpetual sugar high!</p>
<p>Some fun links:</p>
<ul>
<li>It was nice to see others getting excited about my <a href="https://bsky.app/profile/oppi.li/post/3mcjcygf3r227">OCaml ATProto client support</a></li>
<li>And also <a href="https://bsky.app/profile/apenwarr.ca/post/3mci727zgxk2s">WebFinger seems more important</a> so I implemented that too.</li>
</ul><h1>References</h1><ul><li>Madhavapeddy (2025). GeoTessera Python library released for geospatial embeddings. <a href="https://doi.org/10.59350/7hy6m-1rq76" target="_blank"><i>10.59350/7hy6m-1rq76</i></a></li>
<li>Fang et al (2026). Earth Embeddings as Products: Taxonomy, Ecosystem, and Standardized Access. arXiv. <a href="https://doi.org/10.48550/arXiv.2601.13134" target="_blank"><i>10.48550/arXiv.2601.13134</i></a></li></ul>
