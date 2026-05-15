---
title: Reading the Gas Meter
description: "My gas supplier has tried and failed to install a smart gas meter, so
  I\u2019ll give it a go myself."
url: https://www.tunbury.org/2025/11/23/gas-meter/
date: 2025-11-23T18:30:00-00:00
preview_image: https://www.tunbury.org/images/gas-meter.png
authors:
- Mark Elvers
source:
ignore:
---

<p>My gas supplier has tried and failed to install a smart gas meter, so I’ll give it a go myself.</p>

<p>Numerous videos on YouTube demonstrate a pipeline for capturing and processing images with AI, but this is a heavyweight solution for basic image recognition. With a fixed camera, I can compare the reference images of each digit with the current values.</p>

<p>I have placed a Raspberry Pi with a camera module pointing at the gas meter.</p>

<p><img src="https://www.tunbury.org/images/gas-meter-camera.png" alt=""></p>

<p>In an ideal world, my image would be a grid of numbers with 0 = black and 255 = white.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[[  0,   0, 255, 255, 255,   0,   0  ];
 [  0, 255,   0,   0,   0, 255,   0  ];
 [  0,   0,   0,   0,   0, 255,   0  ];
 [  0,   0, 255, 255, 255,   0,   0  ];
 [  0,   0,   0,   0,   0, 255,   0  ];
 [  0, 255,   0,   0,   0, 255,   0  ];
 [  0,   0, 255, 255, 255,   0,   0  ]]
</code></pre></div></div>

<p>This would flatten into a 1D vector.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[ 0; 0; 255; 255; 255; 0; 0; 0; 255; 0; 0; 0; 255; 0; ...]
</code></pre></div></div>

<p>Then I could use the Euclidean distance to see how far apart the current image is from each of the reference images:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">euclidean_distance</span> <span class="n">v1</span> <span class="n">v2</span> <span class="o">=</span>
  <span class="nn">Array</span><span class="p">.</span><span class="n">mapi</span> <span class="p">(</span><span class="k">fun</span> <span class="n">i</span> <span class="n">x</span> <span class="o">-&gt;</span> <span class="p">(</span><span class="n">x</span> <span class="o">-.</span> <span class="n">v2</span><span class="o">.</span><span class="p">(</span><span class="n">i</span><span class="p">))</span> <span class="o">**</span> <span class="mi">2</span><span class="o">.</span><span class="p">)</span> <span class="n">v1</span>
  <span class="o">|&gt;</span> <span class="nn">Array</span><span class="p">.</span><span class="n">fold_left</span> <span class="p">(</span> <span class="o">+.</span> <span class="p">)</span> <span class="mi">0</span><span class="o">.</span><span class="mi">0</span>
  <span class="o">|&gt;</span> <span class="n">sqrt</span>
</code></pre></div></div>

<p>However, as the brightness of images may vary due to reflections from the plastic housing, using the angle between the two vectors would likely be more effective. Ranging from -1 to 1, where 1 = identical.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">dot_product</span> <span class="n">v1</span> <span class="n">v2</span> <span class="o">=</span>
  <span class="nn">Array</span><span class="p">.</span><span class="n">map2</span> <span class="p">(</span> <span class="o">*.</span> <span class="p">)</span> <span class="n">v1</span> <span class="n">v2</span> <span class="o">|&gt;</span> <span class="nn">Array</span><span class="p">.</span><span class="n">fold_left</span> <span class="p">(</span> <span class="o">+.</span> <span class="p">)</span> <span class="mi">0</span><span class="o">.</span><span class="mi">0</span>

<span class="k">let</span> <span class="n">magnitude</span> <span class="n">v</span> <span class="o">=</span>
  <span class="nn">Array</span><span class="p">.</span><span class="n">fold_left</span> <span class="p">(</span><span class="k">fun</span> <span class="n">acc</span> <span class="n">x</span> <span class="o">-&gt;</span> <span class="n">acc</span> <span class="o">+.</span> <span class="n">x</span> <span class="o">*.</span> <span class="n">x</span><span class="p">)</span> <span class="mi">0</span><span class="o">.</span><span class="mi">0</span> <span class="n">v</span> <span class="o">|&gt;</span> <span class="n">sqrt</span>

<span class="k">let</span> <span class="n">cosine_similarity</span> <span class="n">v1</span> <span class="n">v2</span> <span class="o">=</span>
  <span class="n">dot_product</span> <span class="n">v1</span> <span class="n">v2</span> <span class="o">/.</span> <span class="p">(</span><span class="n">magnitude</span> <span class="n">v1</span> <span class="o">*.</span> <span class="n">magnitude</span> <span class="n">v2</span><span class="p">)</span>
</code></pre></div></div>

<p>My gas meter is the kind where the digits rotate on mechanical wheels, which makes their vertical position vary over time. If I capture the basic area where the digit is, it could be near the top, near the bottom, or anywhere in between, resulting in a wide range of outcomes.</p>

<p>Therefore, I must first find the bounding box of the number. As the numbers are white on a black background, the simplest approach is to find the maximum and minimum brightness levels and set a threshold accordingly. I tested levels from 10% to 90% in steps of 10 and opted for 85%.</p>

