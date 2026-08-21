---
title: "Segmenting IoT from Trusted with Zero New Hardware"
date: 2026-08-07
description: "How I built a one-way trusted→IoT network using only the gear I already owned — a Calix ISP router, an OpenWrt box, and Tailscale."
draft: false
---

Like a lot of people, I collected smart gadgets faster than I built any hygiene around them. Cameras,
plugs, an ESPHome setup, a couple of cloud-only gizmos, all sitting on the same flat network as my
laptop, my phone, and the homelab. The day one of those ships with a known CVE and gets popped, it's
a straight line into everything I actually care about.

So I wanted one rule, stated simply and annoying to enforce:

> Trusted devices can reach IoT devices. IoT devices cannot reach trusted devices.

And I added a constraint that made it interesting: use only the hardware I already own. No new access
point, no paid firewall, no subscription. This is the story of how I got there, including the dead ends,
because the dead ends are where I actually learned something.

## The starting point

My edge is a Calix Gigaspire from the ISP. It gives me a trusted Wi-Fi on `10.10.1.0/24` and a guest
Wi-Fi on `192.168.0.0/24`, isolated from trusted. What it does not give me is the ability to add static
routes. The two LANs are fully isolated at layer 3, and the gateway (`10.10.1.1`) lives inside the
trusted subnet.

On the IoT side I had an OpenWrt box acting as the IoT gateway and AP: its WAN on the guest LAN
(`192.168.0.x`), its LAN on `172.16.1.0/24` with its own IoT Wi-Fi. The homelab runs on an Incus host
on the trusted network.

That's the whole inventory. The puzzle was to make trusted reach IoT one way, with exactly that pile.

## Dead end: OPNsense as the trusted gateway

My first instinct was the textbook one. Stand up OPNsense in a VM, make it the router between trusted
and IoT, write a zone firewall: trusted to IoT allow, IoT to trusted deny.

It failed, and the reasons are worth sitting with:

1. The ISP router won't take a static route. Trusted devices use the Calix as their gateway, and the
   Calix has no route to `172.16.1.0/24`, nor will it accept one. So trusted traffic to IoT dies at the
   Calix.
2. The subnet topology fights you. The Calix gateway (`10.10.1.1`) sits *inside* the trusted LAN. For
   OPNsense to be the gateway, its WAN and LAN would both have to be `10.10.1.0/24`, the same subnet on
   two interfaces, which OPNsense simply won't allow.
3. Kea DHCP won't serve a WAN interface. Even when I forced the trusted network onto OPNsense's WAN, the
   DHCP server refused to run there.

The lesson stuck with me: when the ISP router is the gateway and won't do static routes, you can't make
a downstream box the trusted gateway without fighting the subnet topology the whole way down.

## The pivot: stop routing, start tunneling

The thing that broke the logjam was realizing I didn't need to *route* trusted to IoT at layer 3. I
needed trusted devices to *reach* IoT. A tunnel does that without touching the Calix's routing table.

Raw WireGuard would work, but I'd be hand-managing keys per device, and anything off-LAN means port
forwarding. Tailscale is WireGuard underneath with automatic key exchange, NAT traversal, and magic DNS,
so it won. Then the elegant bit: OpenWrt can run Tailscale. I turned the OpenWrt IoT box into a Tailscale
subnet router for `172.16.1.0/24`.

## The architecture that worked

```
                              INTERNET
                                 │
                    ┌────────────┴─────────────┐
                    │   ISP Router: Calix      │
                    │   Gigaspire (EXOS)       │
                    │  (DHCP: trusted OFF,     │
                    │   guest ON)              │
                    └──────────────────────────┘
                       │                │
         TRUSTED Wi-Fi │                │ GUEST Wi-Fi (isolated)
         10.10.1.0/24  │                │ 192.168.0.0/24
                       │                │
          ┌────────────┴───┐            │
          │ Trusted devices│            │
          │ (laptop/phone) │            │
          │ + Tailscale    │            │
          │   client       │            │
          └────────────────┘            │
                  │ (Tailscale tunnel)  │
                  │  to 172.16.1.0/24   │
                  ▼                     │
          ┌─────────────────────────────┴──────────┐
          │ OpenWrt (IoT GW/AP + Tailscale subnet   │
          │          router)                        │
          │  WAN  = 192.168.0.x  (guest side)       │
          │  LAN  = 172.16.1.1 (IoT Wi-Fi)          │
          │  Tailscale advertises 172.16.1.0/24     │
          └─────────────────────────────────────────┘
                       │
                 IOT Wi-Fi SSID
                       │
              ┌────────┴─────────┐
              │ IoT devices      │
              │ 172.16.1.0/24    │
              └──────────────────┘
```

Trusted devices join my Tailscale tailnet and get a route to `172.16.1.0/24` through the OpenWrt subnet
router. IoT devices stay behind OpenWrt, which itself sits on the isolated guest LAN.

## How the one-way rule actually holds

This is the subtle part, and it's easy to get wrong by assuming one control point does all the work.
There are two, and both matter:

| Flow | Allowed? | Enforced by |
|------|----------|-------------|
| Trusted → IoT | yes | Tailscale ACL (`tag:trusted` → `172.16.1.0/24`) + OpenWrt `tailscale → lan` forward |
| IoT → Trusted | no | No route + Tailscale default-deny |
| IoT → Internet | yes | OpenWrt `lan → wan` + masquerade |
| IoT → Guest | no | OpenWrt explicit REJECT |
| Guest → OpenWrt mgmt | no | OpenWrt WAN input REJECT + port block |

The real insight is that IoT to trusted is already blocked by the topology (IoT lives on the isolated
guest LAN behind OpenWrt, with no route to `10.10.1.0/24`). The Tailscale ACL is a second layer.
Defense in depth, not a single gate.

## What it costs you

This is good enough for home, not enterprise, and I'll be honest about the trade-offs:

1. Third-party metadata. Tailscale sees your node graph and relay metadata. The traffic itself is
   end-to-end WireGuard. If that bugs you, raw WireGuard is the alternative.
2. Control-plane dependency. If Tailscale's coordination server is down, new connections can't be
   negotiated. Tunnels already up keep running.
3. OpenWrt storage. The Tailscale package needs flash and RAM. On a tiny router it may not fit, in
   which case run the subnet router as a container on the homelab host instead.
4. No layer 2 segmentation. This is layer 3 plus a tunnel. Turn on client isolation on the IoT SSID
   for real device-to-device isolation.
5. mDNS doesn't cross. HomeKit, AirPlay, Chromecast from trusted to IoT won't work without an mDNS
   reflector.
6. The Calix is still the perimeter. Its guest/trusted isolation is doing real work. Verify it every
   so often.

## What I'd do differently

If I were building from scratch I'd buy a VLAN-capable router, UniFi or Omada or OPNsense on a mini-PC,
and do this with real VLANs and a zone firewall. But given the "use what I have" constraint, the
Tailscale-on-OpenWrt approach got me the security property I wanted with zero new hardware and an
afternoon of tinkering.

The takeaway is less "buy a fancy firewall" and more "know what your ISP router will and won't let you
do, then tunnel around the parts it won't."
