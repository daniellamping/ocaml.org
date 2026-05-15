---
title: Weeknotes 2026 week 12
description:
url: https://jon.recoil.org/blog/2026/03/weeknotes-2026-12.html
date: 2026-03-23T00:00:00-00:00
preview_image:
authors:
- Jon Ludlam
source:
ignore:
---

<h1><a href="https://jon.recoil.org/atom.xml#weeknotes-2026-week-12" class="anchor"></a>Weeknotes 2026 week 12</h1>
<ul class="at-tags"><li class="published"><span class="at-tag">published</span> <p>2026-03-23</p></li></ul>
<ul class="at-tags"><li class="page-tags"><span class="at-tag">page-tags</span> <p><a href="https://jon.recoil.org/tags/weeknotes.html" title="weeknotes">weeknotes</a> <a href="https://jon.recoil.org/tags/tessera.html" title="tessera">tessera</a></p></li></ul>
<h2><a href="https://jon.recoil.org/atom.xml#what-did-i-do?" class="anchor"></a>What did I do?</h2>
<p>End of term this week, so my tutorial interviews kept me quite busy for some of the week. Then Paul-Elliot has returned from his sojourn working on Slipshow, so I had a useful few hours working with him to try and begin the process of handing over the dune odoc PR.</p>
<p>I mentioned <a href="https://jon.recoil.org/weeknotes-2026-11.html" title="weeknotes-2026-11">last week</a> that the TESSERA reprojection was causing issues with the overlay alignment. Now the original loading of the patches was taking ages, so I switched to zarr to be more efficient. Also, the PCA was slow, so I switched that over to tensorflow.js to use the GPU.</p>
<p>With these two optimisations in place, it was now a lot quicker to see if the misalignment was still there. It seemed likely that the issue was translating between the UTM grid that TESSERA uses and the WGS84 coordinate system that Leaflet.js is using. It was quite quick to whip up a <a href="https://jon.recoil.org/reference/tessera-geotessera/tessera-geotessera/Geotessera/Utm/index.html#val-wgs84_to_utm" title="Geotessera.Utm.wgs84_to_utm">conversion routine</a> (click on 'source' to see the gory details!). With that in place, the overlay now matches up precisely as you can see on the map below. Be patient while it runs, you'll see the overlay on the map in a few seconds!</p>
<div><pre class="language-ocaml"><code>#require "tessera-zarr-jsoo";;
#require "tessera-viz-jsoo";;
#require "tessera-tfjs";;
#require "js_top_worker-widget-leaflet";;
open Widget_leaflet;;
register ();;
(* Load fzstd (Zstd decompressor) and TensorFlow.js *)
let () =
  let open Js_of_ocaml in
  let import url : unit = Js.Unsafe.fun_call
    (Js.Unsafe.get Js.Unsafe.global (Js.string "importScripts"))
    [| Js.Unsafe.inject (Js.string url) |] in
  import "https://cdn.jsdelivr.net/npm/fzstd@0.1.1/umd/index.js";
  import "https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4/dist/tf.min.js"</code></pre></div>
<h3><a href="https://jon.recoil.org/atom.xml#reprojection-test" class="anchor"></a>Reprojection test</h3>
<p>Create the map, fetch embeddings from the Zarr store, run PCA via TensorFlow.js, and overlay — all in one cell so the async pipeline completes before the overlay is drawn:</p>
<div><pre class="language-ocaml"><code>let status_view text =
  let open Widget.View in
  Element { tag = "div"; attrs = [Style ("padding", "8px"); Style ("font-family", "monospace")];
    children = [Text text] }

let () = Widget.display ~id:"status" ~handlers:[] (status_view "Initialising...")

let map = Leaflet_map.create
  ~center:(52.30690, -0.03296) ~zoom:14 ~height:"500px"
  ()

let bbox = Geotessera.{
  min_lat = 52.29924; min_lon = -0.05845;
  max_lat = 52.31745; max_lon = -0.00755;
}

let downsample mat ~h ~w ~max_pixels =
  let n = h * w in
  if n &lt;= max_pixels then (mat, h, w)
  else
    let stride = int_of_float (ceil (sqrt (float_of_int n /. float_of_int max_pixels))) in
    let h' = (h + stride - 1) / stride in
    let w' = (w + stride - 1) / stride in
    let out = Linalg.create_mat ~rows:(h' * w') ~cols:mat.Linalg.cols in
    for i = 0 to h' - 1 do
      for j = 0 to w' - 1 do
        let si = min (i * stride) (h - 1) in
        let sj = min (j * stride) (w - 1) in
        for f = 0 to mat.Linalg.cols - 1 do
          Linalg.mat_set out (i * w' + j) f
            (Linalg.mat_get mat (si * w + sj) f)
        done
      done
    done;
    (out, h', w')

let () =
  Widget.update ~id:"status" (status_view "Opening Zarr store...");
  Lwt.async (fun () -&gt;
    let open Lwt.Syntax in
    let* store = Tessera_zarr_jsoo.open_store () in
    let progress msg = Widget.update ~id:"status" (status_view msg) in
    let* (mat_full, h_full, w_full, geo_bounds) =
      Tessera_zarr.fetch_region ~progress ~store bbox in
    Widget.update ~id:"status"
      (status_view (Printf.sprintf "Fetched %d×%d. Downsampling..." h_full w_full));
    let (mat, h, w) = downsample mat_full ~h:h_full ~w:w_full ~max_pixels:500_000 in
    let bounds = Leaflet_map.{
      south = geo_bounds.Geotessera.min_lat;
      north = geo_bounds.Geotessera.max_lat;
      west = geo_bounds.Geotessera.min_lon;
      east = geo_bounds.Geotessera.max_lon;
    } in
    Widget.update ~id:"status"
      (status_view (Printf.sprintf "Computing PCA on %d×%d mosaic..." h w));
    let proj = Tfjs.pca mat ~n_components:3 in
    let img = Viz.pca_to_rgba ~width:w ~height:h proj in
    let url = Viz_jsoo.to_data_url img in
    Leaflet_map.add_image_overlay map ~url ~bounds ~opacity:0.7 ();
    Widget.update ~id:"status"
      (status_view (Printf.sprintf
        "Done. Input bbox: S%.5f W%.5f N%.5f E%.5f | Overlay bounds: S%.5f W%.5f N%.5f E%.5f"
        bbox.min_lat bbox.min_lon bbox.max_lat bbox.max_lon
        bounds.south bounds.west bounds.north bounds.east));
    Lwt.return_unit)</code></pre></div>
