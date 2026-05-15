---
title: Can a CPU with Intel AMX Match a GPU for ML Inference?
description: "GPU acceleration is the default assumption for machine learning inference.
  But Intel\u2019s AMX (Advanced Matrix Extensions) may close the gap. AMX is built
  into recent Xeon processors, which are available from Azure. Can they compete with
  similarly priced GPU-based machines for the Tessera pipeline?"
url: https://www.tunbury.org/2026/04/08/intel-amx/
date: 2026-04-08T21:00:00-00:00
preview_image: https://www.tunbury.org/images/intel-xeon-2024.jpg
authors:
- Mark Elvers
source:
ignore:
---

<p>GPU acceleration is the default assumption for machine learning inference. But Intel’s AMX (Advanced Matrix Extensions) may close the gap. AMX is built into recent Xeon processors, which are available from Azure. Can they compete with similarly priced GPU-based machines for the Tessera pipeline?</p>

<p>The Tessera encoder produces 128-dimensional embeddings from multi-temporal Sentinel-2 and Sentinel-1 satellite imagery.</p>

<ul>
  <li>Inputs: 40 time-sampled optical observations <code class="language-plaintext highlighter-rouge">[batch, 40, 11]</code> and 40 SAR observations <code class="language-plaintext highlighter-rouge">[batch, 40, 3]</code></li>
  <li>Output: one embedding per pixel <code class="language-plaintext highlighter-rouge">[batch, 128]</code></li>
</ul>

<p>The model processes the Earth’s land surface as a grid of 0.1° tiles. At 10m resolution, each tile is roughly 1,000 x 1,000 pixels. The exact dimensions vary with latitude, but 1 million is a good representative figure per tile.</p>

<h1>Hardware and Cost</h1>

<p>In this post, I am going to compare our monster <a href="https://www.tunbury.org/2026/03/11/gpu-vs-cpu/">AMD EPYC with NVIDIA L4</a> with two commonly available machines on Azure, which have nearly identical hourly rates:</p>

<table>
  <thead>
    <tr>
      <th>Machine</th>
      <th>Hardware</th>
      <th>Cores</th>
      <th>Accelerator</th>
      <th>$/hr</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Azure D16s_v6</td>
      <td>Intel Xeon 8573C</td>
      <td>8 physical</td>
      <td>AMX (bf16)</td>
      <td>$0.98</td>
    </tr>
    <tr>
      <td>Azure NC8as_T4_v3</td>
      <td>AMD EPYC 7V12</td>
      <td>4 physical</td>
      <td>Tesla T4</td>
      <td>$0.94</td>
    </tr>
    <tr>
      <td>Monteverde</td>
      <td>AMD EPYC 9965 (x2)</td>
      <td>384 physical</td>
      <td>AVX-512</td>
      <td>—</td>
    </tr>
    <tr>
      <td>Monteverde</td>
      <td>—</td>
      <td>—</td>
      <td><strong>NVIDIA L4</strong></td>
      <td>—</td>
    </tr>
  </tbody>
</table>

<p>The T4 is the most common cloud GPU. The AMX VM is a standard compute instance with no GPU drivers, no CUDA libraries, nor any special VM image. The L4 represents the current generation of inference GPUs.</p>

<h1>Convert to bfloat16</h1>

<p>AMX accelerates bfloat16 matrix operations but does nothing for float32. The model was trained in float32, but to use AMX, we must convert to bfloat16. However, to do a like-for-like comparison, we must acknowledge that a GPU has Tensor cores designed for bfloat16 calculations, and if we accept a reduction in resolution on the CPU, we should also do so on the GPU. The code change is simple; we only need to add one line:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">with</span> <span class="n">torch</span><span class="p">.</span><span class="n">no_grad</span><span class="p">(),</span> <span class="n">torch</span><span class="p">.</span><span class="n">autocast</span><span class="p">(</span><span class="n">device_type</span><span class="o">=</span><span class="s">"cpu"</span><span class="p">,</span> <span class="n">dtype</span><span class="o">=</span><span class="n">torch</span><span class="p">.</span><span class="n">bfloat16</span><span class="p">):</span>
    <span class="n">output</span> <span class="o">=</span> <span class="n">model</span><span class="p">(</span><span class="n">s2_input</span><span class="p">,</span> <span class="n">s1_input</span><span class="p">)</span>
</code></pre></div></div>

<p>PyTorch’s <code class="language-plaintext highlighter-rouge">autocast</code> automatically converts supported operations (linear layers, matmuls, attention) to bfloat16 while keeping numerically sensitive operations (layer norms, softmax) in float32. On CPU, this routes matrix multiplications through the AMX tile units. On the GPU, it engages the Tensor Cores: same API, same line of code, different hardware backend.</p>

