---
title: 'GPU vs CPU for ONNX Inference: NVIDIA L4 vs AMD EPYC 9965'
description: In a previous post, I compared the ONNX Runtime with PyTorch on the CPU
  and GPU. In this post, I take this to the extreme to see if a CPU can outpace the
  NVIDIA L4 GPU.
url: https://www.tunbury.org/2026/03/11/gpu-vs-cpu/
date: 2026-03-11T13:00:00-00:00
preview_image: https://www.tunbury.org/images/nvidia-l4.jpg
authors:
- Mark Elvers
source:
ignore:
---

<p>In a previous <a href="https://www.tunbury.org/2026/02/15/ocaml-tessera/">post</a>, I compared the <a href="https://github.com/mtelvers/onnxruntume">ONNX Runtime</a> with PyTorch on the CPU and GPU. In this post, I take this to the extreme to see if a CPU can outpace the NVIDIA L4 GPU.</p>

<p>I’m going to use my <a href="https://github.com/mtelvers/onnxruntume">OCaml bindings</a> to the <a href="https://onnxruntime.ai">ONNX Runtime</a> and benchmark the inference performance of <a href="https://github.com/ucam-eo/tessera">Tessera model</a> using a NVIDIA L4 GPU against an AMD EPYC 9965 192-core CPU with AVX-512 support.</p>

<h1>The Model</h1>

<p>The <a href="https://www.tunbury.org/2026/02/25/teserra-pipeline/">model</a> produces 128-dimensional embeddings from multi-temporal Sentinel-2 and Sentinel-1 satellite imagery. Each inference takes two inputs:</p>

<ul>
  <li>S2 input: <code class="language-plaintext highlighter-rouge">[batch, 40, 11]</code> 40 time-sampled Sentinel-2 observations across 11 bands</li>
  <li>S1 input: <code class="language-plaintext highlighter-rouge">[batch, 40, 3]</code> 40 time-sampled Sentinel-1 SAR observations across 3 channels</li>
</ul>

<p>The output is <code class="language-plaintext highlighter-rouge">[batch, 128]</code> embedding per pixel.</p>

<h1>The Benchmark</h1>

<p>For the benchmark, I’m going to use a minimal OCaml program that isolates pure ONNX Runtime inference, removing all data loading and preprocessing overhead. It pre-fills input tensors with dummy data and runs the model repeatedly in a loop:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">for</span> <span class="n">_</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">to</span> <span class="n">num_batches</span> <span class="o">-</span> <span class="mi">1</span> <span class="k">do</span>
  <span class="k">let</span> <span class="n">_outputs</span> <span class="o">=</span> <span class="nn">Onnxruntime</span><span class="p">.</span><span class="nn">Session</span><span class="p">.</span><span class="n">run_cached_ba</span> <span class="n">session</span>
    <span class="p">[</span><span class="o">|</span><span class="p">(</span><span class="n">s2_input</span><span class="o">,</span>
       <span class="p">[</span><span class="o">|</span> <span class="nn">Int64</span><span class="p">.</span><span class="n">of_int</span> <span class="n">bs</span><span class="p">;</span> <span class="nn">Int64</span><span class="p">.</span><span class="n">of_int</span> <span class="n">sample_size_s2</span><span class="p">;</span> <span class="mi">11</span><span class="nc">L</span> <span class="o">|</span><span class="p">]);</span>
      <span class="p">(</span><span class="n">s1_input</span><span class="o">,</span>
       <span class="p">[</span><span class="o">|</span> <span class="nn">Int64</span><span class="p">.</span><span class="n">of_int</span> <span class="n">bs</span><span class="p">;</span> <span class="nn">Int64</span><span class="p">.</span><span class="n">of_int</span> <span class="n">sample_size_s1</span><span class="p">;</span> <span class="mi">3</span><span class="nc">L</span> <span class="o">|</span><span class="p">])</span><span class="o">|</span><span class="p">]</span>
    <span class="o">~</span><span class="n">output_sizes</span><span class="o">:</span><span class="p">[</span><span class="o">|</span> <span class="n">bs</span> <span class="o">*</span> <span class="n">latent_dim</span> <span class="o">|</span><span class="p">]</span>
  <span class="k">in</span>
  <span class="bp">()</span>
