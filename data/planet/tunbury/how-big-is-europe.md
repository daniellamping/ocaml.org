---
title: How big is Europe?
description: Tessera produces global land-cover embeddings at 0.1-degree resolution,
  roughly 11 km square at the equator. For each year and each grid tile, there is
  a directory containing NumPy files of the embeddings.
url: https://www.tunbury.org/2026/03/21/how-big-europe/
date: 2026-03-21T18:20:00-00:00
preview_image: https://www.tunbury.org/images/coverage_2024_diff_europe.png
authors:
- Mark Elvers
source:
ignore:
---

<p><a href="https://geotessera.org">Tessera</a> produces global land-cover embeddings at 0.1-degree resolution, roughly 11 km square at the equator. For each year and each grid tile, there is a directory containing NumPy files of the embeddings.</p>

<p>Each tile is about 100MB; multiply that by every year since 2017, and you end up with a directory tree containing millions of entries across hundreds of terabytes. If you wanted to copy a subset, how much storage would you need? For example, how much storage does the European subset occupy? This obviously calls for an OCaml tool to calculate it.</p>

<h1>The directory tree</h1>

<p>The embeddings are held in on the file system in this layout:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>/data/tessera/v1/global_0.1_degree_representation/
  2017/
    grid_7.55_46.05/
      grid_7.55_46.05.npy
      grid_7.55_46.05_scales.npy
      SHA256
    grid_-1.25_50.05/
      ...
  2018/
    ...
</code></pre></div></div>

<p>Each year directory can contain as many as 1.6 million <code class="language-plaintext highlighter-rouge">grid_&lt;lon&gt;_&lt;lat&gt;</code> subdirectories. The longitude and latitude encoded in the directory name represent the centre of the 0.1-degree cell. This naming convention is the key that lets us filter geographically without opening a single file.</p>

<h1>Reading shapefiles from OCaml</h1>

<p>To determine which grid cells fall within “Europe”, I need country boundary polygons. <a href="https://www.naturalearthdata.com/">Natural Earth</a> provides free vector data at several resolutions. The 110m admin-0 countries dataset comes as a pair of files: a <code class="language-plaintext highlighter-rouge">.shp</code> containing polygon geometry and a <code class="language-plaintext highlighter-rouge">.dbf</code> containing attribute columns.</p>

