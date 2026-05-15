---
title: Dynamic DNS with DHCP
description: DHCP-assigned addresses are very convenient, except when they change,
  and your DNS server becomes out of sync.
url: https://www.tunbury.org/2026/03/25/dhcp-dns/
date: 2026-03-25T18:40:00-00:00
preview_image: https://www.tunbury.org/images/isc-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>DHCP-assigned addresses are very convenient, except when they change, and your DNS server becomes out of sync.</p>

<p>This post consists of three parts. In the first part, I look at <a href="https://github.com/mirage/ocaml-dns">mirage/ocaml-dns</a> as a drop-in replacement for BIND. The second part covers the limitations of our Ubiquity Edge Router and the final part covers a BIND deployment.</p>

<h2>The Setup</h2>

<p>The router issues a DHCP address, and a separate machine acts as the authoritative DNS server. The goal is to automatically create forward (A) and reverse (PTR) DNS records when an IP address is allocated via DHCP.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Client --DHCPREQUEST--&gt; DHCP Server --DNS UPDATE (TSIG)--&gt; BIND Server
</code></pre></div></div>

<h2>Part 1: The OCaml Solution</h2>

<p>The <a href="https://github.com/mirage/ocaml-dns">ocaml-dns</a> library provides a complete DNS implementation in OCaml, including DNS UPDATE (RFC 2136) and TSIG authentication with HMAC-SHA256. There is an authoritative DNS server available <a href="https://github.com/roburio/dns-primary-git">dns-primary-git</a> which is built on the library, it accepts authenticated dynamic updates and persists zone changes to a git repository.</p>

<p>While <code class="language-plaintext highlighter-rouge">dns-primary-git</code> is designed as a MirageOS unikernel, it can also be compiled as a Unix application. Hannes Mehnert’s <a href="https://hannes.robur.coop/Posts/DnsServer">blog post</a> walks through the full setup: building the server, configuring zone files, deploying with Let’s Encrypt certificates, and running secondary servers. It reads standard zone files, serves authoritative DNS on UDP and TCP, and accepts TSIG-signed updates.</p>

<h3>TSIG Keys</h3>

<p>In <code class="language-plaintext highlighter-rouge">ocaml-dns</code>, the TSIG key name encodes the permissions. A key named <code class="language-plaintext highlighter-rouge">mykey._update.example.local</code> grants update access to the <code class="language-plaintext highlighter-rouge">example.local</code> zone. This differs from BIND, where key names are arbitrary identifiers and authorisation is configured separately with <code class="language-plaintext highlighter-rouge">allow-update</code>. The key is expressed as a DNSKEY record in the zone file, as below, where algorithm 163 is HMAC-SHA256.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>mykey._update.example.local. DNSKEY 0 3 163 K8aJhr8FpOhKGH0bP1dGa4VPCqmM3bRJf2TUGB1JyT0=
</code></pre></div></div>

<h3>ISC DHCP configuration</h3>

<p>The DHCP server needs to know:</p>

<ol>
  <li>The TSIG key</li>
  <li>Which DNS server to send updates to</li>
  <li>Which zone names to update</li>
  <li>What domain name to append to hostnames</li>
</ol>

<p>Here’s the relevant <code class="language-plaintext highlighter-rouge">dhcpd.conf</code> configuration:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>ddns-update-style interim;
ddns-domainname "example.local.";

key mykey._update.example.local {
    algorithm hmac-sha256;
    secret "K8aJhr8FpOhKGH0bP1dGa4VPCqmM3bRJf2TUGB1JyT0=";
};

key mykey._update.1.168.192.in-addr.arpa {
    algorithm hmac-sha256;
    secret "K8aJhr8FpOhKGH0bP1dGa4VPCqmM3bRJf2TUGB1JyT0=";
};

zone example.local. {
    primary 192.168.1.1;
    key mykey._update.example.local;
}

zone 1.168.192.in-addr.arpa. {
    primary 192.168.1.1;
    key mykey._update.1.168.192.in-addr.arpa;
}

subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.1;
    option domain-name "example.local";
    default-lease-time 7200;
    max-lease-time 7200;
}
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">ddns-update-style interim</code> tells ISC DHCP to use a TXT record alongside each A record to track the ownership of the DNS entry. This prevents one client from overwriting another client’s record.</p>

