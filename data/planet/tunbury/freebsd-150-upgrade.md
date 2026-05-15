---
title: FreeBSD 15.0 Upgrade
description: FreeBSD 15.0 has been out for a while, with issue#1036 pending resolution.
  The CI update is easy, but the CI worker rosemary needed an upgrade and new base
  images first.
url: https://www.tunbury.org/2026/04/29/freebsd-15.0/
date: 2026-04-29T13:30:00-00:00
preview_image: https://www.tunbury.org/images/freebsd-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>FreeBSD 15.0 has been out for a while, with <a href="https://github.com/ocurrent/ocaml-ci/issues/1036">issue#1036</a> pending resolution. The CI update is easy, but the CI worker <code class="language-plaintext highlighter-rouge">rosemary</code> needed an upgrade and new base images first.</p>

<p>A quick poke around on <code class="language-plaintext highlighter-rouge">rosemary</code> confirmed the starting state.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># freebsd-version -kru</span>
14.3-RELEASE-p3
14.3-RELEASE-p3
14.3-RELEASE-p4
</code></pre></div></div>

<p>The first requirement is to install the latest patches as it’s a bit behind. The host is a generic kernel, no <code class="language-plaintext highlighter-rouge">/usr/src</code>, single-disk EFI boot (260M ESP, UFS root on <code class="language-plaintext highlighter-rouge">da0p2</code>, ZFS <code class="language-plaintext highlighter-rouge">obuilder</code> pool on <code class="language-plaintext highlighter-rouge">da0p3</code>), 87 packages, and 13 obuilder build jails currently running. Nothing custom in <code class="language-plaintext highlighter-rouge">rc.conf</code> or <code class="language-plaintext highlighter-rouge">loader.conf</code> beyond a serial console and the worker service.</p>

<p>A 15.0 host kernel will happily run the existing 14.3 base image jails via <code class="language-plaintext highlighter-rouge">COMPAT_FREEBSD14</code>, but the opposite is not true.</p>

<h1>Pausing the worker</h1>

<p>Before any updates, pause <code class="language-plaintext highlighter-rouge">rosemary</code> so the cluster builds can finish in an orderly way, and outstanding jobs will remain queued.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>ocluster-admin <span class="nt">--connect</span> ~/admin.cap pause <span class="nt">--wait</span> freebsd-x86_64 rosemary
Waiting <span class="k">for </span><span class="nb">jobs </span>to finish…
rosemary: Running <span class="nb">jobs</span>: 2
rosemary: Running <span class="nb">jobs</span>: 1
rosemary: Running <span class="nb">jobs</span>: 0
Success.
</code></pre></div></div>

<h1>Catching up on patches</h1>

<p>With the worker paused, install the pending patches to get to a clean baseline before the major upgrade.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># freebsd-update install</span>
src component not installed, skipped
Installing updates...
Restarting sshd after upgrade
Performing sanity check on sshd configuration.
Stopping sshd.
Waiting <span class="k">for </span>PIDS: 1967.
Performing sanity check on sshd configuration.
Starting sshd.
 <span class="k">done</span><span class="nb">.</span>
</code></pre></div></div>

<p>Reboot to pick up the new kernel.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>reboot
</code></pre></div></div>

<p>After the reboot, <code class="language-plaintext highlighter-rouge">freebsd-version</code> confirms a fully-patched starting point:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># freebsd-version -kru</span>
14.3-RELEASE-p11
14.3-RELEASE-p11
14.3-RELEASE-p11
</code></pre></div></div>

<h1>Upgrading to 15.0</h1>

<p>The major-version upgrade is the usual three-phase <code class="language-plaintext highlighter-rouge">freebsd-update</code> procedure: stage the patches, install the kernel, reboot, install the new userland, rebuild ports, then a final cleanup pass. With no <code class="language-plaintext highlighter-rouge">src</code> component installed, there are no source merges to worry about.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># freebsd-update -r 15.0-RELEASE upgrade</span>
src component not installed, skipped
Looking up update.FreeBSD.org mirrors... 3 mirrors found.
Fetching metadata signature <span class="k">for </span>14.3-RELEASE from update2.freebsd.org... <span class="k">done</span><span class="nb">.</span>
...
The following components of FreeBSD seem to be installed:
kernel/generic world/base

The following components of FreeBSD <span class="k">do </span>not seem to be installed:
kernel/generic-dbg world/base-dbg world/lib32 world/lib32-dbg

