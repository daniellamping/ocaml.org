---
title: Linux input devices (with libinput-ocaml)
description:
url: https://roscidus.com/blog/blog/2026/03/28/input-devices/
date: 2026-03-28T09:00:00-00:00
preview_image:
authors:
- Thomas Leonard
source:
ignore:
---

<p>I've been investigating how keyboards, mice, etc work in Linux.
In this blog post we'll see how input events work, using libinput-ocaml,
and then use that to write a little game.</p>

<p><a href="https://roscidus.com/blog/images/lander/pad.png"><span class="caption-wrapper center"><img src="https://roscidus.com/blog/images/lander/pad.png" title="Using libinput-ocaml to make a game" class="caption"><span class="caption-text">Using libinput-ocaml to make a game</span></span></a></p>
<p><strong>Table of Contents</strong></p>
<ul>
<li><a href="https://roscidus.com/#device-files">Device files</a>
<ul>
<li><a href="https://roscidus.com/#using-devices-directly">Using devices directly</a>
</li>
<li><a href="https://roscidus.com/#permissions">Permissions</a>
</li>
</ul>
</li>
<li><a href="https://roscidus.com/#libinput">libinput</a>
<ul>
<li><a href="https://roscidus.com/#getting-a-repl">Getting a REPL</a>
</li>
<li><a href="https://roscidus.com/#opening-devices-with-libinput">Opening devices with libinput</a>
</li>
<li><a href="https://roscidus.com/#thoughts-on-the-c-bindings">Thoughts on the C bindings</a>
</li>
</ul>
</li>
<li><a href="https://roscidus.com/#lander-game">Lander game</a>
<ul>
<li><a href="https://roscidus.com/#non-libinput-aspects-of-the-game">Non-libinput aspects of the game</a>
</li>
</ul>
</li>
</ul>
<h2>Device files</h2>
<p>My Wayland compositor (Sway) makes use of the keyboard and mouse, so let's see what devices it has open
(note: all shell commands are using <a href="https://fishshell.com/">fish</a> syntax):</p>
<pre><code>$ lsof -p (pgrep -x sway) -a +D /dev
COMMAND  PID USER  FD   TYPE  DEVICE SIZE/OFF NODE NAME
sway    2031  tal mem    CHR 226,128           892 /dev/dri/renderD128
sway    2031  tal   0u   CHR     4,1      0t0   20 /dev/tty1
...
sway    2031  tal  46u   CHR   13,69      0t0  648 /dev/input/event5
sway    2031  tal  47u   CHR   13,68      0t0  647 /dev/input/event4
sway    2031  tal  48u   CHR   13,67      0t0  646 /dev/input/event3
sway    2031  tal  49u   CHR   13,64      0t0  641 /dev/input/event0
sway    2031  tal  50u   CHR   13,65      0t0  644 /dev/input/event1
sway    2031  tal  52r   CHR     1,9      0t0    9 /dev/urandom
sway    2031  tal  54u   CHR   13,66      0t0  645 /dev/input/event2
</code></pre>
<p><code>/dev/dri</code> is for graphics cards, which I looked at earlier -
see <a href="https://roscidus.com/blog/blog/2025/06/24/graphics/">Investigating Linux graphics</a> for details.
<code>/dev/tty1</code> is the Linux console where Sway is running.
<code>/dev/input/event*</code> are the input devices.
The names aren't very helpful, but I see I also have some symlinks e.g.</p>
<pre><code>$ ls -l /dev/input/by-id/usb-Logitech_USB_Optical_Mouse-event-mouse
lrwxrwxrwx 1 root root 9 2026-02-28 07:46
  /dev/input/by-id/usb-Logitech_USB_Optical_Mouse-event-mouse -&gt; ../event2
