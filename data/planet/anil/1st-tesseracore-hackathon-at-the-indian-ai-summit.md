---
title: 1st TESSERA/CoRE hackathon at the Indian AI Summit
description: First TESSERA hackathon held at the Indian AI Impact Summit in Delhi,
  exploring integration with IIT-Delhi's CoRE Stack for geospatial analysis and testing
  TESSERA labeling workflows.
url: https://anil.recoil.org/notes/first-tessera-hackathon
date: 2026-02-19T00:00:00-00:00
preview_image: https://anil.recoil.org/images/tessera-hackathon-3.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>We held the first <a href="https://anil.recoil.org/projects/tessera">TESSERA</a> hackathon over the <a href="https://anil.recoil.org/notes/india-ai-summit">Indian AI Impact summit</a> in Delhi today, thanks to sponsorship
from <a href="https://openuk.uk">OpenUK</a>.  Despite only announcing it the weekend
before, we had a full house of active participants (some of whom came up all
the way from Bangalore for the day to attend!).</p>
<p>This is the first TESSERA hackathon I've taken part in outside the UK (where
we've had several really fun ones with the <a href="https://iucn.org">IUCN</a> and
<a href="https://unep-wcmc.org">UNEP-WCMC</a>).  It was especially packed because of the
massive buzz in Delhi with the summit going on; every hotel in the city was
booked out and traffic was gridlocked as 200,000 people descended.  We held the
hackathon in the lovely <a href="https://iicdelhi.in/">India International Centre</a>
campus in the centre of Delhi which was a peaceful locale amidst the chaos
outside.</p>
<p><img src="https://anil.recoil.org/images/tessera-hackathon-3.webp" alt="%c" title="The hackathon was fueled by marmalade biscuits supplied by Amanda Brock from OpenUK; the best kind of sponsorship!"></p>
<h2><a href="https://anil.recoil.org/news.xml#learning-about-core-stack" class="anchor" aria-hidden="true"></a>Learning about CoRE Stack</h2>
<p><img src="https://anil.recoil.org/images/tessera-hackathon-1.webp" alt="%rc" title="The CoRE stack in action">
I organised the hackathon with <a href="https://www.cse.iitd.ac.in/~aseth/">Aadi Seth</a> from IIT-Delhi, who leads the <a href="https://core-stack.org">CoRE Stack</a>. We first learnt about how the CoRE stack works; my notes follow:</p>
<ul>
<li>The <a href="https://github.com/core-stack-org/">GitHub core-stack-org</a> has the backend service, the <a href="https://github.com/core-stack-org/landscape-explorer">landscape explorer</a> web app, and various <a href="https://github.com/core-stack-org/cc-android-offline">mobile apps</a>.</li>
<li>They use <a href="https://earthengine.google.com/faq/">GEE</a> to perform the geospatial analyses and then export the rasters into their own pipeline, with a <a href="https://geoserver.org/">geoserver</a> instance to serve the site. Currently runs only for ROIs for projects, but would like to scale pan-India but depends on compute availability.
There's a Django, Celery, GEE, Geoserver, Airflow cloud flow, but we discussed <a href="https://yirgacheffe.org/latest/">Yirgacheffe</a> and a local machine <a href="https://digitalflapjack.com/blog/yirgacheffe/">like we do for LIFE</a> to simplify deployment. Concerns include having to manage on-prem resources (but need to balance this vs cost of cloud resources).</li>
<li>They have developed a <a href="https://www.spatialnode.net/articles/building-reproducible-geospatial-pipelines-a-stac-extension-with-dags25119d">dataflow extension to STAC</a> that extends STAC with lineage tracking, algorithm versioning, and incremental recomputation. This is of great <a href="https://www.tunbury.org/2025/11/30/tessera-zarr/">interest to us</a> as we deploy STAC for TESSERA too!  See <a href="https://watch.eeg.cl.cam.ac.uk/w/fEoa7jde33i35w1Xz816ft">PROPL talk</a>, <a href="https://dl.acm.org/doi/10.1145/3759536.3763803">paper</a> and <a href="https://anil.recoil.org/notes/icfp25-propl">my notes</a> on STAC-D.</li>
<li>The <a href="https://www.explorer.core-stack.org/">CoRE Stack Explorer</a> does more than just mapping; it also generates structured reports about items of concern such as hydrological flows, soil health and other indicators being tracked that are of relevance to the local people in the region. It's open access and browsable, and I enjoyed looking through projects in Andhra Pradesh!</li>
</ul>
<h2><a href="https://anil.recoil.org/news.xml#showing-tessera" class="anchor" aria-hidden="true"></a>Showing TESSERA</h2>
<p><img src="https://anil.recoil.org/images/tessera-hackathon-4.webp" alt="%rc" title="I must admit the other event going on here was also of interest!">
To get people going in TESSERA I first of all explained the basic ideas behind the model, on the same lines that <a href="https://svr-sk818-web.cl.cam.ac.uk/keshav/wiki/index.php/Main_Page">Srinivasan Keshav</a> and <a href="https://coomeslab.org">David Coomes</a> did in our <a href="https://anil.recoil.org/notes/foundational-ecosystem-workshop">Ecosystem Resilience Workshop</a>. The two main 'getting started' ways right now are:</p>
<ul>
<li>Clone the <a href="https://github.com/ucam-eo/tessera-interactive-map">tessera-interactive-map</a> repo and setup a pip or uv environment from the requirements.txt, and then open up <code>app.ipynb</code> in VSCode or Jupyter (as you prefer, but I tend to use VSCode as it works better with uv). Then pick an area of interest and try out the labeling workflow directly on your laptop.</li>
<li>Clone Keshav's <a href="https://github.com/sk818/TEE">TESSERA Embeddings Explorer</a> which is a much more feature-rich app but requires a bit more setup. I demonstrated this using the <a href="https://github.com/sk818/TEE/blob/main/docker-compose.yml">docker-compose</a> file and several people with <a href="https://anil.recoil.org/papers/2025-docker-icfp">Docker for Windows</a> got it up and running with a <code>docker compose up</code>.</li>
</ul>
<p>While this all went right, the wifi at the venue was pretty slow, so downloading embeddings all the way from Cambridge was pretty slow. I think there are two avenues we need to explore quickly:</p>
<ul>
<li>I want to prioritise switching the embeddings to Zarr, and there's a <a href="https://eeg.zulipchat.com/#narrow/channel/527258-Tessera/topic/zarr.20file.20format/with/571006960">comprehensive thread</a> on the EEG Zulip about how we can go about this. This is on next week's hacking queue, using the <a href="https://github.com/mtelvers/ocaml-zarr">ocaml-zarr</a> bindings that <a href="https://www.tunbury.org/">Mark Elvers</a> has put together!</li>
<li>We discussed mirroring embeddings for India to IIT Delhi's servers, which <a href="https://www.cse.iitd.ac.in/~aseth/">Aadi Seth</a> is going to investigate. I think this will also be easier once we have Zarr, since then a static webserver is all that's needed and even JavaScript clients could fetch the data they need.</li>
<li>A mobile phone app for TESSERA would be extremely cool, as Sadiq has <a href="https://toao.com/blog/can-we-really-see-brambles-from-space">observed before</a>.</li>
</ul>
<p>After this, we had a brief period to try out some labeling, but the conversations about exactly what we hack on together will continue on our EEG Zulip channel! There was a lot of discussion about how to store labels, as many of the groups there had ground truth information that they weren't quite sure how to share. I think it would be very valuable to have a general geospatial labeling service that could then export to various specific services like OpenStreetMap or for downstream training.</p>
<p>Thanks OpenUK and the Impact Summit for facilitating this, and for IIT-Delhi and the other participants for the fascinating discussions and hacking!</p>
<p><img src="https://anil.recoil.org/images/tessera-hackathon-2.webp" alt="%c" title="The hackers hacking!">
<img src="https://anil.recoil.org/images/tessera-hackathon-5.webp" alt="%c" title="The lovely IIC venue"></p><h1>References</h1><ul><li>Madhavapeddy et al (2025). Functional Networking for Millions of Docker Desktops. <a href="https://doi.org/10.1145/3747525" target="_blank"><i>10.1145/3747525</i></a></li>
<li>Madhavapeddy (2026). At the AI Impact Summit in Delhi: people, planet, progress. <a href="https://doi.org/10.59350/6vc5q-mbk23" target="_blank"><i>10.59350/6vc5q-mbk23</i></a></li>
<li>Madhavapeddy (2025). Programming for the Planet at ICFP/SPLASH 2025. <a href="https://doi.org/10.59350/hasmq-vj807" target="_blank"><i>10.59350/hasmq-vj807</i></a></li>
<li>Madhavapeddy (2025). Foundational AI for Ecosystem Resilience workshop. <a href="https://doi.org/10.59350/26hy6-rry61" target="_blank"><i>10.59350/26hy6-rry61</i></a></li>
<li>Laud et al (2025). STACD: STAC Extension with DAGs for Geospatial Data and Algorithm Management. <a href="https://doi.org/10.1145/3759536.3763803" target="_blank"><i>10.1145/3759536.3763803</i></a></li></ul>
