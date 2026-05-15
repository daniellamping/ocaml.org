---
title: Machine Learning in OxCaml
description: "The inspiration for this post came from Dave\u2019s Garage YouTube Channel.
  It featured dbrll/ATTN-11, which used machine learning on a PDP-11 to reverse a
  sequence of numbers. I\u2019m not (quite) old enough to remember the PDP-11, but
  the idea of a minimal machine learning example was intriguing, particularly as an
  aid to understanding."
url: https://www.tunbury.org/2026/05/07/attn-ox/
date: 2026-05-07T16:00:00-00:00
preview_image: https://www.tunbury.org/images/oxcaml.png
authors:
- Mark Elvers
source:
ignore:
---

<p>The inspiration for this post came from <a href="https://youtu.be/OUE3FSIk46g?si=TNm6tYvX2BsorxQ6">Dave’s Garage YouTube Channel</a>. It featured <a href="https://github.com/dbrll/ATTN-11">dbrll/ATTN-11</a>, which used machine learning on a PDP-11 to reverse a sequence of numbers. I’m not (quite) old enough to remember the PDP-11, but the idea of a minimal machine learning example was intriguing, particularly as an aid to understanding.</p>

<p>This post takes you through the <a href="https://github.com/mtelvers/attn-ox">mtelvers/attn-ox</a> OxCaml program that learns to add single hex digits. The excercise was here was for me to understand the building blocks of machine learning. Claude wrote the code. The system is small enough that everything is concrete: every line of code maps to one of the boxes in the diagrams below.</p>

<h1>1. The problem</h1>

<p>Write a function that takes two single hex digits <code class="language-plaintext highlighter-rouge">x</code> and <code class="language-plaintext highlighter-rouge">y</code> and returns the sum <code class="language-plaintext highlighter-rouge">x + y</code>, e.g. <code class="language-plaintext highlighter-rouge">f + f = 1e</code>.</p>

<p>Rather than a single line of code, the machine-learning approach starts with random floats, which are updated by gradient descent until the function gets all 256 inputs right.</p>

<p>The idea is to see the smallest possible version of “a model learns from examples” running end-to-end. The architecture
of transformer, attention, layer norm, and residual connections are the same as used in much larger models.</p>

<h1>2. Overview</h1>

<p>The model is not the kind of function you would usually write for addition (something like <code class="language-plaintext highlighter-rouge">int -&gt; int -&gt; int</code>). It is a function on token sequences:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>input  : an array of 8 integers (token IDs from a 32-token vocabulary)
output : an array of 8 probability distributions, each over the 32 tokens
</code></pre></div></div>

<p>We encode the question <code class="language-plaintext highlighter-rouge">x + y = ?</code> as a sequence of 8 token IDs:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>positions:   0    1    2    3    4    5    6    7
ids:         BOS  x    +    y    =    c1   c2   PAD
</code></pre></div></div>

<p>The 16 hex digits get token IDs 0..15 (so the digit <code class="language-plaintext highlighter-rouge">0</code> is token 0, digit <code class="language-plaintext highlighter-rouge">f</code> is token 15). Then four reserved tokens follow with the next four IDs: <code class="language-plaintext highlighter-rouge">+</code> = 16, <code class="language-plaintext highlighter-rouge">=</code> = 17, <code class="language-plaintext highlighter-rouge">BOS</code> (begin of sequence) = 18, <code class="language-plaintext highlighter-rouge">PAD</code> = 19. The vocabulary contains 20 used tokens, but we round up to 32 to the next power of 2. The unused slots <code class="language-plaintext highlighter-rouge">20</code>..<code class="language-plaintext highlighter-rouge">31</code> never appear in inputs or as predicted answers.</p>

<p>For input <code class="language-plaintext highlighter-rouge">8 + a</code>, we already know the answer is <code class="language-plaintext highlighter-rouge">12</code> (= 8 + 10 = 18, and in hex 18 is written <code class="language-plaintext highlighter-rouge">12</code>). The training sequence is:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>ids = [18; 8; 16; 10; 17; 1; 2; 19]   (BOS, 8, +, a, =, 1, 2, PAD)
</code></pre></div></div>

<p>We hand the whole sequence (including the answer) to the model during training. The model produces 8 probability distributions, one at each position, each predicting “what comes next?”. A probability distribution here is a row of 32 non-negative floats that sum to 1. There is one float per vocabulary token, expressing the model’s confidence that this token is next.</p>

<p>Here is the actual 8 × 32 output of a trained model on our <code class="language-plaintext highlighter-rouge">8 + a</code> example. Each row is a probability distribution; each row sums to 1.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>                       0     1     2     3     4     5     6     7     8     9     a     b     c     d     e     f     +     =   BOS   PAD    20   ..    31
  pos 0 (BOS)       0.99     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·   ..     ·
  pos 1 (8  )       0.99     ·     ·     ·     ·     ·   0.01    ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·   ..     ·
  pos 2 (+  )       0.86  0.02     ·     ·     ·     ·     ·  0.08  0.03     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·   ..     ·
  pos 3 (a  )       1.00     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·   ..     ·
  pos 4 (=  )          ·  1.00     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·   ..     ·
  pos 5 (c1 )          ·     ·  0.99     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·   ..     ·
  pos 6 (c2 )       1.00     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·   ..     ·
  pos 7 (PAD)       1.00     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·     ·   ..     ·
</code></pre></div></div>

<p>At position 4, the model assigns a probability of 1.00 to token <code class="language-plaintext highlighter-rouge">1</code> (the high digit of <code class="language-plaintext highlighter-rouge">12</code>), and at position 5, it assigns 0.99 to token <code class="language-plaintext highlighter-rouge">2</code> (the low digit). Predictions at other positions can be arbitrary, as these are not scored during training.</p>

<p>This concept of predicting the next token but only scoring the ones we care about is how GPT-style language models are trained.</p>

<h1>3. Pipeline</h1>