<p><code class="language-plaintext highlighter-rouge">ddns-domainname</code> is the domain appended to the client’s hostname. If a machine sends hostname <code class="language-plaintext highlighter-rouge">webserver</code>, the DHCP server registers <code class="language-plaintext highlighter-rouge">webserver.example.local</code>.</p>

<p>Each <code class="language-plaintext highlighter-rouge">zone</code> block references a key whose name matches the ocaml-dns convention: <code class="language-plaintext highlighter-rouge">name._update.zone</code>. The same secret is used for both, but the key names tell the DNS server which zone each key is authorised to update. The underscore in the key name requires ISC DHCP 4.4+ (see Part 2 for why this matters).</p>

<h3>Compatibility</h3>

<p>This approach works with ISC DHCP 4.4+, which supports HMAC-SHA256 TSIG and allows underscores in key names, or the newer ISC Kea, or any DNS UPDATE client that supports HMAC-SHA256.</p>

<p>It is a great option, giving you a single statically-linked binary that replaces BIND entirely. Zone changes are persisted to git, giving you version history for free. The TSIG key management is simple and self-contained. No configuration files beyond the zone files themselves.</p>

<p>If your DHCP server supports HMAC-SHA256, use this!</p>

<h2>Part 2: Ubiquiti EdgeRouter</h2>

<p>My Ubiquiti EdgeRouter runs ISC DHCP 4.1-ESV-R15-P1, which is pretty old. It has two problems that make it incompatible with <code class="language-plaintext highlighter-rouge">ocaml-dns</code>:</p>

<h3>Problem 1: Hardcoded HMAC-MD5</h3>

<p>ISC DHCP 4.1-ESV accepts <code class="language-plaintext highlighter-rouge">algorithm hmac-sha256</code> in the configuration file without error:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>key rndc-key {
    algorithm hmac-sha256;
    secret "K8aJhr8FpOhKGH0bP1dGa4VPCqmM3bRJf2TUGB1JyT0=";
};
</code></pre></div></div>

<p>But it always sends <code class="language-plaintext highlighter-rouge">HMAC-MD5.SIG-ALG.REG.INT</code> on the wire. The algorithm selection is silently ignored. This was confirmed by deploying an OCaml server and watching the actual packets arrive:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>error not implemented while w.x.y.z sent ...
... HMAC-MD5.SIG-ALG.REG.INT ...
</code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">ocaml-dns</code> library deliberately does not support HMAC-MD5, which is a sound security decision, but makes it incompatible with this DHCP server.</p>

<h3>Problem 2: No Underscores in Key Names</h3>