<span class="k">done</span>
</code></pre></div></div>

<p>The initial tests use a batch size of 2048 and runs ten batches (20,480 pixels). Testing showed that this scaled linearly to longer runs. ONNX Runtime version is 1.24.1.</p>

<h1>Results</h1>

<h2>GPU vs CPU</h2>

<table>
  <thead>
    <tr>
      <th>&nbsp;</th>
      <th>GPU (NVIDIA L4)</th>
      <th>CPU (8 threads)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Total (10 batches)</td>
      <td>9.0s</td>
      <td>113.6s</td>
    </tr>
    <tr>
      <td>Per batch</td>
      <td>900ms</td>
      <td>11,360ms</td>
    </tr>
    <tr>
      <td>Per pixel</td>
      <td>0.44ms</td>
      <td>5.55ms</td>
    </tr>
    <tr>
      <td>GPU speedup</td>
      <td>12.6x</td>
      <td>baseline</td>
    </tr>
  </tbody>
</table>

<p>As we would expect, the GPU is faster: 12.6 times faster!</p>

<h2>CPU Thread Scaling</h2>

<p>I picked 8 threads at random for the first test. How does it compare across a range of thread counts, as more cores should give better performance?</p>

<table>
  <thead>
    <tr>
      <th>Threads</th>
      <th>ms/batch</th>
      <th>Speedup vs 1 thread</th>
      <th>vs GPU (900ms)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>48,168</td>
      <td>1.0x</td>
      <td>53.5x slower</td>
    </tr>
    <tr>
      <td>4</td>
      <td>16,494</td>
      <td>2.9x</td>
      <td>18.3x slower</td>
    </tr>
    <tr>
      <td>8</td>
      <td>11,360</td>
      <td>4.2x</td>
      <td>12.6x slower</td>
    </tr>
    <tr>
      <td>16</td>
      <td>11,138</td>
      <td>4.3x</td>
      <td>12.4x slower</td>
    </tr>
    <tr>
      <td>32</td>
      <td>10,383</td>
      <td>4.6x</td>
      <td>11.5x slower</td>
    </tr>
    <tr>
      <td>64</td>
      <td>10,389</td>
      <td>4.6x</td>
      <td>11.5x slower</td>
    </tr>
    <tr>
      <td>128</td>
      <td>10,161</td>
      <td>4.7x</td>
      <td>11.3x slower</td>
    </tr>
    <tr>
      <td>192</td>
      <td>9,989</td>
      <td>4.8x</td>
      <td>11.1x slower</td>
    </tr>
  </tbody>
</table>

<h1>Analysis</h1>

<p>The CPU thread scaling plateaus at about 16 threads. Going from 1 to 8 threads gave a 4.2x speedup, but doubling further to 16 adds only 2%. Adding the remaining 176 cores contributes almost nothing. This will be investigated below.</p>

<p>The GPU wins by 11-12x for a single job. Even using all 192 cores of the EPYC 9965 in a single process, the L4 GPU is still 11x faster. A single CPU process can only effectively use one or two NUMA nodes’ memory controllers (~75-150 GB/s), while the L4 delivers 300 GB/s to a single computation. More on this later.</p>

<p>A single Sentinel-2 MGRS tile at 10m resolution is 10,980 x 10,980 pixels. Processing this with 1,024 x 1,024 blocks yields 121 blocks averaging ~500K-1M pixels each. At these single-job rates, a full tile takes approximately:</p>

<ul>
  <li>GPU: ~16 hours (inference only)</li>
  <li>CPU (8 threads): ~200 hours</li>