<h1>Results</h1>

<p>For a realistic comparison, I ran the Tessera pipeline on a downloaded dpixel data for my favourite area of Manchester, which has 772,875 pixels.</p>

<table>
  <thead>
    <tr>
      <th>Configuration</th>
      <th>Inference (mm:ss)</th>
      <th>Cost/hr</th>
      <th>Cost/tile</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>L4 GPU, bfloat16</strong></td>
      <td><strong>2:46</strong></td>
      <td>—</td>
      <td>—</td>
    </tr>
    <tr>
      <td>L4 GPU, float32</td>
      <td>7:21</td>
      <td>—</td>
      <td>—</td>
    </tr>
    <tr>
      <td>T4 GPU, float32</td>
      <td>14:31</td>
      <td>$0.94</td>
      <td>$0.23</td>
    </tr>
    <tr>
      <td><strong>AMX CPU, bfloat16</strong></td>
      <td><strong>17:31</strong></td>
      <td><strong>$0.98</strong></td>
      <td><strong>$0.29</strong></td>
    </tr>
    <tr>
      <td>T4 GPU, bfloat16</td>
      <td>19:53</td>
      <td>$0.94</td>
      <td>$0.31</td>
    </tr>
    <tr>
      <td>AMX CPU, float32</td>
      <td>40:36</td>
      <td>$0.98</td>
      <td>$0.66</td>
    </tr>
  </tbody>
</table>

<p>Three results stand out:</p>

<p>At the same price point (~$1/hr), the AMX CPU processes a tile in 17.5 minutes vs the T4’s 14.5 minutes. It’s 20% slower, but that is still impressive: $0.29/tile vs $0.23/tile. The CPU is competitive, but slightly more expensive and slightly slower.</p>

<p>The T4 is slower with bfloat16 than with float32, which is surprising, as it has Tensor cores.</p>

<p>The L4 runs the inference in 2:46, which is five times faster than the T4 and six times faster than AMX. Modern Tensor Cores (Ada Lovelace generation) are clearly very efficient with bfloat16!</p>

<h1>Does bfloat16 affect output quality?</h1>

<p>Given the time and effort required to compute embeddings, I was concerned that converting from float32 to bfloat16 would degrade the resulting embeddings. Going fast is great, but not at the expense of data quality. As I now had the same tile processed six times, I could compare the results.</p>

<p>The pipeline outputs int8 quantised embeddings: each pixel gets a 128-dimensional vector of integers (−128 to +127) plus a per-pixel scale factor that reconstructs the original magnitude.</p>

<p>Firstly, all the float32 runs produce bit-identical output regardless of hardware: L4, T4, and AMX CPUs.</p>

<p>bfloat16 introduces small rounding differences:</p>

<table>
  <thead>
    <tr>
      <th>Comparison</th>
      <th>Exact match</th>
      <th>Off by 1</th>
      <th>Off by 2+</th>
      <th>Cosine similarity</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>L4 float32 vs L4 bfloat16</td>
      <td>64.9%</td>
      <td>33.7%</td>
      <td>1.4%</td>
      <td>0.99991</td>
    </tr>
    <tr>
      <td>T4 float32 vs T4 bfloat16</td>
      <td>64.9%</td>
      <td>33.7%</td>
      <td>1.4%</td>
      <td>0.99991</td>
    </tr>
    <tr>
      <td>AMX float32 vs AMX bfloat16</td>
      <td>62.3%</td>
      <td>35.9%</td>
      <td>1.8%</td>
      <td>0.99990</td>
    </tr>
  </tbody>
</table>

<p>About 65% of embedding values are identical between float32 and bfloat16. Of the rest, almost all differ by just 1 quantisation step out of 256. Cosine similarity averages 0.9999, and no pixel in the entire tile falls below 0.999.</p>

<p>The int8 quantisation itself introduces far more rounding than the float32-to-bfloat16 precision change.</p>

<h1>Framework Matters: PyTorch vs ONNX Runtime</h1>

<p>Previously, I found that <a href="https://www.tunbury.org/2026/02/15/ocaml-tessera/">ONNX outperformed PyTorch</a> by 14% on both GPU and CPU; however, on AMX hardware, that reverses completely.</p>

<p>Testing the ONNX Runtime 1.24.4 with both the original float32 model and a float16-converted variant vs PyTorch bfloat16:</p>

<table>
  <thead>
    <tr>
      <th>Threads</th>
      <th>PyTorch bfloat16</th>
      <th>ONNX float32</th>
      <th>ONNX float16</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>4</td>
      <td><strong>1.81</strong></td>
      <td>8.88</td>
      <td>8.91</td>
    </tr>
    <tr>
      <td>8</td>
      <td><strong>1.08</strong></td>
      <td>5.75</td>
      <td>5.57</td>
    </tr>
    <tr>
      <td>16</td>
      <td><strong>1.20</strong></td>
      <td>4.62</td>
      <td>4.33</td>
    </tr>
  </tbody>