Does this look reasonable <span class="o">(</span>y/n<span class="o">)</span>? y
...
To <span class="nb">install </span>the downloaded upgrades, run <span class="s1">'freebsd-update [options] install'</span><span class="nb">.</span>
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">rosemary</code> has no locally-modified files in <code class="language-plaintext highlighter-rouge">/etc</code> that <code class="language-plaintext highlighter-rouge">freebsd-update</code> cares about, so the merge phase passed silently.</p>

<h2>A small detour: forgetting to reboot</h2>

<p>The correct flow is <code class="language-plaintext highlighter-rouge">install</code> -&gt; reboot -&gt; <code class="language-plaintext highlighter-rouge">install</code> again, with the first call installing the new kernel, the reboot activates it, and the second call installing the userland onto a kernel that already understands the new syscalls. I accidentally ran the two <code class="language-plaintext highlighter-rouge">install</code>s back-to-back without rebooting, which merged both stages into one:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># freebsd-update install</span>
src component not installed, skipped
Installing updates...
Restarting sshd after upgrade
Performing sanity check on sshd configuration.
Bad system call <span class="o">(</span>core dumped<span class="o">)</span>

Completing this upgrade requires removing old shared object files.
Please rebuild all installed 3rd party software <span class="o">(</span>e.g., programs
installed from the ports tree<span class="o">)</span> and <span class="k">then </span>run
<span class="s1">'freebsd-update [options] install'</span> again to finish installing updates.
</code></pre></div></div>

<p>The new 15.0 sshd had been started against the still-running 14.3 kernel and immediately tripped a syscall that the old kernel doesn’t have. As I said, <code class="language-plaintext highlighter-rouge">COMPAT_FREEBSD14</code> in the 15.0 kernel handles old binaries on a new kernel; there’s no compat layer in the other direction. However, a reboot was all that was required to get back on track.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>reboot
</code></pre></div></div>

<h2>Rebuilding ports against the 15.0 ABI</h2>

<p>The pkg ABI moves from <code class="language-plaintext highlighter-rouge">FreeBSD:14:amd64</code> to <code class="language-plaintext highlighter-rouge">FreeBSD:15:amd64</code>, so every installed package needs to be reinstalled. <code class="language-plaintext highlighter-rouge">pkg</code> itself goes first, since the new binary is what understands the new ABI:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># pkg-static install -f pkg</span>
pkg-static: Warning: Major OS version upgrade detected.  Running <span class="s2">"pkg bootstrap -f"</span> recommended
...
New version of pkg detected<span class="p">;</span> it needs to be installed first.
The following 1 package<span class="o">(</span>s<span class="o">)</span> will be affected <span class="o">(</span>of 0 checked<span class="o">)</span>:

Installed packages to be UPGRADED:
        pkg: 2.5.1 -&gt; 2.6.2_1 <span class="o">[</span>FreeBSD-ports]
...
<span class="o">[</span>1/1] Upgrading pkg from 2.5.1 to 2.6.2_1...
<span class="o">[</span>1/1] Extracting pkg-2.6.2_1: 100%
</code></pre></div></div>

<p>Then force a reinstall of every other package:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>pkg upgrade <span class="nt">-f</span>
</code></pre></div></div>

<p>87 packages, a few minutes. The only post-install message worth noting was a reminder that <code class="language-plaintext highlighter-rouge">tmux</code> needs to be detached and restarted after a binary swap.</p>

<h2>Final state</h2>

<p>The normal workflow expects a final <code class="language-plaintext highlighter-rouge">freebsd-update install</code> to remove obsolete shared libraries left over from the old userland. But because of the missed reboot, <code class="language-plaintext highlighter-rouge">pkg upgrade -f</code> didn’t find any dangling 14.x <code class="language-plaintext highlighter-rouge">.so</code> references for the cleanup pass to find as they’d already been removed:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># freebsd-update fetch</span>
...
No updates needed to update system to 15.0-RELEASE-p6.
<span class="c"># freebsd-update install</span>
src component not installed, skipped
No updates are available to install.
Run <span class="s1">'freebsd-update [options] fetch'</span> first.
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">freebsd-version</code> confirms a clean 15.0 baseline:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># freebsd-version -kru</span>
15.0-RELEASE-p6
15.0-RELEASE-p6
15.0-RELEASE-p6
</code></pre></div></div>

<h1>Updating the EFI loader</h1>