</code></pre>
<p>So <code>event2</code> is the mouse. Not all of them have symlinks though, and if I plug in a second identical mouse then I get a new <code>event18</code> device but still only one symlink, so this doesn't seem very useful.
What creates these device files?</p>
<pre><code>$ sudo opensnoop | grep /dev/input
[ I plug in a second USB mouse here ]
45754  (udev-worker)      21   0 /dev/input/mouse1
45757  (udev-worker)      25   0 /dev/input/event18
45757  (udev-worker)      25   0 /dev/input/event18
1236   systemd-logind     36   0 /dev/input/event18
</code></pre>
<p>The device files are created by <a href="https://www.man7.org/linux/man-pages/man8/systemd-udevd.8.html">systemd-udevd</a>.
The <code>mouse*</code> devices seems to be from some older event system and we can ignore them.</p>
<p>When the Linux kernel detects a new device, it adds some information about it under
<code>/sys/devices/</code> and broadcasts an event (in the <code>NETLINK_KOBJECT_UEVENT</code> <a href="https://www.man7.org/linux/man-pages/man7/netlink.7.html">netlink</a> group).
Any process can subscribe to these events, and you can see them like this:</p>
<pre><code>$ udevadm monitor --kernel
monitor will print the received events for:
KERNEL - the kernel uevent
[ I plug in a USB mouse here ]
KERNEL[56090.928942] add      /devices/pci0000:00/0000:00:14.0/usb1/1-10 (usb)
KERNEL[56090.929735] change   /devices/pci0000:00/0000:00:14.0/usb1/1-10 (usb)
KERNEL[56090.930056] add      /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0 (usb)
KERNEL[56090.933228] add      /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/0003:046D:C077.0009 (hid)
KERNEL[56090.933351] add      /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/0003:046D:C077.0009/input/input22 (input)
KERNEL[56090.933401] add      /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/0003:046D:C077.0009/input/input22/mouse1 (input)
KERNEL[56090.933445] add      /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/0003:046D:C077.0009/input/input22/event18 (input)
KERNEL[56090.933500] add      /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/0003:046D:C077.0009/hidraw/hidraw5 (hidraw)
KERNEL[56090.933571] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/0003:046D:C077.0009 (hid)
KERNEL[56090.933642] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0 (usb)
KERNEL[56090.933714] bind     /devices/pci0000:00/0000:00:14.0/usb1/1-10 (usb)
</code></pre>
<p>There's lots of information here. For example, to see the new device's name:</p>
<pre><code>$ cat /sys/devices/pci0000:00/0000:00:14.0/usb1/1-10/1-10:1.0/0003:046D:C077.0009/input/input22/event18/device/name 
Logitech USB Optical Mouse
</code></pre>
<p><code>systemd-udevd</code> gets these events and creates the device files in <code>/dev/input/</code> according to its configuration. You can see what it will do like this (I've trimmed the output a lot):</p>
<pre><code>$ udevadm test /dev/input/event18
event18: /nix/store/...systemd-258.3/lib/udev/rules.d/50-udev-default.rules:48
  GROUP="input": Set group ID: 174
event18: /nix/store/...systemd-258.3/lib/udev/rules.d/60-persistent-input.rules:28
  SYMLINK+="input/by-id/$env{ID_BUS}-$env{ID_SERIAL}-event-$env{.INPUT_CLASS}":
    Added device node symlink "input/by-id/usb-Logitech_USB_Optical_Mouse-event-mouse".
event18: /nix/store/...systemd-258.3/lib/udev/rules.d/60-persistent-input.rules:38
  SYMLINK+="input/by-path/$env{ID_PATH}-event-$env{.INPUT_CLASS}":
    Added device node symlink "input/by-path/pci-0000:00:14.0-usb-0:10:1.0-event-mouse".
event18: /nix/store/...systemd-258.3/lib/udev/rules.d/60-persistent-input.rules:39
  SYMLINK+="input/by-path/$env{ID_PATH_WITH_USB_REVISION}-event-$env{.INPUT_CLASS}":
    Added device node symlink "input/by-path/pci-0000:00:14.0-usbv2-0:10:1.0-event-mouse".
...
Device name:
  event18
Device node:
  /dev/input/event18 (char 13:82)
Device node owner group:
  input (gid=174)
Device node symlinks: (priority=0)
  /dev/input/by-path/pci-0000:00:14.0-usb-0:10:1.0-event-mouse
  /dev/input/by-path/pci-0000:00:14.0-usbv2-0:10:1.0-event-mouse
  /dev/input/by-id/usb-Logitech_USB_Optical_Mouse-event-mouse
Properties:
  .INPUT_CLASS=mouse
  ACTION=add
...
</code></pre>
<h3>Using devices directly</h3>
<p>Let's see the event data coming from the new mouse:</p>
<pre><code>$ hexdump /dev/input/event18
hexdump: /dev/input/event18: Permission denied

$ sudo hexdump /dev/input/event18
0000000 b9c5 69a6 0000 0000 d5ca 0008 0000 0000
...

$ sudo hexdump /dev/input/event18 \
    -e '/8 "%d." /8 "%06d" /2 " ty=%x" /2 " code=%x" 1/4 " val=%2d\n"'
1772534246.362989 ty=2 code=0 val= 3
1772534246.362989 ty=2 code=1 val=-1
1772534246.362989 ty=0 code=0 val= 0
</code></pre>
<p>The <code>evtest</code> utility formats things better:</p>
<pre><code>$ sudo evtest /dev/input/event18
Event: time 1772534337.699150, type 2 (EV_REL), code 0 (REL_X), value 6
Event: time 1772534337.699150, type 2 (EV_REL), code 1 (REL_Y), value -2
Event: time 1772534337.699150, -------------- SYN_REPORT ------------
Event: time 1772534337.707193, type 2 (EV_REL), code 0 (REL_X), value 4
Event: time 1772534337.707193, type 2 (EV_REL), code 1 (REL_Y), value -2
Event: time 1772534337.707193, -------------- SYN_REPORT ------------
</code></pre>
<p>So, when I move the mouse the kernel generates an event giving the relative motion for the X axis,
then another for the Y axis, then a sync marker to finish the group.
Seems simple enough.</p>
<h3>Permissions</h3>
<p>I had to use <code>sudo</code> above to be able to open the device.
But Sway manages to use it without help. How did it do that?</p>
<pre><code>$ strace -y -p (pgrep -x sway) &amp;| grep /dev/input
[ I plug in a USB mouse here ]
newfstatat(AT_FDCWD&lt;/home/tal&gt;, "/dev/input/event18",
           {st_mode=S_IFCHR|0660, st_rdev=makedev(0xd, 0x52), ...}, 0) = 0
recvmsg(6&lt;socket:[14582]&gt;, {
  msg_name=NULL, msg_namelen=0, msg_iov=[{iov_base="...", iov_len=24}], msg_iovlen=1,
  msg_control=[{cmsg_len=20, cmsg_level=SOL_SOCKET, cmsg_type=SCM_RIGHTS,
                cmsg_data=[101&lt;/dev/input/event18&gt;]}],
  msg_controllen=24, msg_flags=MSG_CMSG_CLOEXEC}, MSG_DONTWAIT|MSG_CMSG_CLOEXEC) = 24
</code></pre>
<p>Sway didn't open <code>event18</code> itself. Instead, it received the handle over a Unix domain socket (FD 6).
What is FD 6 connected to?</p>
<pre><code>$ sudo lsof -a -U +E -p (pgrep -x sway) -d6
COMMAND    PID       USER FD   TYPE             DEVICE SIZE/OFF  NODE NAME
dbus-daem 1217 messagebus 16u  unix 0xffff8d960b22b800      0t0  9814 /run/dbus/system_bus_socket type=STREAM -&gt;INO=14582 2031,sway,8u 2031,sway,6u (CONNECTED)
sway      2031        tal  6u  unix 0xffff8d960b228800      0t0 14582 @ea2d5fdc608641b3/bus/sway/system type=STREAM -&gt;INO=9814 1217,dbus-daem,16u (CONNECTED)
...
</code></pre>
<p>Sway is using FD 6 to communicate via DBus.
Let's capture the messages and see what it's saying:</p>
<pre><code>$ sudo busctl capture --system &gt; plug-mouse.pcap
Monitoring bus message stream.
[ I plug in a USB mouse here ]
^C
</code></pre>
<p>Examining the capture with <code>wireshark</code>, I see that Sway sent a <code>TakeDevice</code> message
to the <a href="https://www.freedesktop.org/software/systemd/man/latest/org.freedesktop.login1.html">org.freedesktop.login1</a> service (provided by <code>systemd-logind</code>)
with the major and minor number of the device it wanted.
The privileged <code>systemd-logind</code> service returned a handle to the device
(you might have noticed this service opened the device in the <code>opensnoop</code> output earlier).</p>
<p>The documentation notes that:</p>
<blockquote>
<p>systemd-logind automatically mutes the file descriptor if the session is inactive and
resumes it once the session is activated again.
This guarantees that a session can only access session devices if the session is active.</p>
</blockquote>
<p>That makes sense, but it's weird how Linux seems to have three different permission systems here:</p>
<ul>
<li>For graphics output, all users (anyone in the <code>video</code> group) can open the device, but
only the first becomes the "master" and can control it.
</li>
<li>For mice and keyboards, the devices are owned by a privileged service and you must request them
over DBus. The service revokes access when you switch user.
</li>
<li>For joysticks, the permissions on the device allow the currently-logged-in user to open them directly,
and access is not revoked when switching users.
</li>
</ul>
<h2>libinput</h2>
<p>Dealing with input from actual mice looks fairly easy,
but things get very complicated when you add in touchpads, gesture recognition, etc.
<a href="https://gitlab.freedesktop.org/libinput/libinput">libinput</a> knows how to handle a wide variety of devices and provides a uniform API for using them
(it also has surprisingly good <a href="https://wayland.freedesktop.org/libinput/doc/latest/features.html">user documentation</a>).
I made some OCaml bindings to make it easier to play with it.</p>
<h3>Getting a REPL</h3>
<p>If you want to follow along, you'll need to install <a href="https://github.com/talex5/libinput-ocaml">libinput-ocaml</a> and an interactive REPL like <a href="https://github.com/ocaml-community/utop">utop</a>.
With Nix, you can set everything up like this:</p>
<pre><code>git clone https://github.com/talex5/libinput-ocaml.git
cd libinput-ocaml
nix develop
dune utop
</code></pre>
<p>You should see a <code>utop #</code> prompt, where you can enter OCaml expressions.
Use <code>;;</code> to tell the REPL you've finished typing and it's time to evaluate, e.g.</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="mi">1</span><span class="o">+</span><span class="mi">1</span><span class="o">;;</span>
</span><span class="line"><span class="o">-</span> <span class="o">:</span> <span class="kt">int</span> <span class="o">=</span> <span class="mi">2</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>Alternatively, you can install things using <a href="https://opam.ocaml.org/">opam</a> (OCaml's package manager).
Note: you'll need a Linux distribution with libinput 1.20 or later.</p>
<pre><code>opam install libinput utop
utop
</code></pre>
<p>Then, at the utop prompt enter <code>#require "libinput";;</code> (including the leading <code>#</code>).</p>
<h3>Opening devices with libinput</h3>
<p>To use libinput, you start by creating a context:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="k">let</span> <span class="n">ctx</span> <span class="o">=</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Context</span><span class="p">.</span><span class="nn">Path</span><span class="p">.</span><span class="n">create</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Interface</span><span class="p">.</span><span class="n">unix_direct</span><span class="o">;;</span>
</span><span class="line"><span class="k">val</span> <span class="n">ctx</span> <span class="o">:</span> <span class="o">[&gt;</span> <span class="o">`</span><span class="nc">Path</span> <span class="o">]</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Context</span><span class="p">.</span><span class="n">t</span> <span class="o">=</span> <span class="o">&lt;</span><span class="n">abstr</span><span class="o">&gt;</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>Here, I'm using a "path" context, where we add devices manually.
You can also use a "udev" one and it will find all the devices automatically.</p>
<p>The "interface" says how libinput should open the device files.
<code>unix_direct</code> means we just try to open them directly using OCaml's <code>Unix</code> module
(without using DBus).</p>
<p>It's a good idea to turn on logging at this point:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Context</span><span class="p">.</span><span class="nn">Log</span><span class="p">.</span><span class="n">set_priority</span> <span class="n">ctx</span> <span class="nc">Info</span><span class="o">;;</span>
</span><span class="line"><span class="o">-</span> <span class="o">:</span> <span class="kt">unit</span> <span class="o">=</span> <span class="bp">()</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>As noted above, ordinary users can't normally open devices directly:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
<span class="line-number">3</span>
<span class="line-number">4</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Context</span><span class="p">.</span><span class="nn">Path</span><span class="p">.</span><span class="n">add_device</span> <span class="n">ctx</span> <span class="s2">"/dev/input/event18"</span><span class="o">;;</span>
</span><span class="line"><span class="n">libinput</span> <span class="n">info</span><span class="o">:</span> <span class="n">event18</span><span class="o">:</span> <span class="n">opening</span> <span class="n">input</span> <span class="n">device</span> <span class="k">'</span><span class="o">/</span><span class="n">dev</span><span class="o">/</span><span class="n">input</span><span class="o">/</span><span class="n">event18'</span> <span class="n">failed</span> <span class="o">(</span><span class="nc">Permission</span> <span class="n">denied</span><span class="o">).</span>
</span><span class="line"><span class="n">libinput</span> <span class="n">info</span><span class="o">:</span> <span class="n">event18</span> <span class="o">-</span> <span class="n">failed</span> <span class="k">to</span> <span class="n">create</span> <span class="n">input</span> <span class="n">device</span> <span class="k">'</span><span class="o">/</span><span class="n">dev</span><span class="o">/</span><span class="n">input</span><span class="o">/</span><span class="n">event18'</span><span class="o">.</span>
</span><span class="line"><span class="o">-</span> <span class="o">:</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Device</span><span class="p">.</span><span class="n">t</span> <span class="n">option</span> <span class="o">=</span> <span class="nc">None</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>We could use DBus to get access, but for testing purposes this isn't very helpful.
<code>systemd-logind</code> requires us to call <code>TakeControl</code> before we can use <code>TakeDevice</code>,
and this fails because Sway is already using the devices.
We could run it from a text console,
but taking control also takes control of the screen and keyboard away from the console,
and then we can't see what we're doing!
It's actually pretty difficult to recover from this situation
(I managed it once using SysRq somehow, but the second time I had to power-off).</p>
<p>Instead, I suggest just making yourself the owner of the devices:</p>
<pre><code>$ sudo chown "$USER" /dev/input/event*
</code></pre>
<p>Then we can read events from them without interfering with Sway.
The owner will get set back to <code>root</code> next time <code>systemd-udevd</code> creates them
(e.g. after reboot or hotplug).</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
<span class="line-number">3</span>
<span class="line-number">4</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Context</span><span class="p">.</span><span class="nn">Path</span><span class="p">.</span><span class="n">add_device</span> <span class="n">ctx</span> <span class="s2">"/dev/input/event18"</span><span class="o">;;</span>
</span><span class="line"><span class="n">libinput</span> <span class="n">info</span><span class="o">:</span> <span class="n">event18</span> <span class="o">-</span> <span class="nc">Logitech</span> <span class="nc">USB</span> <span class="nc">Optical</span> <span class="nc">Mouse</span><span class="o">:</span> <span class="n">is</span> <span class="n">tagged</span> <span class="n">by</span> <span class="n">udev</span> <span class="k">as</span><span class="o">:</span> <span class="nc">Mouse</span>
</span><span class="line"><span class="n">libinput</span> <span class="n">info</span><span class="o">:</span> <span class="n">event18</span> <span class="o">-</span> <span class="nc">Logitech</span> <span class="nc">USB</span> <span class="nc">Optical</span> <span class="nc">Mouse</span><span class="o">:</span> <span class="n">device</span> <span class="n">is</span> <span class="n">a</span> <span class="n">pointer</span>
</span><span class="line"><span class="o">-</span> <span class="o">:</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Device</span><span class="p">.</span><span class="n">t</span> <span class="n">option</span> <span class="o">=</span> <span class="nc">Some</span> <span class="o">&lt;</span><span class="n">abstr</span><span class="o">&gt;</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>Calling <code>dispatch</code> causes libinput to read low-level events from Linux
and queue up its own high-level ones for us.
We can then pop them off the queue with <code>Event.get</code>:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
<span class="line-number">3</span>
<span class="line-number">4</span>
<span class="line-number">5</span>
<span class="line-number">6</span>
<span class="line-number">7</span>
<span class="line-number">8</span>
<span class="line-number">9</span>
<span class="line-number">10</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Context</span><span class="p">.</span><span class="n">dispatch</span> <span class="n">ctx</span><span class="o">;;</span>
</span><span class="line"><span class="o">-</span> <span class="o">:</span> <span class="kt">unit</span> <span class="o">=</span> <span class="bp">()</span>
</span><span class="line">
</span><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">get</span> <span class="n">ctx</span><span class="o">;;</span>
</span><span class="line"><span class="o">-</span> <span class="o">:</span> <span class="o">[</span> <span class="o">`</span><span class="nc">Unclassified</span> <span class="o">]</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">t</span> <span class="n">option</span> <span class="o">=</span>
</span><span class="line"><span class="nc">Some</span>
</span><span class="line"> <span class="o">{</span><span class="k">type</span> <span class="o">=</span> <span class="o">`</span><span class="nc">Device_added</span> <span class="o">_;</span>
</span><span class="line">  <span class="n">device</span> <span class="o">=</span> <span class="o">{</span><span class="n">sysname</span> <span class="o">=</span> <span class="s2">"event18"</span><span class="o">;</span> <span class="n">name</span> <span class="o">=</span> <span class="s2">"Logitech USB Optical Mouse"</span><span class="o">;</span>
</span><span class="line">            <span class="n">bustype</span> <span class="o">=</span> <span class="mh">0x3</span><span class="o">;</span> <span class="n">vendor</span> <span class="o">=</span> <span class="mh">0x46d</span><span class="o">;</span> <span class="n">product</span> <span class="o">=</span> <span class="mh">0xc077</span><span class="o">;</span> <span class="n">output</span> <span class="o">=</span> <span class="n">null</span><span class="o">;</span>
</span><span class="line">            <span class="n">seat</span> <span class="o">=</span> <span class="o">{</span><span class="n">physical_name</span> <span class="o">=</span> <span class="s2">"seat0"</span><span class="o">;</span> <span class="n">logical_name</span> <span class="o">=</span> <span class="s2">"default"</span><span class="o">}}]}</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>Once there are no more events in the queue, it returns <code>None</code>:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">get</span> <span class="n">ctx</span><span class="o">;;</span>
</span><span class="line"><span class="o">-</span> <span class="o">:</span> <span class="o">[</span> <span class="o">`</span><span class="nc">Unclassified</span> <span class="o">]</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">t</span> <span class="n">option</span> <span class="o">=</span> <span class="nc">None</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>After wiggling the mouse we can use <code>dispatch</code> again:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
<span class="line-number">3</span>
<span class="line-number">4</span>
<span class="line-number">5</span>
<span class="line-number">6</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Context</span><span class="p">.</span><span class="n">dispatch</span> <span class="n">ctx</span><span class="o">;;</span>
</span><span class="line"><span class="n">libinput</span> <span class="n">info</span><span class="o">:</span> <span class="n">event18</span> <span class="o">-</span> <span class="nc">Logitech</span> <span class="nc">USB</span> <span class="nc">Optical</span> <span class="nc">Mouse</span><span class="o">:</span> <span class="nc">SYN_DROPPED</span> <span class="n">event</span>
</span><span class="line">  <span class="o">-</span> <span class="n">some</span> <span class="n">input</span> <span class="n">events</span> <span class="n">have</span> <span class="n">been</span> <span class="n">lost</span><span class="o">.</span>
</span><span class="line">
</span><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="k">let</span> <span class="n">e</span> <span class="o">=</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">get</span> <span class="n">ctx</span><span class="o">;;</span>
</span><span class="line"><span class="k">val</span> <span class="n">e</span> <span class="o">:</span> <span class="o">[</span> <span class="o">`</span><span class="nc">Unclassified</span> <span class="o">]</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">t</span> <span class="n">option</span> <span class="o">=</span> <span class="nc">None</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>We have to call <code>dispatch</code> quickly after an event otherwise it gets dropped.
Also, libinput has various timers that need to be called promptly
(for example, it can discard very short clicks that are just the contacts bouncing after a real click).
So when calling <code>dispatch</code> manually, you'll probably see things like this:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
<span class="line-number">3</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Context</span><span class="p">.</span><span class="n">dispatch</span> <span class="n">ctx</span><span class="o">;;</span>
</span><span class="line"><span class="n">libinput</span> <span class="n">error</span><span class="o">:</span> <span class="n">client</span> <span class="n">bug</span><span class="o">:</span> <span class="n">timer</span> <span class="n">button</span><span class="o">-</span><span class="n">debounce</span><span class="o">-</span><span class="n">debounce</span><span class="o">-</span><span class="n">event18</span><span class="o">:</span>
</span><span class="line">  <span class="n">scheduled</span> <span class="n">expiry</span> <span class="n">is</span> <span class="k">in</span> <span class="n">the</span> <span class="n">past</span> <span class="o">(-</span><span class="mi">947</span><span class="n">ms</span><span class="o">),</span> <span class="n">your</span> <span class="n">system</span> <span class="n">is</span> <span class="n">too</span> <span class="n">slow</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>Still, you should be able to get some events by being reasonably quick at
pressing Return to run <code>dispatch</code> after using the mouse:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
<span class="line-number">3</span>
<span class="line-number">4</span>
<span class="line-number">5</span>
<span class="line-number">6</span>
<span class="line-number">7</span>
<span class="line-number">8</span>
<span class="line-number">9</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Context</span><span class="p">.</span><span class="n">dispatch</span> <span class="n">ctx</span><span class="o">;;</span>
</span><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="k">let</span> <span class="n">e</span> <span class="o">=</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">get</span> <span class="n">ctx</span><span class="o">;;</span>
</span><span class="line"><span class="k">val</span> <span class="n">e</span> <span class="o">:</span> <span class="o">[</span> <span class="o">`</span><span class="nc">Unclassified</span> <span class="o">]</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">t</span> <span class="n">option</span> <span class="o">=</span>
</span><span class="line">  <span class="nc">Some</span>
</span><span class="line">   <span class="o">{</span><span class="k">type</span> <span class="o">=</span> <span class="o">`</span><span class="nc">Pointer_button</span> <span class="o">{</span><span class="n">time</span> <span class="o">=</span> <span class="mi">62650</span><span class="o">.</span><span class="mi">052901</span><span class="o">;</span>
</span><span class="line">                            <span class="n">button</span> <span class="o">=</span> <span class="mi">272</span><span class="o">;</span>
</span><span class="line">                            <span class="n">state</span> <span class="o">=</span> <span class="o">`</span><span class="nc">Pressed</span><span class="o">;</span>
</span><span class="line">                            <span class="n">seat_key_count</span> <span class="o">=</span> <span class="mi">1</span><span class="o">};</span>
</span><span class="line">    <span class="n">device</span> <span class="o">=</span> <span class="o">{</span><span class="n">sysname</span> <span class="o">=</span> <span class="s2">"event18"</span><span class="o">;</span> <span class="n">name</span> <span class="o">=</span> <span class="s2">"Logitech USB Optical Mouse"</span><span class="o">;</span> <span class="o">...}]}</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>Events from <code>Event.get</code> are typed as <code>Unclassified</code>.
Although the REPL displays them as OCaml records, they're actually C structures
and need to be accessed via getter functions.
Calling <code>Event.get_type</code> returns the event as an OCaml variant:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
<span class="line-number">3</span>
<span class="line-number">4</span>
<span class="line-number">5</span>
<span class="line-number">6</span>
<span class="line-number">7</span>
<span class="line-number">8</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="k">let</span> <span class="n">ty</span> <span class="o">=</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">get_type</span> <span class="o">(</span><span class="nn">Option</span><span class="p">.</span><span class="n">get</span> <span class="n">e</span><span class="o">);;</span>
</span><span class="line"><span class="k">val</span> <span class="n">ty</span> <span class="o">:</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">ty</span> <span class="o">=</span>
</span><span class="line">  <span class="o">`</span><span class="nc">Pointer_button</span>
</span><span class="line">    <span class="o">{</span><span class="k">type</span> <span class="o">=</span> <span class="o">`</span><span class="nc">Pointer_button</span> <span class="o">{</span><span class="n">time</span> <span class="o">=</span> <span class="mi">62650</span><span class="o">.</span><span class="mi">052901</span><span class="o">;</span>
</span><span class="line">                             <span class="n">button</span> <span class="o">=</span> <span class="mi">272</span><span class="o">;</span>
</span><span class="line">                             <span class="n">state</span> <span class="o">=</span> <span class="o">`</span><span class="nc">Pressed</span><span class="o">;</span>
</span><span class="line">                             <span class="n">seat_key_count</span> <span class="o">=</span> <span class="mi">1</span><span class="o">};</span>
</span><span class="line">     <span class="n">device</span> <span class="o">=</span> <span class="o">{</span><span class="n">sysname</span> <span class="o">=</span> <span class="s2">"event18"</span><span class="o">;</span> <span class="n">name</span> <span class="o">=</span> <span class="s2">"Logitech USB Optical Mouse"</span><span class="o">;</span> <span class="o">...}]}</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>You can then match on it and get a typed C event:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="k">let</span> <span class="n">bev</span> <span class="o">=</span> <span class="k">match</span> <span class="n">ty</span> <span class="k">with</span> <span class="o">`</span><span class="nc">Pointer_button</span> <span class="n">e</span> <span class="o">-&gt;</span> <span class="n">e</span> <span class="o">|</span> <span class="o">_</span> <span class="o">-&gt;</span> <span class="k">assert</span> <span class="bp">false</span><span class="o">;;</span>
</span><span class="line"><span class="k">val</span> <span class="n">bev</span> <span class="o">:</span> <span class="o">[</span> <span class="o">`</span><span class="nc">Pointer_button</span> <span class="o">]</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="n">t</span> <span class="o">=</span> <span class="o">...</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>Now we know this is a button event, we can use button-event-specific functions, e.g.</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="n">utop</span> <span class="o">#</span> <span class="nn">Input</span><span class="p">.</span><span class="nn">Event</span><span class="p">.</span><span class="nn">Pointer</span><span class="p">.</span><span class="n">get_button_state</span> <span class="n">bev</span><span class="o">;;</span>
</span><span class="line"><span class="o">-</span> <span class="o">:</span> <span class="o">[</span> <span class="o">`</span><span class="nc">Pressed</span> <span class="o">|</span> <span class="o">`</span><span class="nc">Released</span> <span class="o">]</span> <span class="o">=</span> <span class="o">`</span><span class="nc">Pressed</span>
</span></code></pre></td></tr></tbody></table></div></figure><p><a href="https://github.com/talex5/libinput-ocaml/blob/main/examples/simple.ml">simple.ml</a> is an example program that prints out events in a loop:</p>
<pre><code>$ dune exec -- ./examples/simple.exe
Created libinput context
{type = `Device_added _;
 device = {sysname = "event2"; name = "Logitech USB Optical Mouse";
           bustype = 0; vendor = 0x46d; product = 0xc077; output = null;
           seat = {physical_name = "seat0"; logical_name = "default"}}]}
Waiting for events...
{type = `Pointer_scroll_wheel {time = 47360.187142; value120 = (0, -120)};
 device = {sysname = "event2"; name = "Logitech USB Optical Mouse"; ...}]}
...
</code></pre>
<p>There's lots more to the <a href="https://talex5.github.io/libinput-ocaml/local/libinput/Input/index.html">libinput API</a> (I probably should have looked at the length of the header file before deciding to make the OCaml bindings!). For example:</p>
<ul>
<li>Support for keyboards, switches, graphics tablets and touch screens (but not for joysticks, oddly).
</li>
<li>Checking what features devices provide.
</li>
<li>Configuration settings (e.g. left-handled mode, middle-button emulation, scroll direction).
</li>
<li>Controlling LEDs.
</li>
</ul>
<h3>Thoughts on the C bindings</h3>
<p>As with <a href="https://roscidus.com/blog/blog/2025/11/16/libdrm-ocaml/">libdrm-ocaml</a>, I used <a href="https://github.com/yallop/ocaml-ctypes">ctypes</a> to make the bindings and it worked well again.</p>
<h4>Ref-counting and garbage collection</h4>
<p>The main problem I had was with the lifecycle of resources,
due to libinput's unusual reference counting system.
In this system, e.g. <code>libinput_device_ref</code> increments the count for a device,
<code>libinput_device_unref</code> decrements it,
and libinput will free a resource when its count reaches zero.
This is all perfectly normal.
What is unusual about libinput is that it can also free resources when the ref-count is not zero.</p>
<p>Specifically, freeing a context immediately destroys everything owned by that context
(see <a href="https://bugs.freedesktop.org/show_bug.cgi?id=91872">Bug 91872</a>, resolved as <code>WONTFIX</code> to avoid breaking existing users).</p>
<p>I had a look at how <a href="https://python-libinput.readthedocs.io/en/latest/">python-libinput</a> handles this.
The docs say it "provides high-level object oriented api, taking care of
reference counting, memory management and the like automatically".
However, I didn't see any code to deal with this, and when I tested it Python just segfaulted.</p>
<p>I ended up with a somewhat complicated system where every reference can be invalidated or GC'd.
As the GC finaliser can run in any thread and libinput isn't thread-safe,
the finaliser pushes the reference onto a (lock-free) queue and
lets the main thread collect it on the next call to <code>dispatch</code>.
See <a href="https://github.com/talex5/libinput-ocaml/blob/76410a65999376c94cde7a9a225a4337623a4a3c/lib/context0.ml">context0.ml</a> for more details.</p>
<h4>Richer types</h4>
<p>The C code has lots of comments about event types, e.g.</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
<span class="line-number">3</span>
<span class="line-number">4</span>
<span class="line-number">5</span>
<span class="line-number">6</span>
<span class="line-number">7</span>
<span class="line-number">8</span>
<span class="line-number">9</span>
<span class="line-number">10</span>
<span class="line-number">11</span>
<span class="line-number">12</span>
<span class="line-number">13</span>
</pre></td><td class="code"><pre><code class="c"><span class="line"><span class="cm">/**</span>
</span><span class="line"><span class="cm"> [...]</span>
</span><span class="line"><span class="cm"> * For events not of type @ref LIBINPUT_EVENT_TOUCH_DOWN, @ref</span>
</span><span class="line"><span class="cm"> * LIBINPUT_EVENT_TOUCH_UP, @ref LIBINPUT_EVENT_TOUCH_MOTION or @ref</span>
</span><span class="line"><span class="cm"> * LIBINPUT_EVENT_TOUCH_CANCEL, this function returns 0.</span>
</span><span class="line"><span class="cm"> *</span>
</span><span class="line"><span class="cm"> * @note It is an application bug to call this function for events of type</span>
</span><span class="line"><span class="cm"> * other than @ref LIBINPUT_EVENT_TOUCH_DOWN, @ref LIBINPUT_EVENT_TOUCH_UP,</span>
</span><span class="line"><span class="cm"> * @ref LIBINPUT_EVENT_TOUCH_MOTION or @ref LIBINPUT_EVENT_TOUCH_CANCEL.</span>
</span><span class="line"><span class="cm"> [...]</span>
</span><span class="line"><span class="cm"> */</span>
</span><span class="line"><span class="kt">int32_t</span>
</span><span class="line"><span class="nf">libinput_event_touch_get_slot</span><span class="p">(</span><span class="k">struct</span><span class="w"> </span><span class="nc">libinput_event_touch</span><span class="w"> </span><span class="o">*</span><span class="n">event</span><span class="p">);</span>
</span></code></pre></td></tr></tbody></table></div></figure><p>OCaml doesn't need any of this because the type prevents it from being used incorrectly:</p>
<figure class="code"><div class="highlight"><table><tbody><tr><td class="gutter"><pre class="line-numbers"><span class="line-number">1</span>
<span class="line-number">2</span>
</pre></td><td class="code"><pre><code class="ocaml"><span class="line"><span class="k">val</span> <span class="n">get_slot</span> <span class="o">:</span>
</span><span class="line">  <span class="o">[&lt;</span> <span class="o">`</span><span class="nc">Touch_down</span> <span class="o">|</span> <span class="o">`</span><span class="nc">Touch_up</span> <span class="o">|</span> <span class="o">`</span><span class="nc">Touch_motion</span> <span class="o">|</span> <span class="o">`</span><span class="nc">Touch_cancel</span><span class="o">]</span> <span class="n">t</span> <span class="o">-&gt;</span> <span class="kt">int</span>
</span></code></pre></td></tr></tbody></table></div></figure><h4>Callbacks</h4>
<p>One annoying feature of libinput's design is that
it wants to invoke a user-provided function to open devices.
When using DBus, this means we have to run a nested event loop inside the callback handler,
and the application can't do anything else until it gets a reply.</p>
<p>libinput already has a queue of events for the application to process;
it would have been easier if it just added a "new device detected" event to the queue
and let the application open it in its own time and notify the library at the end.</p>
<h2>Lander game</h2>
<p>To test my new libinput bindings, I made a <a href="https://en.wikipedia.org/wiki/Zarch">Lander</a>-style game.
You can run it like this
(if not using Nix, consult the <a href="https://github.com/talex5/vulkan-test/blob/lander/README.md">README</a> for alternative instructions):</p>
<pre><code>git clone https://github.com/talex5/vulkan-test.git -b lander
cd vulkan-test
nix develop
dune exec -- ./src/main.exe
</code></pre>
<p><a href="https://roscidus.com/blog/images/lander/volcano.png"><span class="caption-wrapper center"><img src="https://roscidus.com/blog/images/lander/volcano.png" title="Try to find the landing pad" class="caption"><span class="caption-text">Try to find the landing pad</span></span></a></p>
<p>If run with <code>$WAYLAND_DISPLAY</code> set it will play in a Wayland window (and the compositor will use libinput),
while running it from a text console (e.g. with Ctrl-Alt-F2) will use libinput-ocaml.
It uses DBus to get access to the devices, if the login service is available.</p>
<p>The game starts with the ship flying across a randomly generated landscape,
and you must take control quickly or it will crash
(this is partly for dramatic purposes,
but also so that if my libinput code doesn't work for some reason then
the game will end after a few seconds and you'll get back control of your computer).</p>
<p>Move the mouse to angle the craft and use button-1 for thrust.
The aim of the game is to find the landing pad and land on it.
You can also steer the craft using a graphics tablet,
which works surprisingly well and also gives you pressure-sensitive thrust
(this only works in libinput-ocaml mode, although in Wayland mode your
compositor might have the tablet emulate a mouse, without pressure sensitivity).</p>
<p>You'll probably come across the pad just from flying around randomly for a minute or so,
but you could also read the <a href="https://github.com/talex5/vulkan-test/blob/0ac5db27e47d4f98cbbd269d24bf3540f64d78d5/src/map.ml#L35-L37">map generation code</a> for hints on how to find it more quickly.</p>
<p>To make the game, I started with the Viking room code from the Vulkan tutorial
(from my earlier <a href="https://roscidus.com/blog/blog/2025/09/20/ocaml-vulkan/">Vulkan graphics in OCaml vs C</a> post).
I <a href="https://github.com/talex5/vulkan-test/commit/a017bf7768e79ea5bbfc75a27dabcc98d1a295ee">replaced the room model with a simple spaceship model</a>
and <a href="https://github.com/talex5/vulkan-test/commit/f378569b80859965b88c7e96c81db0709ef0bbd6">added a scrolling landscape</a> below it,
then used libinput to <a href="https://github.com/talex5/vulkan-test/commit/37e19f1c6a5ab17c4ce4b15a8c8ab1d8f3094f41">add controls</a>.
The changes to <code>vt.ml</code> use libinput-ocaml,
while the <code>window.ml</code> changes are similar but for running under Wayland.</p>
<p>Some suggestions for possible extensions you might like to try:</p>
<ul>
<li>Allow using button-3 for half-thrust (very easy).
</li>
<li>Allow using Tab for thrust to make playing on a laptop's touchpad easier (easy for vt.ml, but you'll need to add Wayland keyboard support if you want Wayland to work too).
</li>
<li>Allow pausing the game by pressing <code>P</code>
(working out which key corresponds to which letter looks tricky in general and requires using a keymap,
but for getting it working on your own computer you can just see what number <code>P</code> produces).
</li>
<li>Add touch-screen support (I wanted to try this on my old PinePhone, but sadly its GPU doesn't support Vulkan).
</li>
<li>Hold down <code>M</code> to view a map (hint: the landscape module already has an overhead view for debugging).
</li>
</ul>
<h3>Non-libinput aspects of the game</h3>
<p>Making the game made me realise that I'd misunderstood something important about Vulkan.
I'd got the impression from the tutorial that
a Vulkan application has a single graphics pipeline that renders everything,
but in fact a "pipeline" is more like a drawing tool and typically you need several of them.
The lander game uses three: one to draw the landscape, one to draw the ship, and one for particles.</p>
<p>My previous code had e.g. <code>Pipeline</code> and <code>Input</code> modules,
but this structure doesn't make sense when you have multiple pipelines.
To minimise the diff, I first refactored the Viking room code,
splitting the <code>Pipeline</code> module into <code>Scene</code> (for the whole thing) and <code>Room</code> (the room-rendering pipeline).
After that, the Vulkan API started making a lot more sense to me!</p>
<p>I also discovered that my Viking room code didn't work on the Raspberry Pi 4 (for multiple reasons),
and spent quite a while getting it working there too.
Sadly, the lander game still doesn't run there as the Pi's graphics drivers don't fully support even Vulkan 1.1
(needed for <code>SV_VertexID</code>).</p>
<p>If you'd like to experiment with the game, here are some more ideas:</p>
<ul>
<li>Changing <code>gravity</code> and <code>engine_power</code> at the top of <a href="https://github.com/talex5/vulkan-test/blob/0ac5db27e47d4f98cbbd269d24bf3540f64d78d5/src/ship.ml">ship.ml</a> will change how hard the game is.
</li>
<li>Edit <a href="https://github.com/talex5/vulkan-test/blob/0ac5db27e47d4f98cbbd269d24bf3540f64d78d5/src/map.ml">map.ml</a> to make the terrain more interesting.
For example, you could manually add pillars or pyramids, or simulate coastal erosion to get cliffs, etc.
</li>
<li>I used <a href="https://github.com/talex5/vulkan-test/blob/0ac5db27e47d4f98cbbd269d24bf3540f64d78d5/src/perlin.ml">Perlin noise</a> to generate the landscape.
My code had various bugs while I was writing it, and most of them led to interesting effects!
</li>
<li>Add a radar display showing the map in the corner of the screen.
This should be pretty easy because the tile colours are already available as a texture,
so it should be just a case of displaying a textured rectangle.
</li>
</ul>
<p>Thanks to the OCaml Software Foundation for sponsoring this work.</p>