<p>Two opam libraries handle the parsing. The <a href="https://github.com/cyril-allignol/ocaml-shapefile">cyril-allignol/ocaml-shapefile</a> library reads <code class="language-plaintext highlighter-rouge">.shp</code> files and returns a list of shapes, each being an array of rings (arrays of <code class="language-plaintext highlighter-rouge">{x; y}</code> points). The <a href="https://github.com/pveber/dbf">pveber/dbf</a> library reads the dBASE <code class="language-plaintext highlighter-rouge">.dbf</code> database and returns columns as an association list:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">load_shapefile</span> <span class="n">shp_path</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">_header</span><span class="o">,</span> <span class="n">shapes</span> <span class="o">=</span> <span class="nn">Shapefile</span><span class="p">.</span><span class="nn">Shp</span><span class="p">.</span><span class="n">read</span> <span class="n">shp_path</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">dbf_path</span> <span class="o">=</span> <span class="nn">Filename</span><span class="p">.</span><span class="n">chop_extension</span> <span class="n">shp_path</span> <span class="o">^</span> <span class="s2">".dbf"</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">dbf</span> <span class="o">=</span> <span class="k">match</span> <span class="nn">Dbf</span><span class="p">.</span><span class="n">of_file</span> <span class="n">dbf_path</span> <span class="k">with</span>
    <span class="o">|</span> <span class="nc">Ok</span> <span class="n">d</span> <span class="o">-&gt;</span> <span class="n">d</span>
    <span class="o">|</span> <span class="nc">Error</span> <span class="nt">`Unexpected_end_of_file</span> <span class="o">-&gt;</span> <span class="n">failwith</span> <span class="s2">"DBF: unexpected end of file"</span>
    <span class="o">|</span> <span class="nc">Error</span> <span class="nt">`Unknown_file_type</span> <span class="o">-&gt;</span> <span class="n">failwith</span> <span class="s2">"DBF: unknown file type"</span>
    <span class="o">|</span> <span class="nc">Error</span> <span class="nt">`Unknown_field_type</span> <span class="o">-&gt;</span> <span class="n">failwith</span> <span class="s2">"DBF: unknown field type"</span>
  <span class="k">in</span>
  <span class="k">let</span> <span class="n">names</span> <span class="o">=</span> <span class="n">get_string_column</span> <span class="n">dbf</span> <span class="s2">"NAME"</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">continents</span> <span class="o">=</span> <span class="n">get_string_column</span> <span class="n">dbf</span> <span class="s2">"CONTINENT"</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">subregions</span> <span class="o">=</span> <span class="n">get_string_column</span> <span class="n">dbf</span> <span class="s2">"SUBREGION"</span> <span class="k">in</span>
  <span class="o">...</span>
</code></pre></div></div>

<p>The DBF format stores strings padded with nulls and spaces, so a small <code class="language-plaintext highlighter-rouge">strip_nulls</code> function trims trailing zeros. Each record in the shapefile has a corresponding row in the DBF, so the geometry and metadata are joined together in a list of <code class="language-plaintext highlighter-rouge">{ name; continent; subregion; shape }</code> records.</p>

<p>The Natural Earth DBF includes <code class="language-plaintext highlighter-rouge">CONTINENT</code>, <code class="language-plaintext highlighter-rouge">REGION_UN</code> and <code class="language-plaintext highlighter-rouge">SUBREGION</code> columns, which means we can select countries by group rather than listing them individually. Our tool supports composable flags:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>--continent Europe --exclude-country Russia --include-country Turkey
</code></pre></div></div>

<p>Selection is applied in order: start with all countries matching <code class="language-plaintext highlighter-rouge">--continent</code> or <code class="language-plaintext highlighter-rouge">--subregion</code>, add any <code class="language-plaintext highlighter-rouge">--include-country</code> entries, then remove any <code class="language-plaintext highlighter-rouge">--exclude-country</code> entries. All matching is case-insensitive.</p>

<h1>Ray casting</h1>

<p>For each grid directory, parse the longitude and latitude from the name and test whether that point falls inside any of the selected country polygons. To test if a point is within the region polygon, the standard algorithm casts a horizontal ray from the test point to infinity and counts how many polygon edges it crosses. An odd count means the point is inside.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">point_in_ring</span> <span class="p">(</span><span class="n">px</span><span class="o">,</span> <span class="n">py</span><span class="p">)</span> <span class="p">(</span><span class="n">ring</span> <span class="o">:</span> <span class="nn">Shapefile</span><span class="p">.</span><span class="nn">D2</span><span class="p">.</span><span class="n">point</span> <span class="kt">array</span><span class="p">)</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">n</span> <span class="o">=</span> <span class="nn">Array</span><span class="p">.</span><span class="n">length</span> <span class="n">ring</span> <span class="k">in</span>
  <span class="k">if</span> <span class="n">n</span> <span class="o">&lt;</span> <span class="mi">3</span> <span class="k">then</span> <span class="bp">false</span>
  <span class="k">else</span>
    <span class="k">let</span> <span class="n">test_edge</span> <span class="n">inside</span> <span class="n">i</span> <span class="n">j</span> <span class="o">=</span>
      <span class="k">let</span> <span class="n">pi</span> <span class="o">=</span> <span class="n">ring</span><span class="o">.</span><span class="p">(</span><span class="n">i</span><span class="p">)</span> <span class="ow">and</span> <span class="n">pj</span> <span class="o">=</span> <span class="n">ring</span><span class="o">.</span><span class="p">(</span><span class="n">j</span><span class="p">)</span> <span class="k">in</span>
      <span class="k">if</span> <span class="p">(</span><span class="n">pi</span><span class="o">.</span><span class="n">y</span> <span class="o">&gt;</span> <span class="n">py</span><span class="p">)</span> <span class="o">&lt;&gt;</span> <span class="p">(</span><span class="n">pj</span><span class="o">.</span><span class="n">y</span> <span class="o">&gt;</span> <span class="n">py</span><span class="p">)</span>
         <span class="o">&amp;&amp;</span> <span class="n">px</span> <span class="o">&lt;</span> <span class="p">(</span><span class="n">pj</span><span class="o">.</span><span class="n">x</span> <span class="o">-.</span> <span class="n">pi</span><span class="o">.</span><span class="n">x</span><span class="p">)</span> <span class="o">*.</span> <span class="p">(</span><span class="n">py</span> <span class="o">-.</span> <span class="n">pi</span><span class="o">.</span><span class="n">y</span><span class="p">)</span> <span class="o">/.</span> <span class="p">(</span><span class="n">pj</span><span class="o">.</span><span class="n">y</span> <span class="o">-.</span> <span class="n">pi</span><span class="o">.</span><span class="n">y</span><span class="p">)</span> <span class="o">+.</span> <span class="n">pi</span><span class="o">.</span><span class="n">x</span>
      <span class="k">then</span> <span class="n">not</span> <span class="n">inside</span>
      <span class="k">else</span> <span class="n">inside</span>
    <span class="k">in</span>
    <span class="k">let</span> <span class="k">rec</span> <span class="n">loop</span> <span class="n">inside</span> <span class="n">i</span> <span class="o">=</span>
      <span class="k">if</span> <span class="n">i</span> <span class="o">&gt;=</span> <span class="n">n</span> <span class="k">then</span> <span class="n">inside</span>
      <span class="k">else</span> <span class="n">loop</span> <span class="p">(</span><span class="n">test_edge</span> <span class="n">inside</span> <span class="n">i</span> <span class="p">((</span><span class="n">i</span> <span class="o">+</span> <span class="n">n</span> <span class="o">-</span> <span class="mi">1</span><span class="p">)</span> <span class="ow">mod</span> <span class="n">n</span><span class="p">))</span> <span class="p">(</span><span class="n">i</span> <span class="o">+</span> <span class="mi">1</span><span class="p">)</span>
    <span class="k">in</span>
    <span class="n">loop</span> <span class="bp">false</span> <span class="mi">0</span>
</code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">test_edge</code> function checks two conditions for each edge of the polygon. First, do the two endpoints of the edge straddle the test point’s y-coordinate? The expression <code class="language-plaintext highlighter-rouge">(pi.y &gt; py) &lt;&gt; (pj.y &gt; py)</code> returns true when one endpoint is above and the other below. Second, is the test point to the left of where the ray would cross this edge? The x-coordinate of the intersection is computed by linear interpolation. If both conditions hold, we flip the <code class="language-plaintext highlighter-rouge">inside</code> state.</p>

<p>Shapefile polygons can have multiple rings. The first ring is the outer boundary; subsequent rings are holes. A country like the United Kingdom that consists of multiple landmasses has multiple outer rings. This is handled by parity counting. A point is “in” the polygon if it falls inside an odd number of rings:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">point_in_polygon</span> <span class="n">pt</span> <span class="n">rings</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">count</span> <span class="o">=</span>
    <span class="nn">Array</span><span class="p">.</span><span class="n">fold_left</span>
      <span class="p">(</span><span class="k">fun</span> <span class="n">acc</span> <span class="n">ring</span> <span class="o">-&gt;</span> <span class="k">if</span> <span class="n">point_in_ring</span> <span class="n">pt</span> <span class="n">ring</span> <span class="k">then</span> <span class="n">acc</span> <span class="o">+</span> <span class="mi">1</span> <span class="k">else</span> <span class="n">acc</span><span class="p">)</span>
      <span class="mi">0</span> <span class="n">rings</span>
  <span class="k">in</span>
  <span class="n">count</span> <span class="ow">mod</span> <span class="mi">2</span> <span class="o">=</span> <span class="mi">1</span>
</code></pre></div></div>

<h1>Scanning the filesystem</h1>

<p>Rather than shelling out to <code class="language-plaintext highlighter-rouge">du</code>, (which I did initially), the tool walks the directory tree directly using <code class="language-plaintext highlighter-rouge">Sys.readdir</code> and <code class="language-plaintext highlighter-rouge">Unix.lstat</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">dir_size_bytes</span> <span class="n">path</span> <span class="o">=</span>
  <span class="k">let</span> <span class="k">rec</span> <span class="n">walk</span> <span class="n">dir</span> <span class="n">acc</span> <span class="o">=</span>
    <span class="k">match</span> <span class="nn">Sys</span><span class="p">.</span><span class="n">readdir</span> <span class="n">dir</span> <span class="k">with</span>
    <span class="o">|</span> <span class="n">entries</span> <span class="o">-&gt;</span>
      <span class="nn">Array</span><span class="p">.</span><span class="n">fold_left</span> <span class="p">(</span><span class="k">fun</span> <span class="n">acc</span> <span class="n">name</span> <span class="o">-&gt;</span>
        <span class="k">let</span> <span class="n">full</span> <span class="o">=</span> <span class="nn">Filename</span><span class="p">.</span><span class="n">concat</span> <span class="n">dir</span> <span class="n">name</span> <span class="k">in</span>
        <span class="k">match</span> <span class="nn">Unix</span><span class="p">.</span><span class="n">lstat</span> <span class="n">full</span> <span class="k">with</span>
        <span class="o">|</span> <span class="p">{</span> <span class="nn">Unix</span><span class="p">.</span><span class="n">st_kind</span> <span class="o">=</span> <span class="nn">Unix</span><span class="p">.</span><span class="nc">S_REG</span><span class="p">;</span> <span class="n">st_size</span><span class="p">;</span> <span class="n">_</span> <span class="p">}</span> <span class="o">-&gt;</span> <span class="n">acc</span> <span class="o">+</span> <span class="n">st_size</span>
        <span class="o">|</span> <span class="p">{</span> <span class="nn">Unix</span><span class="p">.</span><span class="n">st_kind</span> <span class="o">=</span> <span class="nn">Unix</span><span class="p">.</span><span class="nc">S_DIR</span><span class="p">;</span> <span class="n">_</span> <span class="p">}</span> <span class="o">-&gt;</span> <span class="n">walk</span> <span class="n">full</span> <span class="n">acc</span>
        <span class="o">|</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="n">acc</span>
        <span class="o">|</span> <span class="k">exception</span> <span class="nn">Unix</span><span class="p">.</span><span class="nc">Unix_error</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="n">acc</span>
      <span class="p">)</span> <span class="n">acc</span> <span class="n">entries</span>
    <span class="o">|</span> <span class="k">exception</span> <span class="nc">Sys_error</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="n">acc</span>
  <span class="k">in</span>
  <span class="n">walk</span> <span class="n">path</span> <span class="mi">0</span>
</code></pre></div></div>

<p>The tool filters before measuring. The per-year scan parses the grid coordinates from each directory name and tests them against the country polygons. Only matching directories get measured with a call to <code class="language-plaintext highlighter-rouge">dir_size_bytes</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">scan_year_filtered</span> <span class="n">year_path</span> <span class="n">year</span> <span class="n">polys</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">grid_dirs</span> <span class="o">=</span> <span class="n">list_dirs</span> <span class="n">year_path</span> <span class="o">|&gt;</span> <span class="nn">List</span><span class="p">.</span><span class="n">filter</span> <span class="n">is_grid_dir</span> <span class="k">in</span>
  <span class="nn">List</span><span class="p">.</span><span class="n">fold_left</span> <span class="p">(</span><span class="k">fun</span> <span class="n">stats</span> <span class="n">dir_name</span> <span class="o">-&gt;</span>
    <span class="k">match</span> <span class="n">parse_grid_coords</span> <span class="n">dir_name</span> <span class="k">with</span>
    <span class="o">|</span> <span class="nc">Some</span> <span class="p">(</span><span class="n">lon</span><span class="o">,</span> <span class="n">lat</span><span class="p">)</span> <span class="k">when</span> <span class="n">point_in_any_country</span> <span class="p">(</span><span class="n">lon</span><span class="o">,</span> <span class="n">lat</span><span class="p">)</span> <span class="n">polys</span> <span class="o">-&gt;</span>
      <span class="k">let</span> <span class="n">bytes</span> <span class="o">=</span> <span class="n">dir_size_bytes</span> <span class="p">(</span><span class="nn">Filename</span><span class="p">.</span><span class="n">concat</span> <span class="n">year_path</span> <span class="n">dir_name</span><span class="p">)</span> <span class="k">in</span>
      <span class="p">{</span> <span class="n">stats</span> <span class="k">with</span> <span class="n">matched_bytes</span> <span class="o">=</span> <span class="n">stats</span><span class="o">.</span><span class="n">matched_bytes</span> <span class="o">+</span> <span class="n">bytes</span><span class="p">;</span> <span class="o">...</span> <span class="p">}</span>
    <span class="o">|</span> <span class="n">_</span> <span class="o">-&gt;</span>
      <span class="n">stats</span>  <span class="c">(* skip — no filesystem traversal *)</span>
 <span class="p">)</span> <span class="n">empty_stats</span> <span class="n">grid_dirs</span>

</code></pre></div></div>

<p>As mentioned above, there could be as many as 1.6 million directories per year, but only ~80,000 match Europe, thus avoiding overworking the filesystem.</p>

<h1>Parallel scanning with OCaml 5 domains</h1>

<p>The scan is embarrassingly parallel. Each year’s directory tree is completely independent. OCaml 5’s multicore support makes this trivial. Each year gets its own <code class="language-plaintext highlighter-rouge">Domain</code>, and results are merged after all domains join:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">scan_parallel</span> <span class="n">root</span> <span class="n">polys</span> <span class="n">scan_year</span> <span class="o">=</span>
  <span class="k">let</span> <span class="n">years</span> <span class="o">=</span> <span class="n">list_dirs</span> <span class="n">root</span> <span class="o">|&gt;</span> <span class="nn">List</span><span class="p">.</span><span class="n">filter</span> <span class="n">is_year_string</span> <span class="k">in</span>
  <span class="k">let</span> <span class="n">domains</span> <span class="o">=</span>
    <span class="nn">List</span><span class="p">.</span><span class="n">map</span> <span class="p">(</span><span class="k">fun</span> <span class="n">year</span> <span class="o">-&gt;</span>
      <span class="k">let</span> <span class="n">year_path</span> <span class="o">=</span> <span class="nn">Filename</span><span class="p">.</span><span class="n">concat</span> <span class="n">root</span> <span class="n">year</span> <span class="k">in</span>
      <span class="nn">Domain</span><span class="p">.</span><span class="n">spawn</span> <span class="p">(</span><span class="k">fun</span> <span class="bp">()</span> <span class="o">-&gt;</span> <span class="n">scan_year</span> <span class="n">year_path</span> <span class="n">year</span> <span class="n">polys</span><span class="p">)</span>
    <span class="p">)</span> <span class="n">years</span>
  <span class="k">in</span>
  <span class="nn">List</span><span class="p">.</span><span class="n">fold_left</span>
    <span class="p">(</span><span class="k">fun</span> <span class="n">acc</span> <span class="n">domain</span> <span class="o">-&gt;</span> <span class="n">merge_stats</span> <span class="n">acc</span> <span class="p">(</span><span class="nn">Domain</span><span class="p">.</span><span class="n">join</span> <span class="n">domain</span><span class="p">))</span>
    <span class="n">empty_stats</span> <span class="n">domains</span>
</code></pre></div></div>

<h1>The results!</h1>

<p>Running the tool against the full global dataset with <code class="language-plaintext highlighter-rouge">--continent Europe --exclude-country Russia --include-country Turkey</code>:</p>

<table>
  <thead>
    <tr>
      <th>Year</th>
      <th>Grid tiles</th>
      <th>Size (TB)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>2017</td>
      <td>79,617</td>
      <td>7.4</td>
    </tr>
    <tr>
      <td>2018</td>
      <td>79,744</td>
      <td>7.4</td>
    </tr>
    <tr>
      <td>2019</td>
      <td>79,700</td>
      <td>7.4</td>
    </tr>
    <tr>
      <td>2020</td>
      <td>79,698</td>
      <td>7.4</td>
    </tr>
    <tr>
      <td>2021</td>
      <td>79,699</td>
      <td>7.4</td>
    </tr>
    <tr>
      <td>2022</td>
      <td>79,750</td>
      <td>7.4</td>
    </tr>
    <tr>
      <td>2023</td>
      <td>79,638</td>
      <td>7.4</td>
    </tr>
    <tr>
      <td>2024</td>
      <td>89,938</td>
      <td>8.5</td>
    </tr>
    <tr>
      <td>2025</td>
      <td>79,678</td>
      <td>7.4</td>
    </tr>
    <tr>
      <td><strong>Total</strong></td>
      <td><strong>727,462</strong></td>
      <td><strong>~68 TB</strong></td>
    </tr>
  </tbody>
</table>

<p>The 2024 has 13% more tiles than the other years, as Turkey is included, along with some extra coastal tiles. From the header image, white pixels is available in all years, while red pixels are only available in 2024.</p>

<h1>Code</h1>

<p>The code is available in <a href="https://github.com/embedding-size">mtelvers/embedding-size</a>.</p>
