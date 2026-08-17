---
title: "Segmenting IoT from Trusted with Zero New Hardware"
date: 2026-08-07
description: "How I built a one-way trusted→IoT network using only the gear I already owned — a Calix ISP router, an OpenWrt box, and Tailscale."
draft: false
---

Like a lot of people, I accumulated smart gadgets faster than I built network hygiene around
them. Cameras, plugs, an ESPHome setup, a few "cloud-only" gizmos — all sitting on the same
flat network as my laptop, my phone, and the homelab. The moment one of those devices gets
compromised (and many ship with known CVEs), it's a straight shot to everything I care about.

The goal was simple to state, annoying to do:

> **Trusted devices can reach IoT devices. IoT devices cannot reach trusted devices.**

And I added a hard constraint: **use only the hardware I already own.** No new access points,
no paid firewall appliance, no subscription.

This post is the story of how I got there — including the dead ends, because the dead ends are
where the real learning lives.

## The starting point

My edge is a **Calix Gigaspire** supplied by the ISP. It gives me:

- A **trusted Wi-Fi** on `10.10.1.0/24`
- A **guest Wi-Fi** on `192.168.0.0/24` (isolated from trusted)
- No ability to add **static routes**
- The two LANs are **fully isolated** at L3
- The gateway IP (`10.10.1.1`) lives *inside* the trusted subnet

For the IoT side I had an **OpenWrt** box acting as the IoT gateway/AP: WAN on the guest LAN
(`192.168.0.x`), LAN on `172.16.1.0/24` with its own IoT Wi-Fi. My homelab runs on an
**Incus** host on the trusted network.

That's the whole inventory. The puzzle: make trusted reach IoT one-way, with this pile.

## Dead end #1: OPNsense as the trusted gateway

My first instinct was the textbook one: stand up **OPNsense** in a VM, make it the router
between trusted and IoT, and write a zone firewall — trusted → IoT allow, IoT → trusted deny.

It failed for three reasons that are worth internalizing:

1. **No static route on the ISP router.** Trusted devices use the Calix as their gateway. The
   Calix has no route to `172.16.1.0/24`, and I can't add one. So trusted traffic to IoT dies
   at the Calix.
2. **Single combined subnet.** The Calix gateway (`10.10.1.1`) is *inside* the trusted LAN. For
   OPNsense to be the gateway, its WAN and LAN would both need to be `10.10.1.0/24` — the same
   subnet on two interfaces, which OPNsense forbids.
3. **Kea DHCP won't serve a WAN interface.** Even when I forced the trusted network onto
   OPNsense's WAN, the DHCP server (Kea) refuses to run there.

**Lesson:** when the ISP router is the gateway and won't do static routes, you cannot make a
downstream box the trusted gateway without fighting the subnet topology the whole way.

## The pivot: stop routing, start tunneling

The realization that broke the logjam: I didn't need to *route* trusted→IoT at L3. I needed
trusted devices to *reach* IoT. A tunnel does that without touching the Calix's routing table.

- **Raw WireGuard** — works, but I'd be hand-managing keys per device, and off-LAN access means
  port forwarding.
- **Tailscale** — WireGuard under the hood, but with automatic key exchange, NAT traversal, and
  magic DNS.

Tailscale won on convenience. And then the elegant part: **OpenWrt can run Tailscale.** So I
turned the OpenWrt IoT box into a **Tailscale subnet router** for `172.16.1.0/24`.

## The architecture that worked

