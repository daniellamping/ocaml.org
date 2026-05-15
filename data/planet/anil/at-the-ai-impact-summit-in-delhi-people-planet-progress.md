---
title: 'At the AI Impact Summit in Delhi: people, planet, progress'
description: Trip report from the Indian AI Impact Summit in New Delhi, covering the
  massive expo, a conversation with Yann LeCun, a hackathon/talk at IIT-Delhi, networking
  at the British High Commission, and reflections on the summit declaration's shift
  from safety to progress and equitable access.
url: https://anil.recoil.org/notes/india-ai-summit
date: 2026-02-21T00:00:00-00:00
preview_image: https://anil.recoil.org/images/aisummit-gen-2.640.webp
authors:
- Anil Madhavapeddy
source:
ignore:
---

<p>Very little sleep this week as I hopped over to India's <a href="https://impact.indiaai.gov.in/">AI Impact Summit</a> for a slew of events to followup my <a href="https://anil.recoil.org/notes/path-to-uk-india-ai-summit">earlier meetings at OpenUK and the Turing</a>.  The Indian government knocked it out of the park with the first summit held in the majority world: there were over 200,000 people registered and the keynote for <a href="https://sarvam.ai">Sarvam AI</a>'s launch had more people attending than the entire French AI summit last year! The venue was the enormous <a href="https://en.wikipedia.org/wiki/Bharat_Mandapam">Bharat Mandapam</a>, which was opened just a couple of years ago in the G20.</p>
<p>I found the summit a fantastic networking event, although unlikely to result in any significant policy shifts aside from establishing India as a serious target region for growth. I was lucky enough to meet Yann LeCun and get a bunch of technical insights into <a href="https://anil.recoil.org/projects/tessera">TESSERA</a> from him, which was my personal highlight!</p>
<p><img src="https://anil.recoil.org/images/aisummit-gen-2.webp" alt="%c" title="The glorious view from the Bharat Mandapam roof"></p>
<h2><a href="https://anil.recoil.org/news.xml#the-main-ai-summit-expo" class="anchor" aria-hidden="true"></a>The main AI Summit expo</h2>
<p><img src="https://anil.recoil.org/images/aisummit-gen-3.webp" alt="%rc" title="The map of the global arena, just one of the halls">
<img src="https://anil.recoil.org/images/aisummit-gen-5.webp" alt="%rc" title="The overall map of the whole summit with all the halls">
Through a series of misadventures not suitable for the public web, I ended up
in the rooftop VIP section of the summit and got to take in the breathtaking
views of the whole crowd. The size of the conference centre is incredible, with a
set of conference halls all connected by a lovely garden through to the gates
into Delhi proper.
The main expo I visited was the "world hall", where every country had an
exhibition (some projects were more <a href="https://www.bbc.co.uk/news/articles/c0q3g0ln274o">credible</a> than
<a href="https://www.bbc.co.uk/news/articles/cge8nd5ve00o">others</a>).</p>
<p></p><div class="video-vertical"><iframe title="Wandering around the Indian Impact AI Summit" src="https://crank.recoil.org/videos/embed/cba9ce1c-ff77-469c-bc8a-daa0308606c7" frameborder="0" allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups allow-forms" style="aspect-ratio: 9/16; width: 100%; height: 100%; max-width: 325px;"></iframe></div>
There were corridor conversations everywhere; an Nvidia
contingent spotted my Docker hoodie and started talking about containers and
GPUs; and some students saw my Jane Street t-shirt and thought I worked there
(I did try not to disappoint them by explaining 'why OCaml' though). I pulled
out my laptop a few times while lingering and started demonstrating TESSERA
(which looks cool due to the maps and false colours), and I had small crowds
watching along. I wish Cambridge had a proper presence at the 'GREAT Britain'
stall, but I only saw Dundee and (I think) Coventry represented there.<p></p>
<p>I didn't see any really mindblowing demos in the stalls, but perhaps that's
because my baselines have shifted in the last year. What was remarkable
was the breadth of solutions on offer: pretty much every aspect of Indian
society seemed to be covered, from urban living to rural food security.</p>
<p>There was little '<a href="https://en.wikipedia.org/wiki/Eating_your_own_dog_food">eat our own dogfood</a>' on display,
as the security lines <a href="https://www.bbc.co.uk/news/articles/ceqvjgrvpn3o">were long</a> and manual. The payment
system was actually really cool: <a href="https://en.wikipedia.org/wiki/Unified_Payments_Interface">UPI</a> allows
a vendor to display a QRCode to receive payment with 0% overhead. The user scans the QRCode on their
mobile, pays online, and then shows the proof to the vendor. The vendor doesn't need to have any electronic
equipment at all.  This had nothing to do with AI, but reminded me strongly of our work on <a href="https://anil.recoil.org/projects/ubiqinteraction">spotcodes</a> over two decades ago!</p>
<p><img src="https://anil.recoil.org/images/aisummit-gen-1.webp" alt="%rc" title="Most talks at the summit were beyond standing room only">
<img src="https://anil.recoil.org/images/ai-hackathon-2.webp" alt="%rc">
With 200,000 people pouring through the expo, it was basically impossible to have any coordinated meetings as even getting from one end to the other took ages. However, this meant that a lot of attendees vibe coded up alternative (and more usable) interfaces, and were showing off QRCodes which I scanned in order to access their versions. One enterprising chap had a UPI payment link to an MCP endpoint that I could point my <a href="https://openclaw.ai/">Claw</a> to help figure out where to go next; I didn't buy this, but I appreciated the hustle!</p>
<p><img src="https://anil.recoil.org/images/aisummit-gen-4.webp" alt="%c" title="The three pillars of the AI Impact Summit"></p>
<h2><a href="https://anil.recoil.org/news.xml#meeting-yann-lecun-and-discussing-satellite-commons" class="anchor" aria-hidden="true"></a>Meeting Yann LeCun and discussing satellite commons</h2>
<p><img src="https://anil.recoil.org/images/aisummit-aia-1.webp" alt="%rc" title="Yann LeCun turns out to be a selfie king">
One of the evening events was hosted by the good folk at the <a href="https://thealliance.ai/">AI Alliance</a>, a not-for-profit aiming to boost open models.
The guest of honour Yann LeCun, who turns out to be the nicest person! He gave a layman's summary to the audience about why LLMs are an evolutionary dead end, and then started talking about how self-supervised methods such as <a href="https://arxiv.org/abs/2103.03230">Barlow Twins</a> and <a href="https://arxiv.org/abs/2301.08243">JEPA</a> are the future. Barlow Twins are, of course, the exact training mechanism we've been using to train <a href="https://anil.recoil.org/papers/2025-tessera">TESSERA</a>, and so I bounded up to him to discuss it in more detail!</p>
<p>Yann had made an argument on stage that the only fair way to train language models that are open is for every country to contribute a large corpus of country-specific data (not necessarily open), and to combine these at the global level via one level of federated learning into a 'united language model'.</p>
<p>While this may work ok for LLMs, I thought it was <em>even more perfect</em> for satellite data! We already have a base set of public data, in the form of Landsat and Sentinel 1/2/3. Meanwhile, many countries have their own geostationary task satellites hovering over them, often with much higher resolutions and interesting instruments. I even heard rumours that India has commissioned a <a href="https://surveyofindia.gov.in/">Lidar survey of the entire country</a>, but this may be a future project as all I could find concretely is one of <a href="https://www.linkedin.com/posts/surveyofindia_surveyofindia-lidar-digitalelevationmodel-activity-7282654478921601024-O62z/">major river systems</a>.</p>
<p>So while open frontier LLMs may be a lost cause in the short term, it strikes me as a real opportunity that TESSERA may be the perfect way to trial Yann's idea of a global training cooperative.  The incentives are all there, and geospatial foundation modeling seems to be maturing rapidly.</p>
<p><img src="https://anil.recoil.org/images/aisummit-aia-2.webp" alt="%c" title="With the leadership team of the AI Alliance"></p>
<h2><a href="https://anil.recoil.org/news.xml#holding-a-hackathon-and-visiting-iit-delhi" class="anchor" aria-hidden="true"></a>Holding a hackathon and visiting IIT-Delhi</h2>
<p><img src="https://anil.recoil.org/images/ai-hackathon-1.webp" alt="%rc">
I skipped the 'government day' of the summit (with all the glitzy tech CEO talks) in order to satisfy demand for <a href="https://anil.recoil.org/projects/tessera">TESSERA</a> talks.  I <a href="https://anil.recoil.org/notes/first-tessera-hackathon">advertised a hackathon</a> with OpenUK on the day before the summit, and we had a full house of signups by the end of the day! I posted a <a href="https://anil.recoil.org/notes/first-tessera-hackathon">trip report</a> for this separately.</p>
<p>There was so much discussion at the hackathon that I trotted along to the IIT-Delhi campus the morning after to give a detailed talk on the bigger picture of TESSERA (similar to the talk <a href="https://anil.recoil.org/notes/2026w6">at ARIA</a> last week). The audience was highly engaged and I went well over time answering questions. A number of students were interested in followup opportunities to work in this area, and I pointed them to <a href="https://kcsrk.info">KC Sivaramakrishnan</a> and his <a href="https://fplaunchpad.org/">FP Launchpad</a> which is taking off in April. We've got <a href="https://www.tunbury.org/2026/02/15/ocaml-tessera/">TESSERA and OCaml</a> playing well together now, so there's a really fun opportunity to combine functional programming with planetary computing now!</p>
<h2><a href="https://anil.recoil.org/news.xml#partyinghhnetworking-at-the-british-high-commission" class="anchor" aria-hidden="true"></a>Partying^H^Hnetworking at the British High Commission</h2>
<p></p><div class="video-vertical"><iframe title="AI Impact Summit at the British High Commission party" src="https://crank.recoil.org/videos/embed/ab39a536-a89a-4756-b04d-9fd6d9cc9bd5" frameborder="0" allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups allow-forms" style="aspect-ratio: 9/16; width: 100%; height: 100%; max-width: 325px;"></iframe></div>
I got a kind invitation from the British High Commissioner <a href="https://www.gov.uk/government/news/change-of-british-high-commissioner-to-india-lindy-cameron">Lindy Cameron</a> to attend a party at her house, which turns out to have the largest private garden in New Delhi.  The good and the great of the Delhi political scene were there, along with a number of visitors to the summit. The main draw of the event was a conversation between Rishi Sunak and David Lammy, but first Kanishka Narayan (the minister for AI and Digital Safety) and Amanda Brock from OpenUK announced the launch of 'open source and AI' video.<p></p>
<p>Both speeches were charming, and it was good to see the emphasis on openness. Kanishka Narayan made a wry observation that Britain might not lead on raw engineering resource, but it does have 'the best technical taste', which I thought was quite an apt claim!</p>
<p>Most of the conversations I had here were all about landuse and datacenter growth. There seems to be massive investment within India for datacentre capacity, so questions of water usage and landuse are obvious barriers. I'm looking forward to working with the <a href="https://core-stack.org">CoRE Stack</a> team to help map out some of these challenges throughout India.</p>
<p>There was also a lot of interest in datacentres in space, so I took the opportunity to explain what we're doing with our startup <a href="https://parsimoni.co/">Parsimoni</a> lead by <a href="https://github.com/samoht">Thomas Gazagnaire</a>. The idea of having a multi-tenant 'Docker in space' was received well by everyone I mentioned it to, and Thomas has been finding <a href="https://gazagnaire.org/blog/2026-02-19-nasa-fprime.html">a lot of similarity to our earlier unikernel work</a> as he builds SpaceOS out in California.</p>
<h2><a href="https://anil.recoil.org/news.xml#summit-outcomes" class="anchor" aria-hidden="true"></a>Summit Outcomes</h2>
<p>The <a href="https://www.mea.gov.in/bilateral-documents.htm?dtl/40809">summit declaration</a> that came out today is remarkable in actually being signed by the US, UK and China <a href="https://www.bbc.co.uk/news/articles/c8edn0n58gwo">unlike last time</a>.
Some snippets of interest to me from the statement are:</p>
<blockquote>
<p>We take note of the voluntary and collaborative International Network of AI for Science Institutions as a platform to connect scientific communities and pool AI research capabilities across regions among participating institutions, in order to accelerate the impactful adoption of AI.
[...]</p>
<p>While encouraging international collaboration on meaningful skilling and reskilling AI initiatives, we take note of the voluntary guiding principles for reskilling in the age of AI and the playbook on AI workforce development, which would support participants in preparation for a future AI driven economy.
[...]</p>
<p>We take note of the Global AI Impact Commons as a voluntary initiative that provides a practical platform to encourage and enable the adoption, replication, and scale-up of successful AI use cases across regions.
<cite>-- <a href="https://www.mea.gov.in/bilateral-documents.htm?dtl/40809">AI Impact Summit Declaration</a>, Feb 2026</cite></p>
</blockquote>
<p>The word "safety" is notably missing; it's all about rapid progress and equitable access now. This year will not be about whether or not AI adoption should happen; it's now a race to defend ourselves against <a href="https://anil.recoil.org/papers/2025-ai-poison">AI poisoning</a> and whether we take the <a href="https://anil.recoil.org/notes/red-pill-conservation">red pill or the blue pill</a> and embrace an open future. It is for this reason I'm very grateful to <a href="https://openuk.uk">OpenUK</a> for all their work on helping make sure the UK takes the red open-source pill. It'll be a harder road, but a worthwhile one.</p>
<p>An excellent recap of the outcomes of the summit can be found in this hour long segment on Indian TV with none other than my colleague <a href="https://inverseprobability.com/">Neil Lawrence</a>!</p>
<p></p><div class="video-center"><iframe title="Neil Lawrence on Indian TV at the AI Impact Expo" width="100%" height="315px" src="https://watch.eeg.cl.cam.ac.uk/videos/embed/46a77cfa-0db8-4010-abf4-357c14801897" frameborder="0" allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups allow-forms"></iframe></div><p></p>
<p>I also contributed to the very comprehensive OpenUK AI Openness Summit Report which is <a href="https://openuk.uk/wp-content/uploads/2026/02/AI-Impact-Summit-2026-AI-Openness-Report.pdf">available here</a> and very comprehensive. I'm not entirely sure what happened to the ATI report I contributed to in November; I suspect it's been filed away Indiana Jones style in some vast document repository underground...</p>
<p>Thanks for the hospitality New Delhi! It was an exhilarating whirlwind to be at the summit. Well done <a href="https://indiaai.gov.in/people/abhishek-singh">Abhishek Singh</a> and other organisers.</p>
<p><small class="credit"><strong>(Updated 23rd Feb 2026 with a link to the OpenUK report. March 6th 2026 with a typo from Sam Reynolds.)</strong></small></p><h1>References</h1><ul><li>Madhavapeddy (2026). Discussing effective conservation with all the UK Chief Scientists. <a href="https://doi.org/10.59350/qjrmv-38130" target="_blank"><i>10.59350/qjrmv-38130</i></a></li>
<li>Feng et al (2025). TESSERA: Temporal Embeddings of Surface Spectra for Earth Representation and Analysis. arXiv. <a href="https://doi.org/10.48550/arXiv.2506.20380" target="_blank"><i>10.48550/arXiv.2506.20380</i></a></li>
<li>Madhavapeddy (2025). On the path to the UK/India AI Summit with OpenUK and the ATI. <a href="https://doi.org/10.59350/x6rea-1g262" target="_blank"><i>10.59350/x6rea-1g262</i></a></li>
<li>Reynolds et al (2025). Will AI speed up literature reviews or derail them entirely?. Nature Publishing Group. <a href="https://doi.org/10.1038/d41586-025-02069-w" target="_blank"><i>10.1038/d41586-025-02069-w</i></a></li>
<li>Madhavapeddy (2026). 1st TESSERA/CoRE hackathon at the Indian AI Summit. <a href="https://doi.org/10.59350/1na80-7ak85" target="_blank"><i>10.59350/1na80-7ak85</i></a></li>
<li>Zbontar et al (2021). Barlow Twins: Self-Supervised Learning via Redundancy Reduction. arXiv. <a href="https://doi.org/10.48550/arXiv.2103.03230" target="_blank"><i>10.48550/arXiv.2103.03230</i></a></li>
<li>Assran et al (2023). Self-Supervised Learning from Images with a Joint-Embedding Predictive Architecture. arXiv. <a href="https://doi.org/10.48550/arXiv.2301.08243" target="_blank"><i>10.48550/arXiv.2301.08243</i></a></li></ul>
