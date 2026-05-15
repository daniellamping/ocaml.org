---
title: Building a UK Fuel Price Finder in OCaml
description: The UK government recently launched the Fuel Finder API, providing real-time
  pricing data for over 7,000 petrol stations across the country.
url: https://www.tunbury.org/2026/03/13/fuel-prices/
date: 2026-03-13T14:00:00-00:00
preview_image: https://www.tunbury.org/images/fuel.png
authors:
- Mark Elvers
source:
ignore:
---

<p>The UK government recently launched the <a href="https://www.developer.fuel-finder.service.gov.uk/apicontent">Fuel Finder API</a>, providing real-time pricing data for over 7,000 petrol stations across the country.</p>

<p>I thought it would be fun to build a small web app to visualise the data allow me to see the cheapest fuel near me. However, I haven’t got time, so I asked Claude to do it… in OCaml, of course.</p>

<h1>What it does</h1>

<p>Enter a postcode or tap “Use my location”, pick a fuel type, and the app shows every station within your chosen radius on a Leaflet map. Markers are colour-coded green to red by price. A ranking bar along the bottom lets you tap a price point to pulse the matching stations on the map, which is useful when several stations share the same price, and you want to see which ones they are.</p>

<p>Stale prices, which have not been updated in over a week, get an orange border so you can see at a glance which bargains might be due to stale data.</p>

<h1>The stack</h1>

<p>The entire backend is a single OCaml file using Eio, cohttp-eio, tls-eio and Yojson. On startup, it authenticates via OAuth2 client credentials, paginates through all station data and prices (about 15 batches of 500 each), merges them, and caches the result for 15 minutes. Token refresh is handled automatically under a mutex.</p>

<h1>Price data quirks</h1>

<p>Some stations report prices in pounds (e.g. 1.39), while most use pence (e.g. 139.9). The app normalises anything under 10 by multiplying by 100 and discards anything outside 50–500p, although we might need to revise that upper bound soon.</p>

<p>BBC reporting this morning said the regulator was bringing forward the requirements for data updates. The <code class="language-plaintext highlighter-rouge">price_last_updated</code> field colours outside of the station marker so you can see that the absurdly cheap stations just haven’t reported in weeks.</p>

<p>The code is available at <a href="https://github.com/mtelvers/ocaml-fuel">mtelvers/ocaml-fuel</a> with a live site running at <a href="https://fuel.mint.caelum.ci.dev">fuel.mint.caelum.ci.dev</a>.</p>