<p><code class="language-plaintext highlighter-rouge">freebsd-update</code> does not touch the EFI System Partition, so the loader binary on the ESP is still the one included with the original 14.x install. The kernel is happy to run under an older <code class="language-plaintext highlighter-rouge">loader.efi</code>, but it should be updated.</p>

<p><code class="language-plaintext highlighter-rouge">rosemary</code> keeps the ESP permanently mounted at <code class="language-plaintext highlighter-rouge">/boot/efi</code>, so the comparison and copy is straightforward:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c"># ls -la /boot/loader.efi /boot/efi/EFI/freebsd/loader.efi /boot/efi/EFI/BOOT/BOOTx64.efi</span>
<span class="nt">-rwxr-xr-x</span>  1 root wheel 660992 Jul 29  2025 /boot/efi/EFI/BOOT/BOOTx64.efi
<span class="nt">-rwxr-xr-x</span>  1 root wheel 660992 Jul 29  2025 /boot/efi/EFI/freebsd/loader.efi
<span class="nt">-r-xr-xr-x</span>  2 root wheel 665088 Apr 29 08:27 /boot/loader.efi
</code></pre></div></div>

<p>Two files need replacing: <code class="language-plaintext highlighter-rouge">EFI/freebsd/loader.efi</code> is what the FreeBSD NVRAM boot entry points at, and <code class="language-plaintext highlighter-rouge">EFI/BOOT/BOOTx64.efi</code> is the removable-media fallback the firmware tries when no NVRAM entry matches.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nb">cp</span> /boot/loader.efi /boot/efi/EFI/freebsd/loader.efi
<span class="nb">cp</span> /boot/loader.efi /boot/efi/EFI/BOOT/BOOTx64.efi
</code></pre></div></div>

<h1>Another reboot and resuming the worker</h1>

<p>A final reboot to confirm the host comes up under 15.0 with the replaced EFI, then unpause the worker:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>ocluster-admin <span class="nt">--connect</span> ~/admin.cap unpause freebsd-x86_64 rosemary
</code></pre></div></div>

<p>The four queued jobs picked straight up on the existing <code class="language-plaintext highlighter-rouge">freebsd-14.3-ocaml-*</code> base images, with the 14.3 jails running against the 15.0 kernel without issue.</p>

<h1>Building 15.0 base images alongside 14.3</h1>

<p>The next job is to build a fresh set of <code class="language-plaintext highlighter-rouge">freebsd-15.0-ocaml-*</code> base images on <code class="language-plaintext highlighter-rouge">obuilder</code> while the 14.3 ones are still serving live jobs. Three changes to <a href="https://github.com/ocurrent/freebsd-infra">ocurrent/freebsd-infra</a>:</p>

<ol>
  <li>Change <code class="language-plaintext highlighter-rouge">BSDINSTALL_DISTSITE</code> URL to <code class="language-plaintext highlighter-rouge">15.0-RELEASE</code>.</li>
  <li>Change the dataset-name template in the role from <code class="language-plaintext highlighter-rouge">freebsd-14.3-ocaml-</code> to <code class="language-plaintext highlighter-rouge">freebsd-15.0-ocaml-</code>. This is the image name in the obuilder spec <code class="language-plaintext highlighter-rouge">(from freebsd-14.3-ocaml-5.4)</code>  and since it includes the version <code class="language-plaintext highlighter-rouge">freebsd-15.0-ocaml-5.4</code> can sit on disk next to <code class="language-plaintext highlighter-rouge">freebsd-14.3-ocaml-5.4</code>.</li>
  <li>Set every entry in <code class="language-plaintext highlighter-rouge">playbook.yml</code> to <code class="language-plaintext highlighter-rouge">default: false</code> for this run. The <code class="language-plaintext highlighter-rouge">default: true</code> entry runs a <code class="language-plaintext highlighter-rouge">zfs clone</code> into <code class="language-plaintext highlighter-rouge">obuilder/base-image/freebsd</code>, which already exists from the 14.3 build; with all entries <code class="language-plaintext highlighter-rouge">false</code> the clone step is skipped, the existing default alias keeps serving the worker, and the new datasets are just inert until promoted. Only opam-health-check uses this default alias and that’s not scheduled to run at this moment.</li>
</ol>

<p>I dropped 5.3.0 from the version list at the same time as nothing calls it and it was only left as an upgrade-compatibility fallback. The list is now <code class="language-plaintext highlighter-rouge">busybox</code>, <code class="language-plaintext highlighter-rouge">4.14.3</code>, <code class="language-plaintext highlighter-rouge">5.4.1</code>.</p>