<p>The ocaml-dns convention encodes permissions in the key name: <code class="language-plaintext highlighter-rouge">mykey._update.zone</code>. But ISC DHCP 4.1-ESV’s parser treats the underscore as a token separator. This was fixed in later ISC DHCP versions (4.4+)</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>key rndc-key._update.example.local { algorithm hmac-sha256; ...
             ^
expecting left brace
</code></pre></div></div>

<h3>The Result</h3>

<p>The EdgeRouter’s DHCP server can only send DNS UPDATE packets signed with HMAC-MD5 using bare key names like <code class="language-plaintext highlighter-rouge">rndc-key</code>. <code class="language-plaintext highlighter-rouge">ocaml-dns</code> requires HMAC-SHA256 with convention key names like <code class="language-plaintext highlighter-rouge">rndc-key._update.zone</code>.</p>

<p>ISC DHCP reached end-of-life at the end of 2022. ISC recommends migrating to Kea, which would solve both problems. But until the EdgeRouter is replaced or the DHCP server upgraded, we need a DNS server that speaks MD5.</p>

<h2>Part 3: The BIND Fallback</h2>

<p>BIND handles HMAC-MD5, HMAC-SHA256, bare key names, and convention key names. It separates key identity from authorisation, so a key named <code class="language-plaintext highlighter-rouge">rndc-key</code> can be granted update access to any zone via <code class="language-plaintext highlighter-rouge">allow-update</code>.</p>

<h3>Generating the TSIG Key</h3>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code>rndc-confgen <span class="nt">-a</span> <span class="nt">-k</span> rndc-key <span class="nt">-A</span> hmac-md5
</code></pre></div></div>

<p>This writes <code class="language-plaintext highlighter-rouge">/etc/bind/rndc.key</code> using the HMAC-MD5 algorithm, matching what the Edge Router sends.</p>

<h3>Configuring BIND</h3>

<p>In <code class="language-plaintext highlighter-rouge">/etc/bind/named.conf.local</code>:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>include "/etc/bind/rndc.key";

zone "example.local" {
    type master;
    file "/var/lib/bind/db.example";
    allow-update { key rndc-key; };
    allow-transfer { none; };
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/var/lib/bind/db.192.168.1";
    allow-update { key rndc-key; };
    allow-transfer { none; };
};
</code></pre></div></div>

<p>Create minimal zone files, and DDNS will populate the rest.</p>

<p>Forward zone (<code class="language-plaintext highlighter-rouge">/var/lib/bind/db.example</code>):</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>$ORIGIN .
$TTL 604800
example.local    IN SOA  ns.example.local. admin.example.local. (
                         1          ; serial
                         604800     ; refresh (1 week)
                         86400      ; retry (1 day)
                         2419200    ; expire (4 weeks)
                         604800     ; minimum (1 week)
                         )
                 NS      ns.example.local.
$ORIGIN example.local.
ns               A       192.168.1.1
</code></pre></div></div>

<p>Reverse zone (<code class="language-plaintext highlighter-rouge">/var/lib/bind/db.192.168.1</code>):</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>$ORIGIN .
$TTL 604800
1.168.192.in-addr.arpa IN SOA ns.example.local. admin.example.local. (
                         1          ; serial
                         604800     ; refresh (1 week)
                         86400      ; retry (1 day)
                         2419200    ; expire (4 weeks)
                         604800     ; minimum (1 week)
                         )
                 NS      ns.example.local.
</code></pre></div></div>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nb">chown bind</span>:bind /var/lib/bind/db.example /var/lib/bind/db.192.168.1
</code></pre></div></div>

<p>Test with <code class="language-plaintext highlighter-rouge">nsupdate</code>:</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code>nsupdate <span class="nt">-k</span> /etc/bind/rndc.key <span class="o">&lt;&lt;</span><span class="no">EOF</span><span class="sh">
server 127.0.0.1
zone example.local
update add testhost.example.local. 3600 A 192.168.1.100
send
</span><span class="no">EOF
</span></code></pre></div></div>

<h3>Configuring the DHCP Server</h3>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>ddns-update-style interim;
ddns-domainname "example.local.";

key rndc-key {
    algorithm hmac-md5;
    secret "K8aJhr8FpOhKGH0bP1dGa4VPCqmM3bRJf2TUGB1JyT0=";
};

zone example.local. {
    primary 192.168.1.1;
    key rndc-key;
}

zone 1.168.192.in-addr.arpa. {
    primary 192.168.1.1;
    key rndc-key;
}

subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 192.168.1.1;
    option domain-name "example.local";
    default-lease-time 7200;
    max-lease-time 7200;
}
</code></pre></div></div>

<p>On an Ubiquiti EdgeRouter:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>set service dhcp-server dynamic-dns-update enable
set service dhcp-server global-parameters "key rndc-key { algorithm hmac-md5; secret &amp;quot;dTuFR8GiHHFgfam+yLkaWQ==&amp;quot;; };"
set service dhcp-server global-parameters "zone example.local. { primary 192.168.1.1; key rndc-key; }"
set service dhcp-server global-parameters "zone 1.168.192.in-addr.arpa { primary 192.168.1.1; key rndc-key; }"
set service dhcp-server global-parameters "ddns-domainname &amp;quot;example.local.&amp;quot;;"
</code></pre></div></div>

<h3>Client Configuration</h3>

<p>DHCP clients need to send their hostname. On Ubuntu/Debian with <code class="language-plaintext highlighter-rouge">systemd-networkd</code>:</p>

<div class="language-yaml highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="na">network</span><span class="pi">:</span>
  <span class="na">version</span><span class="pi">:</span> <span class="m">2</span>
  <span class="na">renderer</span><span class="pi">:</span> <span class="s">networkd</span>
  <span class="na">ethernets</span><span class="pi">:</span>
    <span class="na">eth0</span><span class="pi">:</span>
      <span class="na">dhcp4</span><span class="pi">:</span> <span class="s">yes</span>
</code></pre></div></div>

<h3>Delegating the Zone Publicly</h3>

<p>To make dynamic records resolvable from the internet, delegate from your public DNS provider:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>internal    NS    ns.internal.example.com.
ns.internal A     203.0.113.10
</code></pre></div></div>

<p>The second record is “glue” which is needed because the nameserver is inside the zone it serves. The full resolution can be verify with <code class="language-plaintext highlighter-rouge">dig myhost.internal.example.com +trace</code>.</p>