<p>At the top level, the model is a pipeline of three stages: embedding, a transformer block, and an output projection.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>          ids (length 8)
           │
           ▼
  ┌────────────────┐
  │   Embedding    │   integers → vectors                     (§5)
  └────────┬───────┘
           │ X (8 × 32)
           ▼
  ┌────────────────┐
  │   Transformer  │   vectors → vectors
  │      block     │   (we'll open this up in §7)
  └────────┬───────┘
           │ Y (8 × 32)
           ▼
  ┌────────────────┐
  │     Output     │   vectors → predictions                  (§6)
  │   projection   │
  └────────┬───────┘
           │ logits (8 × 32)
           ▼
   softmax + cross-entropy at positions 4 and 5               (§11)
</code></pre></div></div>

<p>Two numbers govern almost every shape inside.</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">seq_len = 8</code> is the length of every input sequence.</li>
  <li><code class="language-plaintext highlighter-rouge">d_model = 32</code> is the width of the model’s internal representation.</li>
</ul>

<p>Every token, once embedded, is described by a vector of 32 floats. Every intermediate matrix inside the model is <code class="language-plaintext highlighter-rouge">seq_len × d_model = 8 × 32</code>. Larger <code class="language-plaintext highlighter-rouge">d_model</code> means more capacity but more compute and memory; 32 is enough for this task.</p>

<p>Hyperparameters are settings we pick before training and parameters are the floats that gradient descent adjusts. The full set of hyperparameters lives in <code class="language-plaintext highlighter-rouge">Train.default_config</code>.</p>

<table>
  <thead>
    <tr>
      <th>&nbsp;</th>
      <th>value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">vocab_size</code></td>
      <td>32</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">seq_len</code></td>
      <td>8</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">d_model</code></td>
      <td>32</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">n_heads</code></td>
      <td>2</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">head_dim</code></td>
      <td>16</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">d_ff</code></td>
      <td>128</td>
    </tr>
  </tbody>
</table>

<p>We will see the use of each of these in the coming sections.</p>

<p>The three boxes map to OCaml modules:</p>

<table>
  <thead>
    <tr>
      <th>Box</th>
      <th>File</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Embedding</td>
      <td><code class="language-plaintext highlighter-rouge">lib/embedding.ml</code></td>
    </tr>
    <tr>
      <td>Transformer block</td>
      <td><code class="language-plaintext highlighter-rouge">lib/transformer_block.ml</code>, which composes <code class="language-plaintext highlighter-rouge">layernorm.ml</code>, <code class="language-plaintext highlighter-rouge">attention.ml</code>, <code class="language-plaintext highlighter-rouge">ffn.ml</code></td>
    </tr>
    <tr>
      <td>Output projection</td>
      <td><code class="language-plaintext highlighter-rouge">lib/model.ml</code></td>
    </tr>
  </tbody>
</table>

<h1>4. Tensors</h1>

<p>Every intermediate value is a 2D matrix of floats, a <code class="language-plaintext highlighter-rouge">Tensor.t</code>:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">type</span> <span class="n">t</span> <span class="o">=</span> <span class="p">{</span> <span class="n">rows</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span> <span class="n">cols</span> <span class="o">:</span> <span class="kt">int</span><span class="p">;</span> <span class="n">data</span> <span class="o">:</span> <span class="kt">float</span> <span class="kt">array</span> <span class="p">}</span>
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">data</code> holds the cells row by row, so element <code class="language-plaintext highlighter-rouge">(i, j)</code> lives at <code class="language-plaintext highlighter-rouge">data.(i * cols + j)</code>. <code class="language-plaintext highlighter-rouge">Array.make_matrix</code> or <code class="language-plaintext highlighter-rouge">Bigarray.Array2</code> would also work; we use a flat <code class="language-plaintext highlighter-rouge">float array</code>. The fields are exposed so the kernels in <code class="language-plaintext highlighter-rouge">ops.ml</code> can index into <code class="language-plaintext highlighter-rouge">data</code> directly.</p>

<p><code class="language-plaintext highlighter-rouge">lib/ops.ml</code> provides the building-block kernels:</p>

<table>
  <thead>
    <tr>
      <th>Function</th>
      <th>Computes</th>
      <th>Used by</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">matmul</code></td>
      <td><code class="language-plaintext highlighter-rouge">C = A @ B</code></td>
      <td>every linear layer</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">matmul_at_b_t</code></td>
      <td><code class="language-plaintext highlighter-rouge">C = A @ B^T</code></td>
      <td>attention scores, output projection</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">accum_xt_y</code></td>
      <td><code class="language-plaintext highlighter-rouge">dW += X^T @ dY</code></td>
      <td>every linear layer’s backward</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">accum_y_wt</code></td>
      <td><code class="language-plaintext highlighter-rouge">dX += dY @ W^T</code></td>
      <td>every linear layer’s backward</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">add_inplace</code></td>
      <td><code class="language-plaintext highlighter-rouge">dst += src</code></td>
      <td>residual connections</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">softmax_rows</code></td>
      <td>row-wise softmax</td>
      <td>attention</td>
    </tr>
    <tr>
      <td><code class="language-plaintext highlighter-rouge">apply_causal_mask</code></td>
      <td>zero out upper triangle</td>
      <td>attention</td>
    </tr>
  </tbody>
</table>

<p>Section 19 (SIMD) shows how these are implemented.</p>

<h1>5. Embedding: turn integers into vectors</h1>

<p>The first box turns each integer token into a 32-element float vector (the <code class="language-plaintext highlighter-rouge">d_model = 32</code> from §3).</p>

<p>The embedding has two learned matrices:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>E (32 × 32)   one row per token in the vocabulary, 32 floats per row
P (8  × 32)   one row per sequence position
</code></pre></div></div>

<p>Both are initialised to small random numbers (Gaussian, scale 0.02).</p>

<p>For input <code class="language-plaintext highlighter-rouge">ids = [18; 8; 16; 10; 17; 1; 2; 19]</code> (our <code class="language-plaintext highlighter-rouge">8 + a</code> example), the embedded matrix <code class="language-plaintext highlighter-rouge">X</code> (the same <code class="language-plaintext highlighter-rouge">X</code> that leaves the embedding box in §3’s diagram) is built row by row:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>X[i] = E[ids[i]] + P[i]      for each i
</code></pre></div></div>

<p>The model needs both <code class="language-plaintext highlighter-rouge">E</code> (which token is at position i) and <code class="language-plaintext highlighter-rouge">P</code> (which position this is). Without <code class="language-plaintext highlighter-rouge">P</code>, the model would treat <code class="language-plaintext highlighter-rouge">[8, +, a]</code> and <code class="language-plaintext highlighter-rouge">[a, +, 8]</code> identically. There is no other source of order information in the architecture.</p>

<p><code class="language-plaintext highlighter-rouge">E</code> and <code class="language-plaintext highlighter-rouge">P</code> are learned. They start as random noise. After training, rows of <code class="language-plaintext highlighter-rouge">E</code> corresponding to digit tokens end up encoding the token’s numeric value in some 32-dimensional way the rest of the model can use.</p>

<p>(Implementation: <code class="language-plaintext highlighter-rouge">Embedding.forward</code> and <code class="language-plaintext highlighter-rouge">Embedding.backward</code> in <code class="language-plaintext highlighter-rouge">lib/embedding.ml</code>.)</p>

<h1>6. Output projection</h1>

<p>At the other end of the model, after the transformer block, we have a matrix <code class="language-plaintext highlighter-rouge">Y'</code> of shape <code class="language-plaintext highlighter-rouge">(8, 32)</code>. That’s one 32-element vector per sequence position. The
output projection turns each 32-element vector into 32 logits, one per vocabulary token.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>logits[i] = Y'[i] · E^T      (using E from §5, transposed)
</code></pre></div></div>

<p>A logit is just an unnormalised score: any real number, positive or negative, big or small. By itself, it doesn’t mean a probability. To turn a row of 32 logits into a probability distribution (32 non-negative values summing to 1), we apply softmax. We will look at softmax in more detail in §11, where the model’s error is computed; it’s also what produced the 8 × 32 matrix you saw right at the start in §2.</p>

<p>We use the embedding matrix <code class="language-plaintext highlighter-rouge">E</code> as the output projection, which is called weight tying. It saves parameters and ties the input and output representations of each token.</p>

<p>The backward pass for the embedding has to add contributions from both its uses (lookup and unembedding):</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* Model.backward, model.ml *)</span>
<span class="nn">Ops</span><span class="p">.</span><span class="n">matmul</span>       <span class="o">~</span><span class="n">a</span><span class="o">:</span><span class="n">d_logits</span> <span class="o">~</span><span class="n">b</span><span class="o">:</span><span class="n">e</span> <span class="o">~</span><span class="n">c</span><span class="o">:</span><span class="n">d_y_norm</span><span class="p">;</span>        <span class="c">(* dY = d_logits @ E *)</span>
<span class="nn">Ops</span><span class="p">.</span><span class="n">accum_xt_y</span>   <span class="o">~</span><span class="n">x</span><span class="o">:</span><span class="n">d_logits</span> <span class="o">~</span><span class="n">dy</span><span class="o">:</span><span class="n">t</span><span class="o">.</span><span class="n">cache_y_norm</span> <span class="o">~</span><span class="n">dw</span><span class="o">:</span><span class="n">de</span><span class="p">;</span>
                                                       <span class="c">(* dE += d_logits^T @ Y *)</span>
<span class="o">...</span> <span class="n">later</span> <span class="o">...</span>
<span class="nn">Embedding</span><span class="p">.</span><span class="n">backward</span> <span class="n">t</span><span class="o">.</span><span class="n">embed</span> <span class="o">~</span><span class="n">d_out</span><span class="o">:</span><span class="n">d_x</span><span class="p">;</span>                 <span class="c">(* dE += scatter from inputs *)</span>
</code></pre></div></div>

<p>Now that we have covered the input and output, we will now look at the transformer.</p>

<h1>7. Transformer</h1>

<p>Every operation in the transformer operates on an 8 × 32 matrix and produces a new 8 × 32 matrix. <code class="language-plaintext highlighter-rouge">X</code> is generated by the embedding stage and <code class="language-plaintext highlighter-rouge">Y'</code> is used by the output projection.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>           X  (8 × 32)
           │
   ┌───────┤
   │       ▼
   │   LayerNorm
   │       │
   │       ▼
   │   Attention
   │       │
   │       ▼
   └─────► ⊕  X' = X + Attention(LayerNorm(X))
           │
   ┌───────┤
   │       ▼
   │   LayerNorm
   │       │
   │       ▼
   │  Feed-forward
   │       │
   │       ▼
   └─────► ⊕  Y  = X' + Feed-forward(LayerNorm(X'))
           │
           ▼
           Y  (8 × 32)
           │
   --- end of transformer block ---
           │
           ▼
       LayerNorm                  one more stabilising step
           │                      before the output projection
           ▼
           Y' (8 × 32)            this is what §6 receives
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">X</code> is processed through the LayerNorm and Attention steps, and then <code class="language-plaintext highlighter-rouge">X</code> is added in creating a residual connection. The input to the second stage is <code class="language-plaintext highlighter-rouge">X' = X + Attention(LayerNorm(X))</code>. This pattern is repeated with <code class="language-plaintext highlighter-rouge">X' being processed via LayerNorm and Feed-forward before being accumulated again with </code>Y  = X’ + Feed-forward(LayerNorm(X’)).</p>

<p>The residual connection matters, as it provides an identity highway to carrying <code class="language-plaintext highlighter-rouge">X</code> straight through, with the operation’s output added in. The model can ignore an operation entirely by training it to output near-zero. Gradients flow back along the highway in addition to through each operation, which makes optimisation much easier.</p>

<p>The two <code class="language-plaintext highlighter-rouge">+</code>s in the diagram are literally these calls in the code:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c">(* Transformer_block.forward, transformer_block.ml *)</span>
<span class="nn">Ops</span><span class="p">.</span><span class="n">add_inplace</span> <span class="o">~</span><span class="n">dst</span><span class="o">:</span><span class="n">t</span><span class="o">.</span><span class="n">cache_x_mid</span> <span class="o">~</span><span class="n">src</span><span class="o">:</span><span class="n">t</span><span class="o">.</span><span class="n">cache_attn_out</span><span class="p">;</span>  <span class="c">(* X' = X + Attn(LN1(X)) *)</span>
<span class="nn">Ops</span><span class="p">.</span><span class="n">add_inplace</span> <span class="o">~</span><span class="n">dst</span><span class="o">:</span><span class="n">out</span> <span class="o">~</span><span class="n">src</span><span class="o">:</span><span class="n">t</span><span class="o">.</span><span class="n">cache_ffn_out</span><span class="p">;</span>             <span class="c">(* Y  = X' + FFN(LN2(X')) *)</span>
</code></pre></div></div>