<p>The <code class="language-plaintext highlighter-rouge">busybox</code> image doesn’t include the FreeBSD version, so the role’s <code class="language-plaintext highlighter-rouge">zfs destroy -R -r obuilder/base-image/busybox</code> step does fire, and the existing 14.3 busybox is replaced with a 15.0 build. However, this is really fine as busybox is only used internally by ocluster’s worker periodic health check.</p>

<h2><code class="language-plaintext highlighter-rouge">bsdinstall</code> changes</h2>

<p>Running the playbook against <code class="language-plaintext highlighter-rouge">rosemary</code> revealed that the role’s <code class="language-plaintext highlighter-rouge">bsdinstall script</code> has been broken by a change in <code class="language-plaintext highlighter-rouge">bsdinstall</code> itself. This script was originally copied from a FreeBSD installation many years ago and things have moved on:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>pgrep <span class="nt">-fl</span> bsdinstall
42582 /bin/sh /usr/libexec/bsdinstall/script jail /obuilder/base-image/busybox/rootfs
42602 /usr/libexec/bsdinstall/scriptedpart
<span class="nv">$ </span>procstat <span class="nt">-f</span> 42602 | <span class="nb">grep</span> <span class="s2">"0 v"</span>
42602 scriptedpart  0 v c rw------ 9 0 - /dev/pts/1
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">scriptedpart</code> was sitting on <code class="language-plaintext highlighter-rouge">/dev/pts/1</code> waiting for input. The cause is that <code class="language-plaintext highlighter-rouge">bsdinstall script</code> now splits the script file at the first <code class="language-plaintext highlighter-rouge">#!</code> shebang into a <em>preamble</em> (sourced for variables like <code class="language-plaintext highlighter-rouge">PARTITIONS</code>, <code class="language-plaintext highlighter-rouge">DISTRIBUTIONS</code>, <code class="language-plaintext highlighter-rouge">BSDINSTALL_DISTSITE</code>) and a setup script. The role’s <code class="language-plaintext highlighter-rouge">jail</code> script was no longer in the right format to continue.</p>

<p>I could have figured out the new layout, but the easy fix is to stop using <code class="language-plaintext highlighter-rouge">bsdinstall script</code> and use the purpose-built <code class="language-plaintext highlighter-rouge">bsdinstall jail</code> instead. It does exactly what we want for a chroot install: no partitioning, supports <code class="language-plaintext highlighter-rouge">nonInteractive=YES</code> to skip the interactive dialogues, and reads the install URL from <code class="language-plaintext highlighter-rouge">BSDINSTALL_DISTSITE</code>.</p>

<div class="language-yaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="pi">-</span> <span class="na">name</span><span class="pi">:</span> <span class="s">Install FreeBSD in jail //base-image//rootfs</span>
  <span class="na">shell</span><span class="pi">:</span> <span class="s">bsdinstall jail //base-image//rootfs</span>
  <span class="na">environment</span><span class="pi">:</span>
    <span class="na">nonInteractive</span><span class="pi">:</span> <span class="s2">"</span><span class="s">YES"</span>
    <span class="na">BSDINSTALL_DISTSITE</span><span class="pi">:</span> <span class="s2">"</span><span class="s">https://download.freebsd.org/ftp/releases/amd64/amd64/15.0-RELEASE"</span>
    <span class="na">DISTRIBUTIONS</span><span class="pi">:</span> <span class="s2">"</span><span class="s">base.txz"</span>
</code></pre></div></div>

<p>Note that it’s important to wipe the cached download directory <code class="language-plaintext highlighter-rouge">/usr/freebsd-dist</code> as this currently holds the 14.3 installation. I have a block in <code class="language-plaintext highlighter-rouge">update.yml</code> for this and copied it over into <code class="language-plaintext highlighter-rouge">playbook.yml</code>.</p>

<h2>Result</h2>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>NAME                                                 USED
obuilder/base-image                                 8.11G
obuilder/base-image/busybox                          746M
obuilder/base-image/freebsd                            8K
obuilder/base-image/freebsd-14.3-ocaml-4.14         1.56G
obuilder/base-image/freebsd-14.3-ocaml-5.3          1.47G
obuilder/base-image/freebsd-14.3-ocaml-5.4          1.47G
obuilder/base-image/freebsd-15.0-ocaml-4.14         1.49G
obuilder/base-image/freebsd-15.0-ocaml-5.4          1.40G
obuilder/base-image/freebsd/rootfs                     8K
</code></pre></div></div>

