---
title: Work distribution with OCluster
description: "We use OCluster to manage the build cluster for the CI services backing
  OCaml-CI and opam-repo-ci. However, it is a general-purpose tool and isn\u2019t
  tied to being a build system; it can distribute any jobs across multiple worker
  machines."
url: https://www.tunbury.org/2026/03/09/ocluster/
date: 2026-03-09T12:00:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>We use <a href="https://github.com/ocurrent/ocluster">OCluster</a> to manage the build cluster for the CI services backing OCaml-CI and opam-repo-ci. However, it is a general-purpose tool and isn’t tied to being a build system; it can distribute any jobs across multiple worker machines.</p>

<p>In my case, I need to generate some training data for Tessera by downloading some Sentinel-1 and Sentinel-2 satellite data and applying a tiny AI model for cloud masking. I need to generate about 5,000 tiles. With the HTTP round-trip, querying the STAC catalogue and downloading a single tile takes about 10 minutes. GNU Parallel showed that running more than four concurrently on my machine had no performance advantage. Enter OCluster.</p>

<p>The <a href="https://github.com/ocurrent/ocluster/blob/master/README.md">README.md</a> on the project homepage explains how to set up a cluster. In brief, you need one machine with a fixed and accessible IP address to be the scheduler, plus as many worker machines as you like. The workers make an outgoing connection to the scheduler so they can be behind NAT. Workers are grouped in pools based upon administrative boundaries, such as machine architecture. Run <code class="language-plaintext highlighter-rouge">ocluster-scheduler ... --capnp-listen-address=tcp:0.0.0.0:9000 --capnp-public-address=tcp:w.x.y.z:9000 --pools=foo,bar</code> which generates <code class="language-plaintext highlighter-rouge">pool-foo.cap</code> and <code class="language-plaintext highlighter-rouge">pool-bar.cap</code> which include the public IP address. Run a worker with <code class="language-plaintext highlighter-rouge">ocluster-worker --connect=pool-foo.cap --name worker-1 ...</code></p>

<p>For ease of local testing, I created a Dockerfile which builds my (OCaml) project from source and installs third-party libraries such as the ONNX runtime. I could submit the Dockerfile directly to the cluster, but the cache pruning is more sophisticated when using an OBuilder spec. OCluster runs <code class="language-plaintext highlighter-rouge">docker system prune</code> when the Docker partition is low on space, whereas OBuilder prunes individual layers on a least-recently-used basis.</p>

<p>An OBuilder spec is really just an s-expression version of the Dockerfile. For example, converting a trivial Dockerfile into <code class="language-plaintext highlighter-rouge">hello.spec</code></p>

<h2><code class="language-plaintext highlighter-rouge">Dockerfile.hello</code></h2>

<div class="language-dockerfile highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">FROM</span><span class="s"> debian:13</span>
<span class="k">USER</span><span class="s"> 1000:1000</span>
<span class="k">RUN </span><span class="nb">echo </span>Hello World
</code></pre></div></div>

<h2><code class="language-plaintext highlighter-rouge">hello.spec</code></h2>

<pre><code class="language-sexp">((from debian:13)
 (user (uid 1000) (gid 1000))
 (run (shell "echo Hello World"))
)
</code></pre>

<p>This can be submitted to OCluster using your capability.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>ocluster-client submit-obuilder <span class="nt">--connect</span> ~/mtelvers.cap <span class="nt">--pool</span> <span class="nb">test</span> <span class="nt">--local-file</span> ./hello.spec 
Tailing log:
Building on worker-1.ci.dev

<span class="o">(</span>from debian:13<span class="o">)</span>
Unable to find image <span class="s1">'debian:13'</span> locally
13: Pulling from library/debian
ac9148dc57ca: Already exists
Digest: sha256:3615a749858a1cba49b408fb49c37093db813321355a9ab7c1f9f4836341e9db
Status: Downloaded newer image <span class="k">for </span>debian:13
2026-03-09 11:28.03 <span class="nt">---</span><span class="o">&gt;</span> saved as <span class="s2">"4ea035d1f0cfdda7660f299954022c3a974ec9e1ba5d06b3a9aa2bca24fdcfb7"</span>

/: <span class="o">(</span>user <span class="o">(</span>uid 1000<span class="o">)</span> <span class="o">(</span>gid 1000<span class="o">))</span>

/: <span class="o">(</span>run <span class="o">(</span>shell <span class="s2">"echo Hello World"</span><span class="o">))</span>
Hello World
2026-03-09 11:28.05 <span class="nt">---</span><span class="o">&gt;</span> saved as <span class="s2">"e3859ae9dcce742a0d612e55f69b5ed1614551ca2b49109e43d08f2f2595fd57"</span>
Job succeeded
Result: <span class="s2">"e3859ae9dcce742a0d612e55f69b5ed1614551ca2b49109e43d08f2f2595fd57"</span>
</code></pre></div></div>

<p>OCluster doesn’t provide any native mechanism to copy artefacts back from the worker machine. In the CI pipeline, there have been creative methods to do this. For example,</p>

<ul>
  <li>print markers to the log, followed by JSON structured data, which can be extracted and parsed</li>
  <li>base64-encode some binary objects and print that to the log</li>
  <li>setup a remote SSH server and add steps to the build to <code class="language-plaintext highlighter-rouge">rsync</code> the data</li>
</ul>

<p>In a private environment, it may not be necessary to provide a secure upload and <code class="language-plaintext highlighter-rouge">curl -X POST -F file=@somefile.bin http://w.x.y.z:8080</code> to a one-line Python HTTP server may be sufficient.</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">python3</span> <span class="o">-</span><span class="n">c</span> <span class="s">"
  import cgi
  from http.server import HTTPServer, BaseHTTPRequestHandler
  class H(BaseHTTPRequestHandler):
      def do_POST(self):
          form = cgi.FieldStorage(fp=self.rfile, headers=self.headers, environ={'REQUEST_METHOD':'POST'})
          f = form['file']
          open(f.filename, 'wb').write(f.file.read())
          print(f'Saved {f.filename}')
          self.send_response(200)
          self.end_headers()
          self.wfile.write(b'OK</span><span class="se">\n</span><span class="s">')
  HTTPServer(('0.0.0.0', 8080), H).serve_forever()
"</span>
</code></pre></div></div>

<p>However, if you do need an authentication mechanism, <code class="language-plaintext highlighter-rouge">ocluster-client</code> supports secrets via the command-line option <code class="language-plaintext highlighter-rouge">--secret foo:/path/to/local/file</code>, which can be read into an environment variable or placed in <code class="language-plaintext highlighter-rouge">~/.ssh/id_ed25519</code> or similar. e.g.</p>

<pre><code class="language-sexp">(run
  (secrets (foo (target /path/on/remote/filesystem)))
  (shell "TOKEN=$(cat /path/on/remote/filesystem) curl -H \"X-Token: $TOKEN\" ... "))
</code></pre>

<p>I used <code class="language-plaintext highlighter-rouge">m4</code> to process my spec, replacing <code class="language-plaintext highlighter-rouge">LAT</code> and <code class="language-plaintext highlighter-rouge">LON</code> with actual latitude and longitude values, and submitted the jobs with a simple <code class="language-plaintext highlighter-rouge">bash</code> loop that invoked <code class="language-plaintext highlighter-rouge">ocluster-client</code> with each spec and redirected stdout to a log file.</p>