</ul>

<p>These projections get revisited after optimisation, and the final numbers look very different!</p>

<h1>What About PyTorch?</h1>

<p>An obvious question: is this an ONNX Runtime with OCaml bindings, or does native PyTorch show the same gap? An equivalent benchmark using PyTorch 2.10 with CUDA 12.6 loads the original checkpoint directly and runs the same model architecture with dummy tensors:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">with</span> <span class="n">torch</span><span class="p">.</span><span class="n">no_grad</span><span class="p">():</span>
    <span class="k">for</span> <span class="n">_</span> <span class="ow">in</span> <span class="nb">range</span><span class="p">(</span><span class="n">num_batches</span><span class="p">):</span>
        <span class="n">_</span> <span class="o">=</span> <span class="n">model</span><span class="p">(</span><span class="n">s2_input</span><span class="p">,</span> <span class="n">s1_input</span><span class="p">)</span>
</code></pre></div></div>

<h2>PyTorch vs ONNX Runtime</h2>

<table>
  <thead>
    <tr>
      <th>Framework</th>
      <th>GPU (L4)</th>
      <th>CPU (8 threads)</th>
      <th>GPU speedup</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ONNX Runtime 1.24.1</td>
      <td>9.0s (900 ms/batch)</td>
      <td>113.6s (11,360 ms/batch)</td>
      <td>12.6x</td>
    </tr>
    <tr>
      <td>PyTorch 2.10</td>
      <td>10.3s (1,026 ms/batch)</td>
      <td>140.2s (14,016 ms/batch)</td>
      <td>13.6x</td>
    </tr>
  </tbody>
</table>

<p>The results speak for themselves: across both frameworks, the GPU is the clear winner, 12-14x faster than the CPU. ONNX Runtime does edge out PyTorch by about 14%. This is likely due to graph optimisations applied during model export, but the dominant factor is GPU vs CPU, not the inference framework!</p>

<h1>Does Batch Size Matter?</h1>

<p>All the results above used a batch size of 2048. The GPU has thousands of CUDA cores while the CPU has far fewer. Should the CPU use smaller batches that fit better in cache?</p>

<p>Running a loop through the batch sizes from 32 to 5120 on both the CPU and GPU gave these results.</p>

<table>
  <thead>
    <tr>
      <th>Batch Size</th>
      <th>CPU ms/pixel</th>
      <th>GPU ms/pixel</th>
      <th>GPU speedup</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>32</td>
      <td>5.62</td>
      <td>0.63</td>
      <td>8.9x</td>
    </tr>
    <tr>
      <td>64</td>
      <td>5.56</td>
      <td>0.49</td>
      <td>11.3x</td>
    </tr>
    <tr>
      <td>128</td>
      <td>5.40</td>
      <td>0.45</td>
      <td>12.0x</td>
    </tr>
    <tr>
      <td>256</td>
      <td><strong>5.05</strong></td>
      <td>0.44</td>
      <td>11.5x</td>
    </tr>
    <tr>
      <td>512</td>
      <td>5.16</td>
      <td>0.46</td>
      <td>11.2x</td>
    </tr>
    <tr>
      <td>1024</td>
      <td>6.11</td>
      <td>0.45</td>
      <td>13.6x</td>
    </tr>
    <tr>
      <td>2048</td>
      <td>6.07</td>
      <td>0.44</td>
      <td>13.8x</td>
    </tr>
    <tr>
      <td>4096</td>
      <td>6.18</td>
      <td>0.44</td>
      <td>14.0x</td>
    </tr>
    <tr>
      <td>5120</td>
      <td>—</td>
      <td><strong>0.43</strong></td>
      <td>—</td>
    </tr>
    <tr>
      <td>6144+</td>
      <td>—</td>
      <td>OOM</td>
      <td>—</td>
    </tr>
  </tbody>
</table>

