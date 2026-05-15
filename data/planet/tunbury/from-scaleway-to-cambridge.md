---
title: From Scaleway to Cambridge
description: Over the past few days, I migrated several OCaml CI services from Scaleway
  to Cambridge, consolidating them onto fewer machines with fewer services.
url: https://www.tunbury.org/2026/04/01/from-scaleway-to-cambridge/
date: 2026-04-01T16:00:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Over the past few days, I migrated several OCaml CI services from Scaleway to Cambridge, consolidating them onto fewer machines with fewer services.</p>

<h1>watch.ocaml.org</h1>

<p>The first migration was the PeerTube instance behind <a href="https://watch.ocaml.org">watch.ocaml.org</a>. This ran on Scaleway as a Docker Swarm stack with PeerTube, PostgreSQL, Redis, Postfix, nginx, and certbot.</p>

<p>The Ansible playbook still referenced Tarsnap for backups, but Borg Backup had been used for backups for some time without the playbook being updated. I fixed the playbook to match reality, deploying the Borg SSH key, SSH config, and daily cron job.</p>

<p>The data migration was straightforward. I rsync’d the Docker volumes while the service was still running. The bulk of the 118 GB was static video data in the <code class="language-plaintext highlighter-rouge">peertube-data</code> volume. Once the initial copy finished, I scaled the services to zero, ran a final rsync to catch any remaining writes, and brought everything back up while running the playbook against the new host.</p>

<p>The new server is <code class="language-plaintext highlighter-rouge">svr-avsm2-watch.cl.cam.ac.uk</code> in Cambridge.</p>

<h1>ci.mirageos.org</h1>

<p>The more involved migration was <a href="https://ci.mirageos.org">ci.mirageos.org</a>, which ran three services: mirage-ci (the MirageOS CI), a deployer for MirageOS services, and gogs (a git server). Gogs turned out to be empty, so I dropped it.</p>

<p>The target was <code class="language-plaintext highlighter-rouge">chives.caelum.ci.dev</code>, which already hosts <a href="https://ocaml.ci.dev">ocaml.ci.dev</a> and <a href="https://opam.ci.ocaml.org">opam.ci.ocaml.org</a>. Since the service names didn’t clash, everything went into the existing <code class="language-plaintext highlighter-rouge">infra</code> Docker Swarm stack.</p>

<p>The mirage-ci service on the old server mounted capability files from the host filesystem at <code class="language-plaintext highlighter-rouge">/home/camel/mirage-ci/cap/</code>. I converted these to Docker secrets to match the pattern used by the other services on chives.</p>

<p>I renamed the deployer secrets with a <code class="language-plaintext highlighter-rouge">mirage-deployer-</code> prefix to avoid clashing with any existing secrets, while keeping the shared <code class="language-plaintext highlighter-rouge">ocurrentbuilder-password</code> and <code class="language-plaintext highlighter-rouge">ocurrent-hub</code> secrets common.</p>

<p>I extended the Caddy reverse proxy on chives with the mirageos routes. It turned out that the OCurrent web servers don’t support HTTP/2, causing Caddy to return 400 errors. The fix was to force HTTP/1.1 in the proxy transport configuration. I’m surprised this hasn’t been an issue before, but I was focused on getting the services up as quickly as I could.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>ci.mirageos.org {
  reverse_proxy mirage-ci:8080 {
    transport http {
      versions 1.1
    }
  }
}
</code></pre></div></div>

<p>I merged the Let’s Encrypt certificates from the old server’s caddy data volume into the one on chives, so there was no TLS interruption after the DNS switch.</p>

<h2>DNS</h2>

<p>The MirageOS nameservers are MirageOS unikernels. Zone changes are pushed to a git repository, and then you need to trigger a reload of the nameserver using a an authenticated DNS notification using TSIG keys. I pushed the A record changes with <code class="language-plaintext highlighter-rouge">nsupdate</code> for <code class="language-plaintext highlighter-rouge">ci.mirageos.org</code>, <code class="language-plaintext highlighter-rouge">deploy.mirageos.org</code>, <code class="language-plaintext highlighter-rouge">ci.mirage.io</code>, and <code class="language-plaintext highlighter-rouge">deploy.mirage.io</code>, then caused an updated:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>nsupdate -y hmac-sha256:deploy._update:&lt;secret&gt; &lt;&lt;EOF
server ns0.mirageos.org
zone mirageos.org
update delete ci.mirageos.org A
update add ci.mirageos.org 10800 A 128.232.124.253
send
EOF
</code></pre></div></div>

<h2>The deployer</h2>

