---
title: OCaml-CI and native Windows builds
description: Following from post last week about obuilder and Windows Host Compute
  Services, I am pleased to report that this is now running on OCaml-CI. In this early
  phase, I have enabled testing only on Windows 2025 with OCaml 5.4 and opam 2.5 using
  the MinGW toolchain.
url: https://www.tunbury.org/2026/03/03/obuilder-hcs-2/
date: 2026-03-03T22:20:00-00:00
preview_image: https://www.tunbury.org/images/ocaml-logo.png
authors:
- Mark Elvers
source:
ignore:
---

<p>Following from <a href="https://www.tunbury.org/2026/02/19/obuilder-hcs/">post last week about obuilder and Windows Host Compute Services</a>, I am pleased to report that this is now running on OCaml-CI. In this early phase, I have enabled testing only on Windows 2025 with OCaml 5.4 and opam 2.5 using the MinGW toolchain.</p>

<p>Since my earlier post, I have achieved reliable operation and pushed the workarounds I had in obuilder into LWT. Furthermore, I have switched from a JSON configuration file per layer to an S-expression format, as this better matches the existing style, and the PPX deriving was already installed. There have also been numerous other small clean-ups.</p>

<p>Containerd uses the Windows Host Network Service, as does Docker. Docker creates a new network at boot with a random subnet. In the extract below, the network is 172.17.32.0/20.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>PS C:\Users\Administrator&gt; Get-HnsNetwork


ActivityId             : 0F32EF26-00D8-4B04-BB0A-57F18075F9EA
AdditionalParams       : 
CurrentEndpointCount   : 1
Extensions             : {@{Id=E7C3B2F0-F3C5-48DF-AF2B-10FED6D72E7A; IsEnabled=False; 
                         Name=Microsoft Windows Filtering Platform}, 
                         @{Id=F74F241B-440F-4433-BB28-00F89EAD20D8; IsEnabled=False; 
                         Name=Microsoft Azure VFP Switch Extension}, 
                         @{Id=430BDADD-BAB0-41AB-A369-94B67FA5BE0A; IsEnabled=True; Name=Microsoft 
                         NDIS Capture}}
Flags                  : 8
Health                 : @{LastErrorCode=0; LastUpdateTime=134170237512475197}
ID                     : 4EE1C263-FD69-45F9-8F4D-1D7137222B79
IPv6                   : False
LayeredOn              : FBA38879-AA6A-48AF-AD6D-35127F74313A
MacPools               : {@{EndMacAddress=00-15-5D-D2-1F-FF; StartMacAddress=00-15-5D-D2-10-00}}
MaxConcurrentEndpoints : 3
Name                   : nat
NatName                : NAT9A2D26A3-7226-46EE-9D96-5CDA0BF27595
Policies               : {@{Type=VLAN; VLAN=1}}
State                  : 1
Subnets                : {@{AdditionalParams=; AddressPrefix=172.17.32.0/20; Flags=0; 
                         GatewayAddress=172.17.32.1; Health=; 
                         ID=FD5E1DC1-71A1-4669-94D1-AD980E405535; IpSubnets=System.Object[]; 
                         ObjectType=5; Policies=System.Object[]; State=0}}
SwitchGuid             : 4EE1C263-FD69-45F9-8F4D-1D7137222B79
TotalEndpoints         : 13
Type                   : nat
Version                : 68719476736
Resources              : @{AdditionalParams=; AllocationOrder=2; Allocators=System.Object[];        
                         CompartmentOperationTime=0; Flags=0; Health=; 
                         ID=0F32EF26-00D8-4B04-BB0A-57F18075F9EA; PortOperationTime=0; State=1;     
                         SwitchOperationTime=0; VfpOperationTime=0;
                         parentId=95C9A579-958E-4991-A38A-A15BA23F39D9}
</code></pre></div></div>

<p>I had been running these commands on startup:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Get-HnsNetwork | Where-Object { $_.Name -eq 'nat' } | Remove-HnsNetwork
New-HnsNetwork -Type nat -Name nat -AddressPrefix '172.20.0.0/16' -Gateway '172.20.0.1'
</code></pre></div></div>

<p>And setting the network configuration to <code class="language-plaintext highlighter-rouge">172.20.0.0/16</code> in <code class="language-plaintext highlighter-rouge">C:\Program Files\containerd\cni\conf\0-containerd-nat.conf</code>. However, this broke <code class="language-plaintext highlighter-rouge">docker build</code> as it could not find the network it was expecting:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>failed to create endpoint vibrant_tu on network nat: failed during hnsCallRawResponse: hnsCall failed in Win32: Element not found. (0x490)
</code></pre></div></div>

<p>Changing direction, I have instead used <code class="language-plaintext highlighter-rouge">fix-nat.ps1</code> as a scheduled task at reboot to align containerd’s configuration with Docker’s.</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code># Read Docker's existing NAT network configuration and write it into the
# containerd CNI config so both runtimes share the same subnet.
Import-Module c:\windows\hns.psm1
$net = Get-HnsNetwork | Where-Object { $_.Name -eq 'nat' }
if (-not $net) {
    Write-Error "No NAT network found"
    exit 1
}
$subnet = $net.Subnets[0].AddressPrefix
$gateway = $net.Subnets[0].GatewayAddress
$json = @{
    cniVersion = "0.3.0"
    name = "nat"
    type = "nat"
    master = "Ethernet"
    ipam = @{
        subnet = $subnet
        routes = @(@{ gateway = $gateway })
    }
    capabilities = @{
        portMappings = $true
        dns = $true
    }
} | ConvertTo-Json -Depth 3
$json | Set-Content 'C:\Program Files\containerd\cni\conf\0-containerd-nat.conf' -Encoding ASCII
Write-Host "CNI config updated: subnet=$subnet gateway=$gateway"
</code></pre></div></div>

<p>Here is the log of a successful run from OCaml-CI: <a href="https://ocaml.ci.dev/github/mtelvers/mandelbrot/commit/14e08f30f087994a19822546a55405d078acd0d3/variant/windows-server-mingw-ltsc2025-5.4_opam-2.5">mtelvers/mandelbrot/commit/14e08f30f087994a19822546a55405d078acd0d3/variant/windows-server-mingw-ltsc2025-5.4_opam-2.5</a></p>

<p>PRs</p>

<ul>
  <li><a href="https://github.com/ocurrent/obuilder/pull/204">ocurrent/obuilder/pull/204</a></li>
  <li><a href="https://github.com/ocurrent/ocluster/pull/258">ocurrent/ocluster/pull/258</a></li>
  <li><a href="https://github.com/ocurrent/ocaml-ci/pull/1041">ocurrent/ocaml-ci/pull/1041</a></li>
  <li><a href="https://github.com/ocsigen/lwt/pull/1103">ocsigen/lwt/pull/1103</a></li>
</ul>