<p><img src="https://www.tunbury.org/images/gas-threshold-10.png" alt=""> <img src="https://www.tunbury.org/images/gas-threshold-20.png" alt=""> <img src="https://www.tunbury.org/images/gas-threshold-30.png" alt=""> <img src="https://www.tunbury.org/images/gas-threshold-40.png" alt=""> <img src="https://www.tunbury.org/images/gas-threshold-50.png" alt=""> <img src="https://www.tunbury.org/images/gas-threshold-60.png" alt=""> <img src="https://www.tunbury.org/images/gas-threshold-70.png" alt=""> <img src="https://www.tunbury.org/images/gas-threshold-80.png" alt=""> <img src="https://www.tunbury.org/images/gas-threshold-90.png" alt=""></p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">threshold</span> <span class="o">=</span> <span class="n">min_v</span> <span class="o">+</span> <span class="p">(</span><span class="n">max_v</span> <span class="o">-</span> <span class="n">min_v</span><span class="p">)</span> <span class="o">*</span> <span class="mi">85</span> <span class="o">/</span> <span class="mi">100</span>
</code></pre></div></div>

<p>The bounding box can be found by searching for the first row with a bright pixel and the first column with a bright pixel:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">first_row</span> <span class="o">=</span>
  <span class="nn">Array</span><span class="p">.</span><span class="n">find_index</span> <span class="p">(</span><span class="k">fun</span> <span class="n">row</span> <span class="o">-&gt;</span> <span class="nn">Array</span><span class="p">.</span><span class="n">exists</span> <span class="p">(</span><span class="k">fun</span> <span class="n">v</span> <span class="o">-&gt;</span> <span class="n">v</span> <span class="o">&gt;</span> <span class="n">threshold</span><span class="p">)</span> <span class="n">row</span><span class="p">)</span> <span class="n">arr</span>
  <span class="o">|&gt;</span> <span class="nn">Option</span><span class="p">.</span><span class="n">value</span> <span class="o">~</span><span class="n">default</span><span class="o">:</span><span class="mi">0</span>

<span class="k">let</span> <span class="n">first_col</span> <span class="o">=</span>
  <span class="nn">Array</span><span class="p">.</span><span class="n">find_mapi</span> <span class="p">(</span><span class="k">fun</span> <span class="n">x</span> <span class="n">_</span> <span class="o">-&gt;</span>
    <span class="nn">Array</span><span class="p">.</span><span class="n">find_opt</span> <span class="p">(</span><span class="k">fun</span> <span class="n">row</span> <span class="o">-&gt;</span> <span class="n">row</span><span class="o">.</span><span class="p">(</span><span class="n">x</span><span class="p">)</span> <span class="o">&gt;</span> <span class="n">threshold</span><span class="p">)</span> <span class="n">arr</span>
    <span class="o">|&gt;</span> <span class="nn">Option</span><span class="p">.</span><span class="n">map</span> <span class="p">(</span><span class="k">fun</span> <span class="n">_</span> <span class="o">-&gt;</span> <span class="n">x</span><span class="p">)</span>
  <span class="p">)</span> <span class="n">arr</span><span class="o">.</span><span class="p">(</span><span class="mi">0</span><span class="p">)</span> <span class="o">|&gt;</span> <span class="nn">Option</span><span class="p">.</span><span class="n">value</span> <span class="o">~</span><span class="n">default</span><span class="o">:</span><span class="mi">0</span>
</code></pre></div></div>

<p>The captured image is first cropped to the area where the digit is known to appear and converted to grayscale. The 85% threshold is applied to create a two-colour image, which makes it easy to find the bounding box. The grey-scale image is then extracted for processing.</p>

<p><img src="https://www.tunbury.org/images/gas-1-grayscale.png" alt=""> <img src="https://www.tunbury.org/images/gas-2-binary.png" alt=""> <img src="https://www.tunbury.org/images/gas-3-bbox.png" alt=""> <img src="https://www.tunbury.org/images/gas-4-extracted.png" alt=""></p>

<p>With the image extracted, calculate the cosine similarity with all the template images and sort them.</p>

<table>
  <thead>
    <tr>
      <th>Template</th>
      <th>Score</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>6</td>
      <td>0.9260</td>
    </tr>
    <tr>
      <td>4</td>
      <td>0.8447</td>
    </tr>
    <tr>
      <td>8</td>
      <td>0.8358</td>
    </tr>
    <tr>
      <td>0</td>
      <td>0.8123</td>
    </tr>
    <tr>
      <td>5</td>
      <td>0.7764</td>
    </tr>
    <tr>
      <td>9</td>
      <td>0.7449</td>
    </tr>
    <tr>
      <td>3</td>
      <td>0.7640</td>
    </tr>
    <tr>
      <td>1</td>
      <td>0.6674</td>
    </tr>
    <tr>
      <td>2</td>
      <td>0.6623</td>
    </tr>
    <tr>
      <td>7</td>
      <td>0.6062</td>
    </tr>
  </tbody>
</table>

<p>The interpretation success is perfect except for the final digit, which rotates very quickly, and the captured image is often cropped or shows multiple digits.</p>

<p>The code for this project is available at <a href="https://github.com/mtelvers/gas-meter">mtelvers/gas-meter</a>.</p>