<p>The next three sections explain LayerNorm, Attention, and Feed-forward one at a time.</p>

<h1>8. LayerNorm</h1>

<p>LayerNorm operates independently on each row of its input matrix. Let <code class="language-plaintext highlighter-rouge">r</code> be one such row, a 32-element vector (i.e. <code class="language-plaintext highlighter-rouge">X[i]</code> for some <code class="language-plaintext highlighter-rouge">i</code>). The output row <code class="language-plaintext highlighter-rouge">r'</code> is:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>mean   = sum(r) / 32
var    = sum((r - mean)^2) / 32
r'[j]  = γ[j] * (r[j] - mean) / sqrt(var + 1e-5)  +  β[j]
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">γ</code> and <code class="language-plaintext highlighter-rouge">β</code> are learned 32-element vectors, one of each and are shared across every row of the input. Initialised to 1 and 0, respectively. So a LayerNorm has 64 trainable floats total, regardless of how many rows it sees. The rest of the operation has no learned parameters; it just rescales each row to have a mean of 0 and a variance of 1, then applies the same learned per-feature scale and shift to every row.</p>

<p>LayerNorm is a stabiliser that keeps the floats inside the model from drifting to large or too small during training.</p>

<p>LayerNorm is used at three places in the model, all visible in the §7 diagram: before Attention, before Feed-forward, and once more between the transformer block’s output <code class="language-plaintext highlighter-rouge">Y</code> and the output projection.</p>

<p>(Implementation: <code class="language-plaintext highlighter-rouge">Layernorm.forward</code> and <code class="language-plaintext highlighter-rouge">Layernorm.backward</code>, with backward derived from the standard chain rule.)</p>

<h1>9. Attention</h1>

<p>Attention is what makes a transformer a transformer!</p>

<h2>9.1 The mechanism</h2>

<p>You start with <code class="language-plaintext highlighter-rouge">X</code>, the 8 × 32 matrix from the output of LayerNorm. You compute three linear projections of it:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Q = X @ Wq        (8 × 32)
K = X @ Wk        (8 × 32)
V = X @ Wv        (8 × 32)
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">Wq</code>, <code class="language-plaintext highlighter-rouge">Wk</code>, <code class="language-plaintext highlighter-rouge">Wv</code> are 32 × 32 learned matrices. <code class="language-plaintext highlighter-rouge">Q</code>, <code class="language-plaintext highlighter-rouge">K</code>, <code class="language-plaintext highlighter-rouge">V</code> are three different views of the same input. The names come from a database analogy:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">Q[i]</code> is the <strong>query</strong> at position <code class="language-plaintext highlighter-rouge">i</code>. Think of it as “what is position <code class="language-plaintext highlighter-rouge">i</code> looking for?”</li>
  <li><code class="language-plaintext highlighter-rouge">K[j]</code> is the <strong>key</strong> at position <code class="language-plaintext highlighter-rouge">j</code>. Think of it as “what does position <code class="language-plaintext highlighter-rouge">j</code> advertise?”</li>
  <li><code class="language-plaintext highlighter-rouge">V[j]</code> is the <strong>value</strong> at position <code class="language-plaintext highlighter-rouge">j</code>. Think of it as “what would position <code class="language-plaintext highlighter-rouge">j</code> contribute if attended to?”</li>
</ul>

<p>We compute a scalar score for every (query position, key position) pair:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>scores[i, j] = Q[i] · K[j] / sqrt(d_k)        (8 × 8)
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">d_k</code> is the length of the vectors. From the shapes above, <code class="language-plaintext highlighter-rouge">Q[i]</code> and <code class="language-plaintext highlighter-rouge">K[j]</code> both have 32 elements, so <code class="language-plaintext highlighter-rouge">d_k = 32</code>.</p>

<p>Each <code class="language-plaintext highlighter-rouge">scores[i, j]</code> is a single number, the dot product of row <code class="language-plaintext highlighter-rouge">i</code> of <code class="language-plaintext highlighter-rouge">Q</code> with row <code class="language-plaintext highlighter-rouge">j</code> of <code class="language-plaintext highlighter-rouge">K</code>, then divided by <code class="language-plaintext highlighter-rouge">sqrt(d_k)</code>. Collecting them gives an 8 × 8 matrix <code class="language-plaintext highlighter-rouge">scores</code>, with one cell per pair of positions. Higher score means “row <code class="language-plaintext highlighter-rouge">i</code> cares more about row <code class="language-plaintext highlighter-rouge">j</code>”.</p>

<p>For any pair where <code class="language-plaintext highlighter-rouge">j &gt; i</code>, set the score to <code class="language-plaintext highlighter-rouge">-∞</code>, creating a causal mask. This prevents position <code class="language-plaintext highlighter-rouge">i</code> from looking at anything that comes after it, which is required for next-token prediction (otherwise the model could just look ahead at the answer).</p>

<p>Apply softmax to each row of the scores matrix to convert it into a probability distribution:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>probs[i, j] = exp(scores[i, j]) / Σ_k exp(scores[i, k])
</code></pre></div></div>

<p>After softmax, each row of <code class="language-plaintext highlighter-rouge">probs</code> sums to 1. Masked entries (where the score was <code class="language-plaintext highlighter-rouge">-∞</code>) become exactly 0. The rest are non-negative and form a probability distribution over the earlier-or-equal positions.</p>

<p>Finally, use those weights to take a weighted average of the values:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>out[i] = Σ_j probs[i, j] * V[j]              (8 × 32)
</code></pre></div></div>

<p>This Q/K/V/scores/softmax process results in output matrix where row <code class="language-plaintext highlighter-rouge">i</code> is the weighted average of the value rows from earlier positions, where those weights come from the query-key dot products.</p>

<p>This allows position <code class="language-plaintext highlighter-rouge">i</code> to pull information from any earlier position by assigning it a high probability.</p>

<h2>9.2 Multiple heads</h2>

<p>A head is one independent run of the whole §9.1 mechanism (Q/K/V projections, scores, softmax, weighted sum of V). Multi-head attention splits the matrix into multiple slices and operates on them in parallel. This allows the model to perform multiple types of routing simultaneously. §17 shows only one of our two heads actually doing work, while the other remains diffuse. We could probably train this model with <code class="language-plaintext highlighter-rouge">n_heads = 1</code> and it would still converge, but 2 is the typical transformer pattern, and because §17 is more instructive with two heads to compare side by side.</p>

<p>In our model, the hyperparameter defines <code class="language-plaintext highlighter-rouge">n_heads = 2</code>, so we split the 32 columns of Q, K, V down the middle:</p>

<ul>
  <li>Head 0 uses columns 0..15 of Q, K, V (<code class="language-plaintext highlighter-rouge">head_dim = 16</code>).</li>
  <li>Head 1 uses columns 16..31.</li>
</ul>

<p>Each head dot-products and softmaxes only within its own 16-element slice, then takes its own weighted sum of V. So <code class="language-plaintext highlighter-rouge">d_k</code> from §9.1 is now 16 (the per-head slice width), which is why the formula uses <code class="language-plaintext highlighter-rouge">sqrt(16)</code> in practice.</p>

<p>Each head produces its own 8 × 16 output. The two are concatenated back to 8 × 32 (head 0’s columns on the left, head 1’s on the right):</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>concat(out_head_0, out_head_1)         (8 × 32)
</code></pre></div></div>

<p>But that’s not the final answer. Without one more step, head 0’s output would always live in columns 0..15 and head 1’s in columns 16..31; the two would never combine. So we apply a learned 32 × 32 matrix <code class="language-plaintext highlighter-rouge">Wo</code> that mixes the columns:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Y = concat(out_head_0, out_head_1) @ Wo        (8 × 32)
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">Wo</code> lets the model learn how to combine the heads’ outputs. It’s the fourth and last learned matrix in attention (alongside <code class="language-plaintext highlighter-rouge">Wq</code>, <code class="language-plaintext highlighter-rouge">Wk</code>, <code class="language-plaintext highlighter-rouge">Wv</code>).</p>

<p>The total parameter count for the attention block is 4 × (32 × 32) = 4096 floats (Wq, Wk, Wv, Wo).</p>

<p>(Implementation: <code class="language-plaintext highlighter-rouge">Attention.forward</code> and <code class="language-plaintext highlighter-rouge">Attention.backward</code> in <code class="language-plaintext highlighter-rouge">lib/attention.ml</code>. Per-head slicing is done by column-offset arithmetic into the same flat tensors, with no explicit reshape.)</p>

<h1>10. Feed-forward (FFN)</h1>

<p>The simpler of the two operations inside the transformer block is called the Feed-Forward Network or FFN where network is just a generic word for a stack of learnable layers (as in “neural network”).</p>

<p>For each row of the input matrix independently:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>hidden = X @ W1            (32  → 128)
hidden = GELU(hidden)
out    = hidden @ W2       (128 → 32)
</code></pre></div></div>