</table>

<p><em>(ms/pixel, synthetic benchmark)</em></p>

<p>ONNX Runtime doesn’t appear to use AMX. Its float16 model runs no faster than its float32 model. ONNX Runtime’s MLAS library includes AMX-aware kernels, but there doesn’t seem to be an equivalent to PyTorch’s <code class="language-plaintext highlighter-rouge">autocast</code>. The model possibly could be explicitly exported with bfloat16 operations.</p>

<h1>Tuning Details</h1>

<p>For those who want to reproduce or adapt these results, here are the key tuning parameters we discovered.</p>

<h2>Thread Count</h2>

<p>On the 16-core AMX VM (32 vCPUs with hyperthreading):</p>

<table>
  <thead>
    <tr>
      <th>Threads</th>
      <th>bfloat16 ms/pixel</th>
      <th>float32 ms/pixel</th>
      <th>AMX speedup</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>7.07</td>
      <td>—</td>
      <td>—</td>
    </tr>
    <tr>
      <td>2</td>
      <td>3.48</td>
      <td>—</td>
      <td>—</td>
    </tr>
    <tr>
      <td>4</td>
      <td>1.81</td>
      <td>6.03</td>
      <td>3.3x</td>
    </tr>
    <tr>
      <td><strong>8</strong></td>
      <td><strong>1.08</strong></td>
      <td>3.06</td>
      <td><strong>2.8x</strong></td>
    </tr>
    <tr>
      <td>16</td>
      <td>1.20</td>
      <td>1.89</td>
      <td>1.6x</td>
    </tr>
    <tr>
      <td>32</td>
      <td>17.71</td>
      <td>—</td>
      <td>worse</td>
    </tr>
  </tbody>
</table>

<p>Hyperthreading seems to hurt AMX performance, and beyond 8 cores, the performance tails off. 8 physical cores with bfloat16 (1.08 ms/pixel) outperform 16 physical cores with float32 (1.89 ms/pixel).</p>

<h2>Batch Size</h2>

<p>On the 4-core AMX VM, all with bfloat16 autocast:</p>

<table>
  <thead>
    <tr>
      <th>Threads</th>
      <th>Batch 64</th>
      <th>Batch 128</th>
      <th>Batch 256</th>
      <th>Batch 512</th>
      <th>Batch 1024</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>5.55</td>
      <td>5.20</td>
      <td>5.06</td>
      <td>5.44</td>
      <td>5.95</td>
    </tr>
    <tr>
      <td>2</td>
      <td>3.26</td>
      <td>3.20</td>
      <td>2.79</td>
      <td>3.00</td>
      <td>3.15</td>
    </tr>
    <tr>
      <td>4</td>
      <td>1.87</td>
      <td>1.53</td>
      <td><strong>1.42</strong></td>
      <td>1.48</td>
      <td>1.64</td>
    </tr>
  </tbody>
</table>

<p><em>(ms/pixel)</em></p>

<p>A batch size of 256 achieves the best performance, confirming the results from the previous AMD EPYC benchmark. Presumably, this fits in L2/L3
cache where larger batches spill to main memory.</p>

<h1>Cost</h1>

<p>At the ~$1/hr price point, two options deliver similar throughput:</p>

<table>
  <thead>
    <tr>
      <th>Configuration</th>
      <th>Time/tile</th>
      <th>$/hr</th>
      <th>$/tile</th>
      <th>Tiles/$</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>T4 GPU, float32</td>
      <td>14.5 min</td>
      <td>$0.94</td>
      <td>$0.23</td>
      <td>4.4</td>
    </tr>
    <tr>
      <td>AMX CPU, bfloat16</td>
      <td>17.5 min</td>
      <td>$0.98</td>
      <td>$0.29</td>
      <td>3.5</td>
    </tr>
  </tbody>
</table>

<p>Stepping up to an A10 GPU (~$4/hr, Ada Lovelace generation) would likely process a tile in ~3 minutes with bfloat16, giving ~$0.20/tile. This would be 
slightly cheaper per tile, but at four times the hourly rate, they would need to run at capacity.</p>

<p>If you need tiles quickly, regardless of cost, an L4 or A10 with bfloat16 will process a tile in 3 minutes. If cost is a factor, and considering that the Tessera is embarrassingly parallel when you have an entire planet process, the T4 running the float32 model outperforms and undercuts the AMX bfloat16.</p>

<p>However, if you are prepared to use spot pricing, the AMX machines can be had at a substantial discount as low as $0.1785, while the T4 machines are in higher demand and cost $0.5176.</p>