<style>
.net-diagram { max-width:100%; overflow-x:auto; margin:1.5rem 0;
  font-family:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;
  font-size:0.85rem; line-height:1.5; color:#ddd; }
.net-diagram .box { border:1px solid #555; border-radius:6px; padding:0.5rem 0.75rem; background:#2b2b2b; }
.net-diagram .box-guest { border:1px dashed #777; background:#262626; }
.net-diagram .box-iot { border:1px solid #6a8; background:#1f2b22; }
.net-diagram .row { display:flex; gap:1rem; }
.net-diagram .row > div { flex:1; }
.net-diagram .conn { text-align:center; color:#888; margin:0.5rem 0; }
.net-diagram .muted { color:#bbb; }
@media (max-width:520px) {
  .net-diagram .row { flex-direction:column; }
}
</style>

<div class="net-diagram">

  <div class="box" style="text-align:center;"><strong>INTERNET</strong></div>
  <div class="conn">│</div>

  <div class="box">
    <strong>ISP Router: Calix Gigaspire (EXOS)</strong><br>
    <span class="muted">DHCP: trusted OFF · guest ON</span>
  </div>

  <div class="row" style="margin-top:0.5rem;">
    <div class="box">
      <strong>TRUSTED Wi-Fi</strong> · 10.10.1.0/24<br>
      <span class="muted">Trusted devices (laptop/phone)<br>+ Tailscale client</span>
    </div>
    <div class="box box-guest">
      <strong>GUEST Wi-Fi</strong> (isolated) · 192.168.0.0/24<br>
      <span class="muted">OpenWrt WAN side</span>
    </div>
  </div>

  <div class="conn">│ (Tailscale tunnel → 172.16.1.0/24)</div>

  <div class="box box-iot" style="margin-top:0.5rem;">
    <strong>OpenWrt — IoT GW/AP + Tailscale subnet router</strong><br>
    <span class="muted">WAN = 192.168.0.x (guest) · LAN = 172.16.1.1 (IoT Wi-Fi)<br>Tailscale advertises 172.16.1.0/24</span>
  </div>

  <div class="conn">│ IOT Wi-Fi SSID</div>

  <div class="box">
    <strong>IoT devices</strong> · 172.16.1.0/24
  </div>

</div>

> On narrow screens the two side-by-side boxes (Trusted / Guest) stack vertically, and the whole
> diagram scrolls horizontally if it still doesn't fit.

Trusted devices join my Tailscale tailnet and get a route to `172.16.1.0/24` through the
OpenWrt subnet router. IoT devices stay behind OpenWrt, which is on the isolated guest LAN.

## How the one-way rule is actually enforced

This is the subtle part. There are **two** control points, and both matter:

| Flow | Allowed? | Enforced by |
|------|----------|-------------|
| Trusted → IoT | ✅ | Tailscale ACL (`tag:trusted` → `172.16.1.0/24`) + OpenWrt `tailscale → lan` forward |
| IoT → Trusted | ❌ | No route + Tailscale default-deny |
| IoT → Internet | ✅ | OpenWrt `lan → wan` + masquerade |
| IoT → Guest | ❌ | OpenWrt explicit REJECT |
| Guest → OpenWrt mgmt | ❌ | OpenWrt WAN input REJECT + port block |

The key insight: **IoT → trusted is blocked by the network topology already** (IoT is on the
isolated guest LAN behind OpenWrt, with no route to `10.10.1.0/24`). Tailscale ACL is the
second layer. Defense in depth, not a single point.

## Restrictions and limitations

This design is "good enough for home," not "enterprise." Be honest about the trade-offs:

1. **Third-party metadata.** Tailscale sees your node graph and relay metadata. Traffic itself is
   end-to-end WireGuard. If you're privacy-maximal, raw WireGuard is the alternative.
2. **Control-plane dependency.** If Tailscale's coordination server is down, *new* connections
   can't be negotiated. Existing tunnels keep working.
3. **OpenWrt storage.** The Tailscale package needs enough flash/RAM. On a tiny router it may not
   fit — run the subnet router as a container on the homelab host instead.
4. **No L2 segmentation.** This is L3 + tunnel based. Enable client isolation on the IoT SSID for
   true device isolation.
5. **mDNS/discovery doesn't cross.** HomeKit/AirPlay/Chromecast from trusted → IoT won't work
   without an mDNS reflector.
6. **The Calix is still the perimeter.** Its guest/trusted isolation is doing real work. Verify it
   periodically.

## Lessons learned

- **Map the ISP router's actual capabilities first.** "Can it do static routes? Can I disable DHCP
  per-LAN? Are the LANs isolated?" These three questions decided the entire architecture.
- **A downstream firewall can't fix an upstream routing limitation.**
- **Tunneling beats routing when you can't edit the router.**
- **Put the tunnel endpoint where the subnet already lives.**
- **Default-deny is your friend.**
- **Don't disable security features to make a design work.**

## What I'd do differently

If I were building from scratch, I'd buy a VLAN-capable router (UniFi/Omada/OPNsense on a mini-PC)
and do this with real VLANs + a zone firewall. But given "use what I have," the
Tailscale-on-OpenWrt approach got me the security property I wanted with zero new hardware and an
afternoon of tinkering.

The headline takeaway: **you probably don't need a fancy firewall to segment IoT; you need to
understand what your ISP router will and won't let you do, then tunnel around the parts it won't.**