<p>Five base-images are now available in the same pool: the 14.3 set still serving live work, the 15.0 set ready for promotion, and <code class="language-plaintext highlighter-rouge">obuilder/base-image/freebsd</code> (the default alias) still pointing at the 14.3-5.4 snapshot.</p>

<p>The infrastructure changes are bundled in <a href="https://github.com/ocurrent/freebsd-infra/pull/21">ocurrent/freebsd-infra#21</a>.</p>

<h1>Testing the new image</h1>

<p>Before promoting the new image, I built a known working package <a href="https://github.com/mtelvers/mandelbrot">mtelvers/mandelbrot</a> targeted explicitly at <code class="language-plaintext highlighter-rouge">freebsd-15.0-ocaml-5.4</code>:</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>ocluster-client <span class="nt">--connect</span> ~/mtelvers.cap submit-obuilder <span class="se">\</span>
  <span class="nt">--pool</span> freebsd-x86_64 <span class="nt">--local-file</span> mandelbrot.spec <span class="se">\</span>
  https://github.com/mtelvers/mandelbrot 14e08f30f087994a19822546a55405d078acd0d3
</code></pre></div></div>

<p>The spec’s first step is <code class="language-plaintext highlighter-rouge">(from freebsd-15.0-ocaml-5.4)</code>, so ocluster picks up the new dataset directly. Result:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>...
FreeBSD 15.0-RELEASE-p6
The OCaml toplevel, version 5.4.1
2.5.0
...
Job succeeded
</code></pre></div></div>

<p>OCaml 5.4.1 runs against the 15.0, opam pulls dependencies, dune builds the package and runs the tests showing that the base image is functionally complete.</p>

<h1>Promoting the new default</h1>

<p>Since the image is working the default image <code class="language-plaintext highlighter-rouge">obuilder/base-image/freebsd</code> can be repointed at the 15.0-5.4 snapshot.</p>

<div class="language-sh highlighter-rouge"><div class="highlight"><pre class="highlight"><code>zfs destroy <span class="nt">-r</span> obuilder/base-image/freebsd
zfs clone <span class="nt">-p</span> obuilder/base-image/freebsd-15.0-ocaml-5.4@snap obuilder/base-image/freebsd
zfs clone <span class="nt">-p</span> obuilder/base-image/freebsd-15.0-ocaml-5.4/rootfs@snap obuilder/base-image/freebsd/rootfs
zfs snapshot <span class="nt">-r</span> obuilder/base-image/freebsd@snap
</code></pre></div></div>

<h1>CI services</h1>

<p><a href="https://github.com/ocurrent/opam-repo-ci">opam-repo-ci</a> and <a href="https://github.com/ocurrent/ocaml-ci">ocaml-ci</a> each have a hardcoded <code class="language-plaintext highlighter-rouge">freebsd-X.Y</code> distro string that needs updating. Both follow the same template as the previous 14.2 -&gt; 14.3 pass.</p>

<p>For opam-repo-ci, <a href="https://github.com/ocurrent/opam-repo-ci/pull/472">PR#472</a> updates three files: <code class="language-plaintext highlighter-rouge">opam-ci-check/lib/variant.ml</code> (the <code class="language-plaintext highlighter-rouge">freebsd</code> distro constant), <code class="language-plaintext highlighter-rouge">doc/platforms.md</code> (the supported-platforms list and the matrix table), and <code class="language-plaintext highlighter-rouge">test/specs.expected</code>. The expected output is regenerated to match the new constant; <code class="language-plaintext highlighter-rouge">dune runtest</code> confirms the diff is consistent.</p>

<p>For ocaml-ci, <a href="https://github.com/ocurrent/ocaml-ci/pull/1051">PR#1051</a> updates two files: <code class="language-plaintext highlighter-rouge">lib/variant.ml</code> (the <code class="language-plaintext highlighter-rouge">freebsd_distributions</code> list) and <code class="language-plaintext highlighter-rouge">service/conf.ml</code> (the per-OCaml-version platform record and the <code class="language-plaintext highlighter-rouge">fetch_platforms</code> match that special-cases distros backed by ZFS snapshots rather than Docker images).</p>

<p>With both PRs green on CI, they were merged, and the changes were pushed to the live branches; the workers now submit jobs against the <code class="language-plaintext highlighter-rouge">freebsd-15.0-ocaml-*</code> base images by default.</p>