<p>The CPU has an optimum batch size of 256 (5.05 vs 6.07 ms/pixel). At small batch sizes the working set fits in cache, avoiding expensive main memory accesses. Above 1024, performance degrades as intermediate tensors spill to DRAM.</p>

<p>For the GPU, the per-pixel throughput is nearly flat from batch size 128 upwards, with a marginal improvement as the size increases. The maximum batch size is constrained by VRAM (24GB).</p>

<p>Even comparing each device at its optimal batch size (CPU at 256, GPU at 5120), the GPU is 11.5x faster. Tuning the batch size helps the CPU modestly but does not bridge the gap.</p>

<h1>NUMA Topology: The Hidden Variable</h1>

<p>The thread scaling results above were surprisingly poor with 192 threads barely faster than 8. Could NUMA (Non-Uniform Memory Access) explain this?</p>

<p>The AMD EPYC 9965 is a 2-socket system with 24 NUMA nodes (12 per socket). Each node has 16 physical cores, its own 32MB L3 cache, and a local memory controller serving ~128GB of DDR5. The key insight is in the distance table:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>node distances:
       0    1    ...  12   13   ...
  0:  10   11        32   32
  1:  11   10        32   32
 12:  32   32        10   11
 13:  32   32        11   10
</code></pre></div></div>

<p>Accessing memory on the same node costs 10 (local). A different node on the same socket costs 11 (10% penalty). But crossing to the other socket costs 32 aka a 3.2x latency penalty. When ONNX Runtime spawns a large number of threads without NUMA awareness, they scatter across nodes and sockets, and every shared tensor access becomes a cross-socket round trip.</p>

<p>To test this, all threads and memory can be pinned to a single NUMA node using <code class="language-plaintext highlighter-rouge">numactl</code>:</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code>numactl <span class="nt">--cpunodebind</span><span class="o">=</span>17 <span class="nt">--membind</span><span class="o">=</span>17 ./bench_onnx.exe <span class="se">\</span>
    <span class="nt">--model</span> tessera_model.onnx <span class="nt">--batch_size</span> 256 <span class="nt">--num_threads</span> 16
</code></pre></div></div>

<h2>NUMA-Pinned Thread Scaling (Node 17, batch_size=256)</h2>

<table>
  <thead>
    <tr>
      <th>Threads</th>
      <th>ms/pixel</th>
      <th>Speedup vs 4</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>4</td>
      <td>6.45</td>
      <td>1.0x</td>
    </tr>
    <tr>
      <td>8</td>
      <td>4.18</td>
      <td>1.5x</td>
    </tr>
    <tr>
      <td>12</td>
      <td>3.75</td>
      <td>1.7x</td>
    </tr>
    <tr>
      <td>16</td>
      <td><strong>3.38</strong></td>
      <td>1.9x</td>
    </tr>
  </tbody>
</table>

<p>Within a single NUMA node, scaling is nearly linear. All 16 cores share the same L3 cache and memory controller with no cross-node traffic.</p>

<h2>Distributing Across NUMA Nodes</h2>

<p>Pinning to one node gives great per-core efficiency, but limits throughput to a single memory controller’s bandwidth. Would spreading across multiple nodes aggregate their bandwidth and cache?</p>

<p>Using <code class="language-plaintext highlighter-rouge">numactl --interleave</code> to stripe memory across nodes while binding threads to the corresponding cores:</p>