<p>The <a href="https://github.com/ocurrent/ocurrent-deployer">ocurrent-deployer</a> needed updating to reflect the new host. I changed the Docker context from <code class="language-plaintext highlighter-rouge">ci.mirageos.org</code> to <code class="language-plaintext highlighter-rouge">chives.caelum.ci.dev</code> and renamed the deployer service from <code class="language-plaintext highlighter-rouge">infra_deployer</code> to <code class="language-plaintext highlighter-rouge">infra_mirage-deployer</code>. The caddy service was dropped since caddy is already managed on chives. See <a href="https://github.com/ocurrent/ocurrent-deployer/pull/259">PR#259</a>.</p>

<h2>The mirage-www unikernel</h2>

<p>The deployer builds a <a href="https://github.com/mirage/mirage-www">mirage-www</a> unikernel and deploys it to an Equinix Metal bare-metal server running <a href="https://github.com/robur-coop/albatross">albatross</a>. The build was failing because the base image <code class="language-plaintext highlighter-rouge">ocaml/opam:debian-12-ocaml-4.14</code> had been updated to OCaml 4.14.3, which conflicted with the pinned opam-repository snapshot.</p>

<p>I fixed this by pinning the base image to a specific digest, locking it to 4.14.2 (<a href="https://github.com/mirage/mirage-www/pull/864">PR#864</a>, <a href="https://github.com/mirage/mirage-www/pull/865">PR#865</a>).</p>

<h1>Eliminating the Docker socket</h1>

<p>The mirage deployer previously built unikernel Docker images locally and needed the Docker socket mounted into its container, which is a security concern. The build process was:</p>

<ol>
  <li><code class="language-plaintext highlighter-rouge">docker build</code> the mirage-www Dockerfile locally</li>
  <li><code class="language-plaintext highlighter-rouge">docker run</code> + <code class="language-plaintext highlighter-rouge">docker cp</code> to extract the <code class="language-plaintext highlighter-rouge">.hvt</code> unikernel binary</li>
  <li><code class="language-plaintext highlighter-rouge">rsync</code> to the Equinix host</li>
  <li><code class="language-plaintext highlighter-rouge">ssh mirage-redeploy</code> to restart the unikernel via albatross</li>
</ol>

<p>I replaced this with OCluster builds. The unikernel Dockerfile is now submitted to OCluster, which builds it on a worker and pushes the result to the <code class="language-plaintext highlighter-rouge">ocurrentbuilder/staging</code> registry. The deployer then uses <a href="https://github.com/google/go-containerregistry">crane</a> to extract the <code class="language-plaintext highlighter-rouge">.hvt</code> binary from the registry image without needing a Docker daemon at all.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>crane export ocurrentbuilder/staging@sha256:... - | tar xf - --to-stdout unikernel.hvt &gt; /tmp/www.hvt
</code></pre></div></div>

<p>See <a href="https://github.com/ocurrent/ocurrent-deployer/pull/260">PR#260</a>.</p>

<h1>scheduler.ci.dev</h1>

<p>The third Scaleway server hosts the OCluster scheduler, the base-images builder, the ci.dev deployer, and two smaller services: sandmark-nightly and the ocurrent.org watcher. These are being migrated to chives one at a time.</p>

<h2>sandmark and watcher</h2>

<p>Sandmark is stateless with no secrets, so I added it to the infra stack on chives, copied the TLS certificates from the old server’s caddy data volume, and updated DNS (<a href="https://github.com/ocurrent/ocurrent-deployer/pull/261">PR#261</a>). This pattern of pre-copying certificates avoids any TLS interruption during the DNS switch.</p>

<p>The watcher is an OCurrent pipeline that monitors GitHub repos in the <code class="language-plaintext highlighter-rouge">ocurrent</code> org, fetches their READMEs, builds a Hugo site, and pushes the output to GitHub Pages. It needed Docker secrets for GitHub auth and an SSH deploy key.</p>

<p>After migrating it to chives, several issues surfaced: Hugo had removed the <code class="language-plaintext highlighter-rouge">-v</code> flag (replaced by <code class="language-plaintext highlighter-rouge">--logLevel</code>), the <code class="language-plaintext highlighter-rouge">--verbose</code> flag was also gone, the <code class="language-plaintext highlighter-rouge">path</code> front matter field was deprecated, raw HTML rendering needed enabling, and the SSH config in the Docker image was missing a <code class="language-plaintext highlighter-rouge">Host</code> line. Each required a fix to the <a href="https://github.com/ocurrent/ocurrent.org">ocurrent.org</a> repo (<a href="https://github.com/ocurrent/ocurrent.org/pull/28">PR#28</a>).</p>

<h2>Retiring the watcher</h2>

<p>After fixing all of this, the question was: why run a full OCurrent pipeline, Docker image, and deployer entry just to fetch some READMEs and run Hugo? As I could see no reason for this level of complexity and maintenance, I replaced the entire pipeline with a <a href="https://github.com/ocurrent/ocurrent.org/blob/master/.github/workflows/build.yml">GitHub Actions workflow</a> that does the same thing. It runs on push and monthly, fetches docs from the tracked repos, generates index pages, builds with Hugo, and pushes to the <code class="language-plaintext highlighter-rouge">gh-pages</code> branch. No Docker images, no deployer, no secrets to manage. I removed the watcher from the deployer pipeline (<a href="https://github.com/ocurrent/ocurrent-deployer/pull/262">PR#262</a>).</p>

<p>I also moved the GitHub Pages custom domain from the <a href="https://github.com/ocurrent/ocurrent.github.io">ocurrent.github.io</a> repo (now archived) to the source repo itself, simplifying the deployment to a single repository.</p>

<h2>The Tarides deployer for ci.dev</h2>

<p>The Tarides deployer (<code class="language-plaintext highlighter-rouge">deploy.ci.dev</code>) manages deployments for OCaml CI, opam-repo-ci, sandmark, and other services. It has a 2.1 GB state volume containing the SQLite database, git caches, and job logs.</p>

<p>The key concern was avoiding mass redeployments. The deployer decides what to deploy by comparing git HEAD with its last recorded deployment in the SQLite database. By copying the state volume faithfully, the deployer on chives sees everything as already deployed and settles immediately.</p>

<p>The only wrinkle is that the deployer updates its own service. On the old server, the service was called <code class="language-plaintext highlighter-rouge">deployer_deployer</code>; on chives, it’s <code class="language-plaintext highlighter-rouge">infra_tarides-deployer</code>. The first deploy attempt after migration failed because the old code still referenced the old name. I updated the deployer pipeline (<a href="https://github.com/ocurrent/ocurrent-deployer/pull/263">PR#263</a>), but since the running deployer couldn’t update itself (wrong service name), I had to manually pull the new image and update the service once.</p>

<h2>Base-images builder</h2>

<p>The <a href="https://images.ci.ocaml.org">base image builder</a> creates the Docker base images used by all OCaml CI services. It submits builds to OCluster and pushes results to Docker Hub. It has a 13 GB state volume and a small capnp-secrets volume for its capability listener on port 8101.</p>

<p>I followed the same approach: scale down, rsync both volumes to chives, add to the infra stack, update DNS. The SQLite state ensured no rebuilds were triggered, and the builder settled immediately. See <a href="https://github.com/ocurrent/ocurrent-deployer/pull/264">PR#264</a> for the deployer pipeline update.</p>

<h2>The OCluster scheduler</h2>

<p>The scheduler is the hub that all workers connect to via capnp on port 8103. Every worker, solver, and CI service holds a capability reference to <code class="language-plaintext highlighter-rouge">scheduler.ci.dev:8103</code>. The critical piece is the capnp-secrets volume containing the private key, since all capabilities are derived from it. As long as that key is preserved, all existing worker connections remain valid after a DNS switch.</p>

<p>I scaled the scheduler down, rsync’d the 8.3 GB state volume and capnp-secrets volume to chives, added the service to the infra stack, and updated DNS. Workers reconnected within seconds and resumed taking jobs immediately.</p>

<p>The old server ran its own Prometheus instance scraping per-worker metrics via the scheduler’s API. With everything on chives, I merged that config into the existing Prometheus instance, eliminating the second Prometheus and the federation hop that connected them.</p>

<h2>Summary</h2>

<p>All three Scaleway servers have been migrated to Cambridge.</p>

<table>
  <thead>
    <tr>
      <th>Service</th>
      <th>Old host</th>
      <th>New host</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>watch.ocaml.org</td>
      <td>Scaleway (Paris)</td>
      <td>svr-avsm2-watch.cl.cam.ac.uk</td>
    </tr>
    <tr>
      <td>ci.mirageos.org</td>
      <td>Scaleway (Paris)</td>
      <td>chives.caelum.ci.dev</td>
    </tr>
    <tr>
      <td>deploy.mirageos.org</td>
      <td>Scaleway (Paris)</td>
      <td>chives.caelum.ci.dev</td>
    </tr>
    <tr>
      <td>sandmark.tarides.com</td>
      <td>Scaleway (Paris)</td>
      <td>chives.caelum.ci.dev</td>
    </tr>
    <tr>
      <td>watcher.ci.dev</td>
      <td>Scaleway (Paris)</td>
      <td>GitHub Actions</td>
    </tr>
    <tr>
      <td>deploy.ci.dev</td>
      <td>Scaleway (Paris)</td>
      <td>chives.caelum.ci.dev</td>
    </tr>
    <tr>
      <td>images.ci.ocaml.org</td>
      <td>Scaleway (Paris)</td>
      <td>chives.caelum.ci.dev</td>
    </tr>
    <tr>
      <td>scheduler.ci.dev</td>
      <td>Scaleway (Paris)</td>
      <td>chives.caelum.ci.dev</td>
    </tr>
  </tbody>
</table>