<ul>
  <li><code class="language-plaintext highlighter-rouge">W1</code> is a learned 32 x 128 matrix, and <code class="language-plaintext highlighter-rouge">W2</code> is a learned 128 x 32 matrix in the same way as <code class="language-plaintext highlighter-rouge">Wq, Wk, Wv, Wo</code> from §9 are. <code class="language-plaintext highlighter-rouge">X @ W1</code> widens each 32-element row to 128 floats; <code class="language-plaintext highlighter-rouge">hidden @ W2</code> narrows it back to 32.</li>
  <li><code class="language-plaintext highlighter-rouge">GELU</code> is an element-wise non-linear function. Applied to a vector, it transforms each element independently. It passes large positive values through unchanged, squashes large negative values toward zero, and smoothly interpolates between. The “G” is for Gaussian as it is based on the Gaussian distribution.</li>
</ul>

<p>The non-linearity is the whole point of the Feed-forward block. Without something non-linear between <code class="language-plaintext highlighter-rouge">W1</code> and <code class="language-plaintext highlighter-rouge">W2</code>, the full computation <code class="language-plaintext highlighter-rouge">(X @ W1) @ W2</code> would collapse to <code class="language-plaintext highlighter-rouge">X @ (W1 @ W2)</code> as just one bigger  matrix multiplication. GELU between them is what stops that collapse and gives this block expressive power that attention’s linear projections alone don’t have.</p>

<p>The 128 dimension comes from the hyperparameter table and is <code class="language-plaintext highlighter-rouge">d_ff = 4 × d_model</code>. This is the conventional ratio for transformer FFN blocks, and it could shrink or expand trading capacity relative to compute.</p>

<p>The Feed-forward parameter count is <code class="language-plaintext highlighter-rouge">32 × 128 + 128 × 32 = 8192</code> floats, which is bigger than the whole attention block and is typically the case in real transformers where the FFN parameters dominate.</p>

<p>(Implementation: <code class="language-plaintext highlighter-rouge">Ffn.forward</code> and <code class="language-plaintext highlighter-rouge">Ffn.backward</code> in <code class="language-plaintext highlighter-rouge">lib/ffn.ml</code>.)</p>

<h1>11. Loss</h1>

<p>§4–§10 described the forward pass, showing how the model turns input ids into 8 × 16 logits that represent the model’s prediction. But during training, we need a way to score that prediction, which is called the loss. The gradient descent step (§13) will adjust the model’s parameters to make the loss go down.</p>

<p>The standard choice of calculating the loss in a probability distribution over a discrete set is called softmax cross-entropy. For each scored position (positions 4
and 5, the ones flagged <code class="language-plaintext highlighter-rouge">mask[i] = true</code> in our setup, where <code class="language-plaintext highlighter-rouge">c1</code> and <code class="language-plaintext highlighter-rouge">c2</code> are the targets):</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>p[i]   = softmax(logits[i])              (a probability over 32 tokens)
loss_i = -log(p[i][target[i]])           (negative log-prob of the right token)
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">target[i]</code> is the correct token at position i, the one we want the model to predict. For the <code class="language-plaintext highlighter-rouge">8 + a = 12</code> example, <code class="language-plaintext highlighter-rouge">target[4] = 1</code> (the high digit) and <code class="language-plaintext highlighter-rouge">target[5] = 2</code> (the low digit). <code class="language-plaintext highlighter-rouge">p[i][target[i]]</code> is the probability the model assigned to that correct token.</p>

<p>Concretely, look back at the 8 × 32 matrix in §2.</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">p[4]</code> was <code class="language-plaintext highlighter-rouge">[·, 1.00, ·, ·, ...]</code>. The model put 1.00 on token 1 at position 4. Since <code class="language-plaintext highlighter-rouge">target[4] = 1</code>, that’s the correct token, so <code class="language-plaintext highlighter-rouge">p[4][target[4]] = 0.999</code> and <code class="language-plaintext highlighter-rouge">loss_4 = -log(0.999) ≈ 0.001</code>.</li>
  <li><code class="language-plaintext highlighter-rouge">p[5]</code> was <code class="language-plaintext highlighter-rouge">[·, ·, 0.99, ·, ...]</code>, 0.993 on token 2. Since <code class="language-plaintext highlighter-rouge">target[5] = 2</code>, that’s correct too, so <code class="language-plaintext highlighter-rouge">p[5][target[5]] = 0.993</code> and <code class="language-plaintext highlighter-rouge">loss_5 = -log(0.993) ≈ 0.007</code>.</li>
</ul>

<p>Total loss for this example is the mean of <code class="language-plaintext highlighter-rouge">loss_4</code> and <code class="language-plaintext highlighter-rouge">loss_5</code>: <code class="language-plaintext highlighter-rouge">(0.001 + 0.007) / 2 ≈ 0.004</code>. (The total loss across the dataset is the mean of <code class="language-plaintext highlighter-rouge">loss_i</code> over all scored positions across all examples in the batch.)</p>

<p>A few reference points for calibrating the loss number:</p>

<ul>
  <li>If the model assigns 100% probability to the correct token, <code class="language-plaintext highlighter-rouge">loss = 0</code>.</li>
  <li>If 50%, <code class="language-plaintext highlighter-rouge">loss ≈ 0.69</code>.</li>
  <li>If 1%, <code class="language-plaintext highlighter-rouge">loss ≈ 4.6</code>.</li>
  <li>A uniform 1/32 prediction gives <code class="language-plaintext highlighter-rouge">loss = log(32) ≈ 3.47</code>.</li>
</ul>

<p>The gradient of the loss with respect to the logits has a closed form that doesn’t need the <code class="language-plaintext highlighter-rouge">log</code> numerically:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>d_logits[i] = (p[i] - one_hot(target[i])) / num_scored    (if mask[i])
            = 0                                            (otherwise)
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">one_hot(j)</code> is a 32-element vector with a 1 at index j and 0 everywhere else. For our example, <code class="language-plaintext highlighter-rouge">target[5] = 2</code> (the digit “2” is the right answer at position 5), so <code class="language-plaintext highlighter-rouge">one_hot(target[5]) = one_hot(2) = [0, 0, 1, 0, 0, ..., 0]</code>. The marker <code class="language-plaintext highlighter-rouge">1</code> lands at index 2 because that is where <code class="language-plaintext highlighter-rouge">target</code> points.</p>

<p>Subtracting <code class="language-plaintext highlighter-rouge">one_hot(target[i])</code> from <code class="language-plaintext highlighter-rouge">p[i]</code> decreases the probability at the correct token by 1 and leaves the others unchanged. The gradient of <code class="language-plaintext highlighter-rouge">d_logits[i]</code> is therefore positive everywhere except at the right token (where it’s negative), which pushes the model to increase the probability at the right token and decrease it at every other token, exactly what we want.</p>

<p>This is the starting point of the backward pass.</p>

<p>(Implementation: <code class="language-plaintext highlighter-rouge">Loss.forward_and_grad</code> in <code class="language-plaintext highlighter-rouge">lib/loss.ml</code>.)</p>

<h1>12. Backward pass</h1>

<p>We have the loss (§11) as a single floating-point number summarising how wrong the model is. Gradient descent (§13) will update each of the 13,760 parameters in the direction that decreases the loss. To do that, for every single parameter it needs, a number saying “if you nudge me up by ε, the loss changes by <code class="language-plaintext highlighter-rouge">gradient × ε</code>”. That number is the <strong>gradient of the loss with respect to that parameter</strong>. We need 13,760 of them, one per parameter. Computing all of them is the <em>backward pass</em> (or <em>backprop</em>).</p>

<p>The loss depends on the logits; the logits depend on <code class="language-plaintext highlighter-rouge">Y'</code>; <code class="language-plaintext highlighter-rouge">Y'</code> depends on <code class="language-plaintext highlighter-rouge">Y</code>; <code class="language-plaintext highlighter-rouge">Y</code> depends on the block’s parameters and on <code class="language-plaintext highlighter-rouge">X</code>; <code class="language-plaintext highlighter-rouge">X</code> depends on the embedding tables and on the input ids. Each depends-on is a function we already wrote in the forward pass. The chain rule says: differentiate one step at a time from the loss backward function, multiplying derivatives as we go. Apply it to every operation in the forward pass, in reverse order, and we end up with a gradient for every parameter.</p>

<p>Every block in this model has a <code class="language-plaintext highlighter-rouge">backward</code> function that does the chain rule, with shapes and indices written out. Verbose, but it makes the math visible. And <code class="language-plaintext highlighter-rouge">test/grad_check.ml</code> verifies every step numerically against finite-difference perturbations of the inputs (so a sign error or off-by-one anywhere fails a test).</p>

<p>Each block’s <code class="language-plaintext highlighter-rouge">backward</code> function takes one argument and produces one result. The input is <code class="language-plaintext highlighter-rouge">d_out</code>, the gradient of the loss with respect to the block’s output. It was computed by whatever ran after this block (the next block’s backward function, or the loss directly for the last block). The output is <code class="language-plaintext highlighter-rouge">d_in</code>, the gradient of the loss with respect to the block’s input. It is passed to whatever ran <em>before</em> this block. It also accumulates gradients into the block’s parameter tensors (e.g. <code class="language-plaintext highlighter-rouge">dWq</code> for the attention’s <code class="language-plaintext highlighter-rouge">Wq</code>).</p>