<table>
  <thead>
    <tr>
      <th>NUMA Nodes</th>
      <th>Cores</th>
      <th>ms/pixel</th>
      <th>vs 1 node</th>
      <th>vs GPU</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>16</td>
      <td>3.35</td>
      <td>1.00x</td>
      <td>7.8x</td>
    </tr>
    <tr>
      <td><strong>2</strong></td>
      <td><strong>32</strong></td>
      <td><strong>3.09</strong></td>
      <td><strong>1.08x</strong></td>
      <td><strong>7.2x</strong></td>
    </tr>
    <tr>
      <td>4</td>
      <td>64</td>
      <td>3.27</td>
      <td>1.02x</td>
      <td>7.6x</td>
    </tr>
    <tr>
      <td>6</td>
      <td>96</td>
      <td>3.45</td>
      <td>0.97x</td>
      <td>8.0x</td>
    </tr>
    <tr>
      <td>12 (full socket)</td>
      <td>192</td>
      <td>3.64</td>
      <td>0.92x</td>
      <td>8.5x</td>
    </tr>
  </tbody>
</table>

<p>Two nodes is the best with a modest 8% improvement from the extra bandwidth. However, beyond that, cross-node synchronisation overhead outweighs the gains.</p>

<p>Quickly verifying that the optimal batch size still holds in a 2-node configuration:</p>

<table>
  <thead>
    <tr>
      <th>Batch Size</th>
      <th>2 nodes (32 cores) ms/pixel</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>128</td>
      <td>3.09</td>
    </tr>
    <tr>
      <td>256</td>
      <td><strong>3.07</strong></td>
    </tr>
    <tr>
      <td>512</td>
      <td>3.18</td>
    </tr>
    <tr>
      <td>1024</td>
      <td>3.58</td>
    </tr>
    <tr>
      <td>2048</td>
      <td>3.79</td>
    </tr>
  </tbody>
</table>

<h2>Parallel Jobs: Exploiting the Full Machine</h2>

<p>As shown above, a single inference job can’t efficiently use more than 1-2 NUMA nodes. But satellite tile processing is embarrassingly parallel as each tile, block and pixel is independent. What happens when multiple NUMA-pinned jobs run simultaneously?</p>

<p>I tested two strategies:</p>

<ul>
  <li>2 NUMA nodes per job (32 cores, the best single-job configuration), and</li>
  <li>1 NUMA node per job (16 cores, maximum parallelism).</li>
</ul>

<h3>2 Nodes Per Job (32 cores each)</h3>

<table>
  <thead>
    <tr>
      <th>Jobs</th>
      <th>Wall Time</th>
      <th>Total Pixels</th>
      <th>Per-job ms/pixel</th>
      <th>Aggregate ms/pixel</th>
      <th>vs GPU</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>31s</td>
      <td>10,240</td>
      <td>3.07</td>
      <td>3.07</td>
      <td>7.1x slower</td>
    </tr>
    <tr>
      <td>2</td>
      <td>34s</td>
      <td>20,480</td>
      <td>3.29</td>
      <td>1.66</td>
      <td>3.9x slower</td>
    </tr>
    <tr>
      <td>4</td>
      <td>36s</td>
      <td>40,960</td>
      <td>3.54</td>
      <td>0.89</td>
      <td>2.1x slower</td>
    </tr>
    <tr>
      <td>6</td>
      <td>36s</td>
      <td>61,440</td>
      <td>3.35</td>
      <td>0.59</td>
      <td>1.4x slower</td>
    </tr>
    <tr>
      <td><strong>12</strong></td>
      <td><strong>47s</strong></td>
      <td><strong>122,880</strong></td>
      <td><strong>4.33</strong></td>
      <td><strong>0.38</strong></td>
      <td><strong>1.1x faster</strong></td>
    </tr>
  </tbody>
</table>

<h3>1 Node Per Job (16 cores each)</h3>

<table>
  <thead>
    <tr>
      <th>Jobs</th>
      <th>Wall Time</th>
      <th>Total Pixels</th>
      <th>Per-job ms/pixel</th>
      <th>Aggregate ms/pixel</th>
      <th>vs GPU</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>34s</td>
      <td>10,240</td>
      <td>3.33</td>
      <td>3.33</td>
      <td>7.7x slower</td>
    </tr>
    <tr>
      <td>2</td>
      <td>36s</td>
      <td>20,480</td>
      <td>3.54</td>
      <td>1.77</td>
      <td>4.1x slower</td>
    </tr>
    <tr>
      <td>4</td>
      <td>40s</td>
      <td>40,960</td>
      <td>3.86</td>
      <td>0.97</td>
      <td>2.3x slower</td>
    </tr>
    <tr>
      <td>6</td>
      <td>41s</td>
      <td>61,440</td>
      <td>3.96</td>
      <td>0.67</td>
      <td>1.6x slower</td>
    </tr>
    <tr>
      <td>12</td>
      <td>42s</td>
      <td>122,880</td>
      <td>4.00</td>
      <td>0.34</td>
      <td>1.3x faster</td>
    </tr>
    <tr>
      <td><strong>24</strong></td>
      <td><strong>55s</strong></td>
      <td><strong>245,760</strong></td>
      <td><strong>5.21</strong></td>
      <td><strong>0.22</strong></td>
      <td><strong>2.0x faster</strong></td>
    </tr>
  </tbody>
</table>

<p>The 1-node-per-job strategy wins decisively at this scale. Each individual job is slightly slower (3.33 vs 3.07 ms/pixel), but with 24 jobs instead of 12, the aggregate throughput is twice as fast as the GPU!</p>

<p>I am impressed that the scaling is so linear: going from 1 to 24 jobs, the wall time only increases from 34s to 55s while processing 24 times the data. The key is that each job is fully contained within its NUMA node using its own 16 cores, 32MB L3 cache, and memory controller with zero
cross-node traffic.</p>

<h3>Summary</h3>

<table>
  <thead>
    <tr>
      <th>Configuration</th>
      <th>Per-job ms/pixel</th>
      <th>Aggregate ms/pixel</th>
      <th>vs GPU</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Default (8 threads, batch=2048)</td>
      <td>5.55</td>
      <td>5.55</td>
      <td>12.6x slower</td>
    </tr>
    <tr>
      <td>Tuned batch (8 threads, batch=256)</td>
      <td>5.05</td>
      <td>5.05</td>
      <td>11.5x slower</td>
    </tr>
    <tr>
      <td>NUMA 1 node, 16 threads, batch=256</td>
      <td>3.33</td>
      <td>3.33</td>
      <td>7.7x slower</td>
    </tr>
    <tr>
      <td>NUMA 2 nodes, 32 threads, batch=256</td>
      <td>3.07</td>
      <td>3.07</td>
      <td>7.1x slower</td>
    </tr>
    <tr>
      <td>12x NUMA jobs (2 nodes each)</td>
      <td>4.33</td>
      <td>0.38</td>
      <td>1.1x faster</td>
    </tr>
    <tr>
      <td><strong>24x NUMA jobs (1 node each)</strong></td>
      <td><strong>5.21</strong></td>
      <td><strong>0.22</strong></td>
      <td><strong>2.0x faster</strong></td>
    </tr>
    <tr>
      <td>GPU (NVIDIA L4)</td>
      <td>0.43</td>
      <td>0.43</td>
      <td>baseline</td>
    </tr>
  </tbody>
</table>

<p>The optimisation improved performance by 25 times, making the job twice as fast as the GPU.</p>

<p>Revisiting the full-tile projection from earlier: at 0.22 ms/pixel aggregate, the optimised CPU processes a 120-million-pixel MGRS tile in approximately 7.5 hours, about half the GPU’s ~16 hours.</p>

<h1>Conclusion</h1>

<p>You <em>can</em> beat a GPU with enough CPU cores if you take into account the NUMA topology.</p>

<p>Of course, the GPU is the easy option; adding <code class="language-plaintext highlighter-rouge">--cuda 0</code> to the command line gives you 0.43 ms/pixel with zero tuning. The CPU approach took a lot more effort to tune the batch size and use <code class="language-plaintext highlighter-rouge">numactl</code> to pin work to the nodes.</p>