<p>Each forward pass caches whatever it will need for its backward pass. LayerNorm caches the per-row mean and <code class="language-plaintext highlighter-rouge">1/std</code>. Attention caches Q, K, V, and the post-softmax probabilities. FFN caches the pre-GELU activations. The backward function then runs the chain rule using those cached values, never re-running forward.</p>

<p>The full reverse trip:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>   d_logits  (from softmax-CE in lib/loss.ml)
       │
       │  Output projection:  d_y_norm = d_logits @ E
       │                dE      += d_logits^T @ Y'
       ▼
   d_y_norm
       │  Final LN backward
       ▼
   d_y                        (gradient at block output)
       │  Block backward (chain through both residuals)
       ▼
   d_x                        (gradient at block input)
       │  Embedding backward
       ▼
   ─── done ───
   every weight matrix has a gradient
</code></pre></div></div>

<p>A residual connection produces two gradient paths back to its input. One goes through the sublayer, the other through the identity. Both contribute, and they sum at the input. The block backward in <code class="language-plaintext highlighter-rouge">lib/transformer_block.ml</code> makes this explicit.</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nn">Ffn</span><span class="p">.</span><span class="n">backward</span> <span class="n">t</span><span class="o">.</span><span class="n">ffn</span> <span class="o">~</span><span class="n">d_out</span> <span class="o">~</span><span class="n">d_in</span><span class="o">:</span><span class="n">d_ln2_out</span><span class="p">;</span>
<span class="nn">Layernorm</span><span class="p">.</span><span class="n">backward</span> <span class="n">t</span><span class="o">.</span><span class="n">ln2</span> <span class="o">~</span><span class="n">d_out</span><span class="o">:</span><span class="n">d_ln2_out</span> <span class="o">~</span><span class="n">d_in</span><span class="o">:</span><span class="n">d_x_mid</span><span class="p">;</span>
<span class="nn">Ops</span><span class="p">.</span><span class="n">add_inplace</span> <span class="o">~</span><span class="n">dst</span><span class="o">:</span><span class="n">d_x_mid</span> <span class="o">~</span><span class="n">src</span><span class="o">:</span><span class="n">d_out</span><span class="p">;</span>       <span class="c">(* residual: dX' += d_out *)</span>

<span class="nn">Attention</span><span class="p">.</span><span class="n">backward</span> <span class="n">t</span><span class="o">.</span><span class="n">attn</span> <span class="o">~</span><span class="n">d_out</span><span class="o">:</span><span class="n">d_x_mid</span> <span class="o">~</span><span class="n">d_in</span><span class="o">:</span><span class="n">d_ln1_out</span><span class="p">;</span>
<span class="nn">Layernorm</span><span class="p">.</span><span class="n">backward</span> <span class="n">t</span><span class="o">.</span><span class="n">ln1</span> <span class="o">~</span><span class="n">d_out</span><span class="o">:</span><span class="n">d_ln1_out</span> <span class="o">~</span><span class="n">d_in</span><span class="p">;</span>
<span class="nn">Ops</span><span class="p">.</span><span class="n">add_inplace</span> <span class="o">~</span><span class="n">dst</span><span class="o">:</span><span class="n">d_in</span> <span class="o">~</span><span class="n">src</span><span class="o">:</span><span class="n">d_x_mid</span><span class="p">;</span>        <span class="c">(* residual: dX += d_x_mid *)</span>
</code></pre></div></div>

<h1>13. Optimiser</h1>

<p>“Every parameter” means every individual float, all 13,760 of them. Each cell of every learned matrix (<code class="language-plaintext highlighter-rouge">E</code>, <code class="language-plaintext highlighter-rouge">P</code>, <code class="language-plaintext highlighter-rouge">Wq</code>, <code class="language-plaintext highlighter-rouge">Wk</code>, <code class="language-plaintext highlighter-rouge">Wv</code>, <code class="language-plaintext highlighter-rouge">Wo</code>, <code class="language-plaintext highlighter-rouge">W1</code>, <code class="language-plaintext highlighter-rouge">W2</code>, plus the LayerNorm <code class="language-plaintext highlighter-rouge">γ</code>s and <code class="language-plaintext highlighter-rouge">β</code>s) gets its own gradient and its own update. The formulas below all apply element-wise, scalar by scalar.</p>

<p>The simplest update rule is</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>param := param - lr * grad
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">lr</code> is a small positive number (around 0.001 in our setup) called the learning rate that controls how big each step is. Too small and training crawls; too big and the loss bounces around or diverges instead of settling. The minus sign moves the parameter against the gradient, since the gradient points up the loss surface and we want to go down.</p>

<p>That formula is plain gradient descent. In practice, transformers train better with AdamW, which keeps a per-parameter running average of the gradient and its square:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>m   := 0.9   * m + 0.1   * grad
v   := 0.999 * v + 0.001 * grad * grad
m_hat := m / (1 - 0.9^step)
v_hat := v / (1 - 0.999^step)
param := param - lr * (m_hat / (sqrt(v_hat) + 1e-8) + 0.01 * param)
</code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">m_hat / sqrt(v_hat)</code> term normalises by gradient magnitude, so different parameters can move at appropriate rates without manual tuning. The final <code class="language-plaintext highlighter-rouge">+ 0.01 * param</code> is weight decay, a gentle pull toward zero that prevents weights from drifting unbounded.</p>

<p>(Implementation: <code class="language-plaintext highlighter-rouge">Adam.step</code> in <code class="language-plaintext highlighter-rouge">lib/adam.ml</code>.)</p>

<h1>14. Training loop</h1>

<p>In the training loop, a step is one parameter update, one application of the AdamW rule from §13. Over the entire run, we take 5000 steps. A batch is the group of training examples processed together within a single step. Their gradients are averaged before AdamW runs. We use <code class="language-plaintext highlighter-rouge">batch_size = 16</code>, so each step processes 16 examples. (A single example would give a noisy gradient estimate; averaging 16 smooths it out.)</p>

<p><code class="language-plaintext highlighter-rouge">lib/train.ml</code>. For each step:</p>

<ol>
  <li>Zero all parameter gradients.</li>
  <li>For each of the 16 examples in the batch:
    <ul>
      <li>Run forward → logits.</li>
      <li>Compute loss and <code class="language-plaintext highlighter-rouge">d_logits</code>.</li>
      <li>Run backward → gradients added into every parameter’s <code class="language-plaintext highlighter-rouge">grad</code> tensor.</li>
    </ul>
  </li>
  <li>Divide every gradient tensor by <code class="language-plaintext highlighter-rouge">batch_size</code> (= 16), so we have the mean per-example gradient instead of the sum.</li>
  <li>Apply the AdamW update rule (§13) to every parameter, using those averaged gradients.</li>
</ol>

<p>The <code class="language-plaintext highlighter-rouge">lr</code> (from §13) doesn’t stay at 0.001 the whole way through. For the first 50 steps, it ramps linearly from 0 up to 0.001, a learning-rate warmup. AdamW’s running averages <code class="language-plaintext highlighter-rouge">m</code>, <code class="language-plaintext highlighter-rouge">v</code> need a few steps to stabilise; taking large early steps before they have meaningful values can throw the model into a bad region. Warmup avoids that.</p>

<p>Every 250 steps, the trainer evaluates all 256 examples and prints the loss and accuracy. (5000 steps × 16 examples per step = 80,000 forward and backward passes during the whole run, which takes about 50 seconds.)</p>

<h1>15. Training vs inference</h1>

<p>Training and inference run almost the same code: both call <code class="language-plaintext highlighter-rouge">Model.forward</code>, which produces logits from <code class="language-plaintext highlighter-rouge">ids</code>. The difference is what else we keep around.</p>

<p>During training we allocate three things in addition to the parameters:</p>

<table>
  <thead>
    <tr>
      <th>What</th>
      <th>Why</th>
      <th>Size</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>AdamW state <code class="language-plaintext highlighter-rouge">m</code>, <code class="language-plaintext highlighter-rouge">v</code> per parameter</td>
      <td>running averages for the optimizer</td>
      <td>2 × 13,760 floats</td>
    </tr>
    <tr>
      <td>Gradient buffer per parameter</td>
      <td>accumulator for backward</td>
      <td>1 × 13,760 floats</td>
    </tr>
    <tr>
      <td>Forward activation caches per block</td>
      <td>inputs needed by backward</td>
      <td>a few KB</td>
    </tr>
  </tbody>
</table>

<p>So the per-step memory footprint is roughly 4 × the parameter count, plus caches. For 13,760 parameters, that’s tiny, but at the scale of a real LLM (billions of parameters), this 4× overhead is the dominant constraint on what hardware you need to train on. It’s why you can deploy a 7B model on a laptop, but you can’t train one there.</p>

<p>For inference, we need only the trained parameter values:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>E                 32 × 32 = 1,024 floats   (used twice: input embedding + output projection)
P                  8 × 32 =   256 floats
Wq, Wk, Wv, Wo    32 × 32 = 1,024 each, 4,096 total
W1               32 × 128 = 4,096 floats
W2               128 × 32 = 4,096 floats
γ, β × 3       32 × 2 × 3 =   192 floats
                          ─────────
total                      13,760 floats   ≈ 110 KB on disk at 8 bytes/float
</code></pre></div></div>

<p>To use the trained model:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>1. encode the question:  ids = [BOS; 8; +; a; =; PAD; PAD; PAD]
2. logits = Model.forward model ~ids       (same code as training)
3. answer_c1 = argmax of logits row 4
4. answer_c2 = argmax of logits row 5
</code></pre></div></div>

<p>The gradients aren’t computed, the optimiser isn’t called, and the caches aren’t needed. The model itself is the same code; we just stop calling <code class="language-plaintext highlighter-rouge">Model.backward</code>.</p>

<p>In this project we don’t actually save and reload the trained parameters to disk; we train and infer in the same process. Doing so would be a few lines: <code class="language-plaintext highlighter-rouge">Marshal</code> the <code class="language-plaintext highlighter-rouge">Model.t</code> after training, restore it later.</p>

<h1>16. A real training run</h1>

<p>Output from <code class="language-plaintext highlighter-rouge">dune exec bin/main.exe</code>:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>attn-ox: d_model=32 heads=2 d_ff=128  seq=8 vocab=32  batch=16 lr=0.001  params=13760
step  250  loss=0.9444  digit_acc=0.678  ex_acc=0.355
step  500  loss=0.6424  digit_acc=0.803  ex_acc=0.605
step  750  loss=0.4394  digit_acc=0.850  ex_acc=0.699
step 1000  loss=0.4011  digit_acc=0.883  ex_acc=0.766
step 1250  loss=0.2981  digit_acc=0.979  ex_acc=0.957
step 1500  loss=0.2204  digit_acc=0.965  ex_acc=0.930
step 1750  loss=0.1533  digit_acc=1.000  ex_acc=1.000
step 2000  loss=0.0981  digit_acc=0.990  ex_acc=0.980
step 2250  loss=0.0675  digit_acc=1.000  ex_acc=1.000
step 2500  loss=0.0504  digit_acc=1.000  ex_acc=1.000
step 2750  loss=0.0630  digit_acc=1.000  ex_acc=1.000
step 3000  loss=0.0244  digit_acc=1.000  ex_acc=1.000
step 3250  loss=0.0195  digit_acc=1.000  ex_acc=1.000
step 3500  loss=1.0294  digit_acc=0.867  ex_acc=0.750
step 3750  loss=0.0219  digit_acc=1.000  ex_acc=1.000
step 4000  loss=0.0125  digit_acc=1.000  ex_acc=1.000
step 4500  loss=0.0083  digit_acc=1.000  ex_acc=1.000
step 5000  loss=0.0056  digit_acc=1.000  ex_acc=1.000

final: digit_acc=1.000  ex_acc=1.000
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">digit_acc</code> is the fraction of answer digits the model gets right (512 total, two per example, 256 examples). <code class="language-plaintext highlighter-rouge">ex_acc</code> is the fraction of examples where <em>both</em> digits are right. The model first hits 100% example accuracy at step 1750 and converges to it stably by ~step 3000. (One brief regression at step 3500 from gradient noise, but it recovers.)</p>

<p>Initial loss is around 1.5, well below <code class="language-plaintext highlighter-rouge">log(32) ≈ 3.47</code> (random over a 32-token vocabulary) because the warmup completes in 50 steps and the model has already started fitting before the first 250-step report. It drops to 0.006 by the end.</p>

<p>A few sample predictions on random examples after training:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>sample predictions:
  9 + 2 = 0b  (truth 0b)  OK
  3 + b = 0e  (truth 0e)  OK     ← b is the hex digit for 11
  3 + 9 = 0c  (truth 0c)  OK
  1 + 7 = 08  (truth 08)  OK
  9 + 9 = 12  (truth 12)  OK     ← carry (9+9=18, hex 12)
  d + 9 = 16  (truth 16)  OK     ← carry
  b + b = 16  (truth 16)  OK     ← carry
  8 + f = 17  (truth 17)  OK     ← carry
  8 + a = 12  (truth 12)  OK     ← the example we'll inspect in §17
</code></pre></div></div>

<h1>17. What the model learned</h1>

<p>Here is the model’s behaviour on <code class="language-plaintext highlighter-rouge">8 + a</code> after training, captured by <code class="language-plaintext highlighter-rouge">Inspect.dump_example</code>. Recall that <code class="language-plaintext highlighter-rouge">a</code> is the hex digit for 10, so <code class="language-plaintext highlighter-rouge">8 + a = 18</code> (decimal), which is <code class="language-plaintext highlighter-rouge">12</code> in hex.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>────────── inspecting 8 + a ──────────
input ids  : BOS 8 + a = 1 2 PAD
positions  : BOS  8  +  a  =  c1 c2 PAD

top-3 predictions at the answer positions:
  pos 4 (=, target=1):    1=0.999   2=0.001   9=0.000
  pos 5 (c1, target=2):   2=0.993   3=0.005   1=0.002
</code></pre></div></div>

<p>The model is confident: 99.9% probability on the correct first answer digit, 99.3% on the second. (These are the top three entries of rows 4 and 5 of the full 8 × 32 probability distribution shown in §2; the “top-3 predictions” summary just hides the small values.)</p>

<p>Now look at the attention probabilities. Each row is a probability distribution: row <code class="language-plaintext highlighter-rouge">i</code> shows where position <code class="language-plaintext highlighter-rouge">i</code> reaches when computing its output. The causal mask blanks the upper triangle.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>  head 0  (rows attend to cols, causal mask hides upper triangle)
             BOS     8     +     a     =    c1    c2   PAD
      BOS  1.000     ·     ·     ·     ·     ·     ·     ·
        8  0.706 0.294     ·     ·     ·     ·     ·     ·
        +  0.274 0.458 0.268     ·     ·     ·     ·     ·
        a  0.368 0.156 0.358 0.119     ·     ·     ·     ·
        =  0.068 0.259 0.054 0.524 0.095     ·     ·     ·
       c1  0.048 0.192 0.035 0.278 0.050 0.398     ·     ·
       c2  0.123 0.195 0.097 0.156 0.105 0.166 0.158     ·
      PAD  0.112 0.097 0.122 0.106 0.130 0.161 0.148 0.123

  head 1  (rows attend to cols, causal mask hides upper triangle)
             BOS     8     +     a     =    c1    c2   PAD
      BOS  1.000     ·     ·     ·     ·     ·     ·     ·
        8  0.801 0.199     ·     ·     ·     ·     ·     ·
        +  0.316 0.369 0.315     ·     ·     ·     ·     ·
        a  0.346 0.050 0.535 0.069     ·     ·     ·     ·
        =  0.095 0.347 0.076 0.378 0.105     ·     ·     ·
       c1  0.056 0.497 0.033 0.265 0.038 0.111     ·     ·
       c2  0.138 0.111 0.175 0.162 0.171 0.077 0.165     ·
      PAD  0.125 0.039 0.166 0.072 0.194 0.222 0.080 0.102
</code></pre></div></div>

<p>Two rows of head 1 are doing the work. They are rows 4 and 5, the rows the loss actually scores.</p>

<p>Head 1, row <code class="language-plaintext highlighter-rouge">=</code> (position 4). Reading along that row in the matrix: 0.347 on column <code class="language-plaintext highlighter-rouge">8</code> (position 1, where <code class="language-plaintext highlighter-rouge">x</code> lives) and 0.378 on column <code class="language-plaintext highlighter-rouge">a</code> (position 3, where <code class="language-plaintext highlighter-rouge">y</code> lives). When the model is sitting at the <code class="language-plaintext highlighter-rouge">=</code> token, head 1 reaches back and pulls in both operands, roughly equally, because computing the first answer digit requires knowing whether <code class="language-plaintext highlighter-rouge">x + y ≥ 16</code> (which depends on both).</p>

<p>That weighting is the §9.1 mechanism running for this head: it is <code class="language-plaintext highlighter-rouge">softmax(Q[4] · K[j] / sqrt(16))</code> evaluated at all <code class="language-plaintext highlighter-rouge">j</code>. Through training, the query <code class="language-plaintext highlighter-rouge">Q[4]</code> has learned to match the keys at both operand positions, not at any other positions. After softmax, mass concentrates on positions 1 and 3.</p>

<p>Head 1’s output at row <code class="language-plaintext highlighter-rouge">=</code> is <code class="language-plaintext highlighter-rouge">Σ_j probs[4][j] * V[j]</code>, a weighted sum of value vectors. Most of the weight goes on the value vectors at the two operand positions <code class="language-plaintext highlighter-rouge">V[1]</code> and <code class="language-plaintext highlighter-rouge">V[3]</code>. These two vectors are not the literal digits <code class="language-plaintext highlighter-rouge">8</code> and <code class="language-plaintext highlighter-rouge">a</code>; they are learned 16-element vectors that the model has put there during training to encode whatever it needs about each operand. Head 1’s combined output is roughly a 50/50 mix of <code class="language-plaintext highlighter-rouge">V[1]</code> and <code class="language-plaintext highlighter-rouge">V[3]</code>. This vector gets added into the central 32-element state at position 4 via the attention residual <code class="language-plaintext highlighter-rouge">+</code> (§7’s diagram). That central running state, the vertical column in the §7 diagram (the value of <code class="language-plaintext highlighter-rouge">X</code>, then <code class="language-plaintext highlighter-rouge">X'</code>, then <code class="language-plaintext highlighter-rouge">Y</code> at each position), is typically called the residual stream. Each sub-block adds its output to it rather than replacing it. From here, the residual stream at position 4 is processed by the Feed-forward block (which adds another contribution), then the final LayerNorm, then the output projection. Only at the end does it become the prediction <code class="language-plaintext highlighter-rouge">1</code> (the high digit of 12). Head 1’s job is <em>routing</em>: matching <code class="language-plaintext highlighter-rouge">Q</code> against <code class="language-plaintext highlighter-rouge">K</code>
to choose where to read <code class="language-plaintext highlighter-rouge">V</code> from. Producing the answer digit is the rest of the pipeline’s job.</p>

<p>The pattern is position-based so for any input this row of head 1 attends to positions 1 and 3 (wherever the two operands sit). The columns happen to be labelled <code class="language-plaintext highlighter-rouge">8</code> and <code class="language-plaintext highlighter-rouge">a</code> only because in this example the operands have those values.</p>

<p>Head 1, row <code class="language-plaintext highlighter-rouge">c1</code> (position 5). 0.497 on column <code class="language-plaintext highlighter-rouge">8</code> (position 1) plus 0.265 on column <code class="language-plaintext highlighter-rouge">a</code> (position 3): again, both operands. Plus 0.111 on column <code class="language-plaintext highlighter-rouge">c1</code> (position 5, the row’s own position). Position 5 already has information about both operands mixed in from the previous step, so the residual stream at <code class="language-plaintext highlighter-rouge">c1</code> ends up holding both <code class="language-plaintext highlighter-rouge">x</code> and <code class="language-plaintext highlighter-rouge">y</code>. That’s exactly what the Feed-forward block needs to compute the second answer digit.</p>

<p>Head 0 has a similar pattern but slightly more diffuse: 0.259 on column <code class="language-plaintext highlighter-rouge">8</code> and 0.524 on column <code class="language-plaintext highlighter-rouge">a</code> at row <code class="language-plaintext highlighter-rouge">=</code>. With only two heads, both heads do meaningful work here; the model evidently needs both sources of information at every answer position, since computing the carry from <code class="language-plaintext highlighter-rouge">x + y</code> always requires both inputs.</p>

<h1>18. Did the model learn addition?</h1>

<p>§16 reports 100% accuracy. But it’s measured on the same 256 examples the model was trained on. I became concerned whether the model had learned the rule <code class="language-plaintext highlighter-rouge">x + y → digits</code> or has merely memorised the 256 input/output pairs which it was provided during training.</p>

<p>Rather than train on the entire dataset, only train on a random subset and then evaluate on the remainder. A rule-learner would be expected to get the unseen cases correct, but if the model simply memorised the results, it would fail on new data.</p>

<p>Furthermore, with <code class="language-plaintext highlighter-rouge">d_model = 32</code>, there are roughly 54 parameters per training example, making memorisation almost the logical approach. With <code class="language-plaintext highlighter-rouge">d_model = 4</code>, there would only be 376 parameters in total, which is just above the 256 threshold, thereby requiring a rule-based solution.</p>

<p><code class="language-plaintext highlighter-rouge">bin/long_train.exe</code> uses <code class="language-plaintext highlighter-rouge">d_model = 4</code> (376 parameters, with the same architecture, just narrower vectors), trained on 230 hex examples (a random 90% of the 256), and evaluated on the 26 remaining examples it has never seen, here’s what happens over training:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>d_model=4, 376 params, 230 train, 26 held-out:
  step    train_ex_acc   held_ex_acc
  1000      0.270          0.115
  2000      0.739          0.615
  3000      0.865          0.692
  4000      0.943          0.885
  5000      0.974          1.000   ← held-out hits 100% BEFORE training does
  6000      0.991          1.000
  7000      0.996          0.962   ← brief regression
  8000      1.000          1.000   ← full convergence on both
  ...
  50000     1.000          1.000   (stays at 100/100 for the rest of the run)
</code></pre></div></div>

<p>Held-out accuracy goes from 0.12 at step 1000 to 1.000 at step 5000 and stays there. The 26 held-out (x, y) pairs the model never saw during training all get correct answers.</p>

<p>There’s a giveaway in the trajectory: held-out hits 100% before training does. At step 5000 the model is at 0.974 on training (still missing some) but already 1.000 on held-out. The only way that can happen is if the model has found a generalising solution, one that gets every input right, including ones it has never been shown. There is no possible way to “memorise the held-out cases without seeing them”; the only way to get them right is to have extracted the rule.</p>

<p>This phenomenon is called <strong>grokking</strong> (Power et al., 2022).</p>

<p>What does the default <code class="language-plaintext highlighter-rouge">d_model = 32</code> do at 90/10? It’s more nuanced, and the contrast is informative. Across multiple random splits of the data (different <code class="language-plaintext highlighter-rouge">split_seed</code>), training to 50,000 steps:</p>

<table>
  <thead>
    <tr>
      <th><code class="language-plaintext highlighter-rouge">split_seed</code></th>
      <th>held-out trajectory</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>42</td>
      <td>groks at step 5000, stable at 100% for the rest</td>
    </tr>
    <tr>
      <td>7</td>
      <td>groks briefly at step 10000, drifts back to 0.92 by step 35000</td>
    </tr>
    <tr>
      <td>100</td>
      <td>never groks, plateau at 0.85–0.92</td>
    </tr>
    <tr>
      <td>999</td>
      <td>groks at step 20000, drifts back to 0.92 by step 50000</td>
    </tr>
  </tbody>
</table>

<p>So with <code class="language-plaintext highlighter-rouge">d_model = 32</code>, grokking is possible but unstable. Some seeds find the rule and stay; others find it briefly and drift back to memorisation-shaped solutions; others never find it at all. With <code class="language-plaintext highlighter-rouge">d_model = 4</code>, in the same setup, grokking happens reliably and stays. Once the small model has found the rule, it stays there for the rest of the training.</p>

<p>The reason is the loss landscape. At <code class="language-plaintext highlighter-rouge">d_model = 32</code> (about 54 parameters per training example), there’s room to memorise and room to encode the rule. Both are valid solutions.</p>

<p>At <code class="language-plaintext highlighter-rouge">d_model = 4</code> (about 1.6 parameters per example), there’s no memorising solution that fits the training set. The only minimum that drives the loss to zero is the rule-extracting one. Gradient descent finds it and there’s nowhere else to go.</p>

<p>A sweep over training fractions confirms the picture. At <code class="language-plaintext highlighter-rouge">d_model = 4</code>, with 50,000 training steps and varying how much of the 256-example dataset is held out:</p>

<table>
  <thead>
    <tr>
      <th>Train</th>
      <th>Held-out</th>
      <th>Train acc</th>
      <th>Held-out acc</th>
      <th>Verdict</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>128</td>
      <td>128</td>
      <td>1.000</td>
      <td>0.953</td>
      <td>memorises, plateaus</td>
    </tr>
    <tr>
      <td>153</td>
      <td>103</td>
      <td>1.000</td>
      <td>0.981</td>
      <td>almost groks</td>
    </tr>
    <tr>
      <td>179</td>
      <td>77</td>
      <td>1.000</td>
      <td>1.000</td>
      <td>groks fully</td>
    </tr>
    <tr>
      <td>204</td>
      <td>52</td>
      <td>1.000</td>
      <td>1.000</td>
      <td>groks fully</td>
    </tr>
    <tr>
      <td>230</td>
      <td>26</td>
      <td>1.000</td>
      <td>1.000</td>
      <td>groks fully</td>
    </tr>
  </tbody>
</table>

<p>So with <code class="language-plaintext highlighter-rouge">d_model = 4</code>, 70% of the 256 examples is the minimum training fraction at which the model groks fully. Below that, even 50,000 steps isn’t enough; held-out accuracy plateaus.</p>

<p>The threshold makes sense: at 70% (179 training examples), the model has 376 parameters / 179 examples ≈ 2.1 params per example. Above ~2 params per example the easy memorisation path comes back, the model takes it, and held-out lags. Below, the model is forced into the rule.</p>

<p><code class="language-plaintext highlighter-rouge">d_model = 2</code> would give 140 parameters, well below the memorisation capacity for 256 examples, but neither the training nor the held-out accuracy got above random-guessing.</p>

<p>A 376-parameter transformer trained on 70% of the 256 hex addition examples reliably extracts the rule of addition and applies it to inputs it has never seen.</p>

<h1>19. SIMD optimisation</h1>

<p><code class="language-plaintext highlighter-rouge">lib/ops.ml</code> is the only module that uses OxCaml extensions. The matmul kernels run on AVX 256-bit vectors via <code class="language-plaintext highlighter-rouge">Ocaml_simd_avx.Float64x4</code> (four float64s per vector, fused multiply-add). The loop pattern is “reorder the loops so the inner one walks contiguous memory, then SIMD the inner loop”:</p>

<div class="language-ocaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">let</span> <span class="n">matmul</span> <span class="o">~</span><span class="n">a</span> <span class="o">~</span><span class="n">b</span> <span class="o">~</span><span class="n">c</span> <span class="o">=</span>
  <span class="o">...</span>
  <span class="nn">Array</span><span class="p">.</span><span class="n">fill</span> <span class="n">cd</span> <span class="mi">0</span> <span class="p">(</span><span class="n">m</span> <span class="o">*</span> <span class="n">n</span><span class="p">)</span> <span class="mi">0</span><span class="o">.</span><span class="mi">0</span><span class="p">;</span>
  <span class="k">let</span> <span class="n">nv</span> <span class="o">=</span> <span class="n">n</span> <span class="o">/</span> <span class="mi">4</span> <span class="o">*</span> <span class="mi">4</span> <span class="k">in</span>
  <span class="k">for</span> <span class="n">i</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">to</span> <span class="n">m</span> <span class="o">-</span> <span class="mi">1</span> <span class="k">do</span>
    <span class="k">let</span> <span class="n">c_off</span> <span class="o">=</span> <span class="n">i</span> <span class="o">*</span> <span class="n">n</span> <span class="k">in</span>
    <span class="k">for</span> <span class="n">p</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">to</span> <span class="n">k</span> <span class="o">-</span> <span class="mi">1</span> <span class="k">do</span>
      <span class="k">let</span> <span class="n">a_ip</span> <span class="o">=</span> <span class="n">ad</span><span class="o">.</span><span class="p">(</span><span class="n">i</span> <span class="o">*</span> <span class="n">k</span> <span class="o">+</span> <span class="n">p</span><span class="p">)</span> <span class="k">in</span>
      <span class="k">let</span> <span class="n">av</span> <span class="o">=</span> <span class="nn">Float64x4</span><span class="p">.</span><span class="n">set1</span> <span class="p">(</span><span class="n">f_of</span> <span class="n">a_ip</span><span class="p">)</span> <span class="k">in</span>    <span class="c">(* broadcast scalar *)</span>
      <span class="k">let</span> <span class="n">b_off</span> <span class="o">=</span> <span class="n">p</span> <span class="o">*</span> <span class="n">n</span> <span class="k">in</span>
      <span class="k">let</span> <span class="k">mutable</span> <span class="n">j</span> <span class="o">=</span> <span class="mi">0</span> <span class="k">in</span>                       <span class="c">(* let mutable, not ref *)</span>
      <span class="k">while</span> <span class="n">j</span> <span class="o">&lt;</span> <span class="n">nv</span> <span class="k">do</span>
        <span class="k">let</span> <span class="n">bv</span> <span class="o">=</span> <span class="nn">Float64x4</span><span class="p">.</span><span class="nn">Float_array</span><span class="p">.</span><span class="n">unsafe_get</span> <span class="n">bd</span> <span class="o">~</span><span class="n">idx</span><span class="o">:</span><span class="p">(</span><span class="n">b_off</span> <span class="o">+</span> <span class="n">j</span><span class="p">)</span> <span class="k">in</span>
        <span class="k">let</span> <span class="n">cv</span> <span class="o">=</span> <span class="nn">Float64x4</span><span class="p">.</span><span class="nn">Float_array</span><span class="p">.</span><span class="n">unsafe_get</span> <span class="n">cd</span> <span class="o">~</span><span class="n">idx</span><span class="o">:</span><span class="p">(</span><span class="n">c_off</span> <span class="o">+</span> <span class="n">j</span><span class="p">)</span> <span class="k">in</span>
        <span class="nn">Float64x4</span><span class="p">.</span><span class="nn">Float_array</span><span class="p">.</span><span class="n">unsafe_set</span> <span class="n">cd</span> <span class="o">~</span><span class="n">idx</span><span class="o">:</span><span class="p">(</span><span class="n">c_off</span> <span class="o">+</span> <span class="n">j</span><span class="p">)</span>
          <span class="p">(</span><span class="nn">Float64x4</span><span class="p">.</span><span class="n">mul_add</span> <span class="n">av</span> <span class="n">bv</span> <span class="n">cv</span><span class="p">);</span>          <span class="c">(* one FMA: cv + av*bv *)</span>
        <span class="n">j</span> <span class="o">&lt;-</span> <span class="n">j</span> <span class="o">+</span> <span class="mi">4</span>
      <span class="k">done</span><span class="p">;</span>
      <span class="k">while</span> <span class="n">j</span> <span class="o">&lt;</span> <span class="n">n</span> <span class="k">do</span>
        <span class="n">cd</span><span class="o">.</span><span class="p">(</span><span class="n">c_off</span> <span class="o">+</span> <span class="n">j</span><span class="p">)</span> <span class="o">&lt;-</span> <span class="n">cd</span><span class="o">.</span><span class="p">(</span><span class="n">c_off</span> <span class="o">+</span> <span class="n">j</span><span class="p">)</span> <span class="o">+.</span> <span class="p">(</span><span class="n">a_ip</span> <span class="o">*.</span> <span class="n">bd</span><span class="o">.</span><span class="p">(</span><span class="n">b_off</span> <span class="o">+</span> <span class="n">j</span><span class="p">));</span>
        <span class="n">j</span> <span class="o">&lt;-</span> <span class="n">j</span> <span class="o">+</span> <span class="mi">1</span>
      <span class="k">done</span>
    <span class="k">done</span>
  <span class="k">done</span>
</code></pre></div></div>

<p>The OxCaml-specific bits:</p>

<ul>
  <li><code class="language-plaintext highlighter-rouge">Float64x4</code> is an unboxed 256-bit vector type. Passed in YMM registers, no heap allocation.</li>
  <li><code class="language-plaintext highlighter-rouge">Float64x4.mul_add av bv cv</code> compiles to one <code class="language-plaintext highlighter-rouge">vfmadd</code> instruction computing <code class="language-plaintext highlighter-rouge">cv + av * bv</code> on 4 doubles in parallel.</li>
  <li><code class="language-plaintext highlighter-rouge">Float64x4.Float_array.unsafe_get arr ~idx:i</code> loads <code class="language-plaintext highlighter-rouge">arr[i..i+3]</code> into a vector register with no allocation.</li>
  <li><code class="language-plaintext highlighter-rouge">f_of</code> and <code class="language-plaintext highlighter-rouge">f_to</code> use the <code class="language-plaintext highlighter-rouge">%unbox_float</code> and <code class="language-plaintext highlighter-rouge">%box_float</code> intrinsics to convert between OCaml’s boxed <code class="language-plaintext highlighter-rouge">float</code> and the unboxed <code class="language-plaintext highlighter-rouge">float#</code>.</li>
  <li><code class="language-plaintext highlighter-rouge">let mutable j = 0</code> is OxCaml syntax for a mutable local that doesn’t allocate a <code class="language-plaintext highlighter-rouge">ref</code> cell.</li>
</ul>

<p>Bench results for the matmul shapes that appear in the model:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>=== matmul 32x32x32 ===     scalar 80.8 µs   SIMD 16.8 µs   4.8×
=== matmul 8x32x16 ===      scalar  9.7 µs   SIMD  2.4 µs   4.1×
=== matmul 8x32x128 ===     scalar 77.4 µs   SIMD 15.7 µs   4.9×
=== matmul 8x128x32 ===     scalar 87.9 µs   SIMD 15.1 µs   5.8×
</code></pre></div></div>

<p>Max numerical difference between scalar and SIMD: about <code class="language-plaintext highlighter-rouge">1e-17</code>. The grad-check tests pass for the SIMD version with the same tolerance as the scalar version (FMA gives more accurate results, not less, because it skips the intermediate rounding step).</p>

<p>End-to-end training time: 17.6s scalar -&gt; 10.3s SIMD (1.7× overall). The end-to-end speedup is smaller than the kernel speedup because softmax, per-head attention scoring, LayerNorm, AdamW, GELU, and the embeddingscatter are still scalar.</p>

<h1>20. Reproducing</h1>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>git clone https://github.com/mtelvers/attn-ox
<span class="nv">$ </span><span class="nb">cd</span> ~/attn-ox
<span class="nv">$ </span>opam <span class="nb">exec</span> <span class="nt">--switch</span> 5.2.0+ox <span class="nt">--</span> dune build
<span class="nv">$ </span>opam <span class="nb">exec</span> <span class="nt">--switch</span> 5.2.0+ox <span class="nt">--</span> dune runtest              <span class="c"># 12 tests, ~50ms</span>
<span class="nv">$ </span>opam <span class="nb">exec</span> <span class="nt">--switch</span> 5.2.0+ox <span class="nt">--</span> dune <span class="nb">exec </span>bin/main.exe    <span class="c"># train, ~10s</span>
<span class="nv">$ </span>opam <span class="nb">exec</span> <span class="nt">--switch</span> 5.2.0+ox <span class="nt">--</span> dune <span class="nb">exec </span>bench/bench.exe <span class="c"># SIMD bench</span>
</code></pre></div></div>

<p>The seed is fixed in <code class="language-plaintext highlighter-rouge">Train.run</code>, so the loss curve and the attention maps in §17 are reproducible.</p>
