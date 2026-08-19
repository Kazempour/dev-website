---
title: "Incus in my home lab: why I dropped Proxmox (and then clustering)"
date: 2026-08-18
description: "A year of running Incus for LXC containers and VMs across amd64 and arm64 — what sold me, where clustering broke down, and what I'd tell myself starting out."
---

For most of my home-lab life the default answer was Proxmox. It's a great product. But over the
last year I moved my containers and VMs to **Incus**, and I want to write down why — and the one
place it didn't work out the way I hoped.

## What Incus actually is

Incus is the successor to LXD, maintained by the original LXC team at Linux Containers. Same
mental model: a single daemon (`incusd`) you talk to with a client (`incus`), managing both **system
containers** (LXC — share the host kernel, very light) and **virtual machines** (QEMU/KVM — full
guest, stronger isolation). If you've used LXD, you already know 90% of Incus; the commands even
mirror (`incus launch`, `incus exec`, `incus config device add`).

The thing that pulled me in: Incus doesn't drag a whole custom Debian spin with it. You install the
package on the distro you already run, `incus admin init`, and you're done. Proxmox wants to be the
OS; Incus is happy being a layer on top of one.

## The Web UI is genuinely good

Proxmox ships a polished web UI out of the box and that's a real advantage for day-to-day clicking.
Incus doesn't ship one by default, but the **Incus Web UI** (the `incus-ui-canonical` frontend) is
excellent once enabled:

```
incus config set core.https_address :8443
incus webui
```

Point a browser at the host on `:8443`, accept the cert, and you get a clean console for launching,
console-ing into, snapshotting, and networking instances — VMs and LXCs alike. I do most of my
routine management there now rather than the CLI, which surprised me because I expected to live in
the terminal. For a home lab, that UI is the feature that makes Incus feel as turnkey as Proxmox.

## Running on two architectures

My lab isn't one box — I've got **amd64** and **arm64** hosts (the arm64 boxes are quiet and cheap,
great for always-on services). Incus handles both natively, and the image server tags architectures
cleanly (`images:debian/12/arm64`, `images:debian/12/amd64`), so launching the right image per host
is trivial. That heterogeneous setup is where Proxmox historically lags (its arm64 support is far
behind x86), and it's a genuine reason Incus fits my fleet better.

## Where it broke down: clustering across architectures

I started with **Incus clustering** — join the nodes, get a unified view, live-migrate VMs between
hosts. The clustering story is arguably nicer than Proxmox's Corosync-based one: no hard node limit,
and it can auto-rebalance VMs off a busy node.

But the promise of "one pool, move anything anywhere" falls apart at the architecture boundary.
**Containers don't migrate cleanly between amd64 and arm64.** Incus will refuse a copy/move across
architectures because the binaries inside the container are built for one ISA. I could move an
arm64 container to another arm64 node, and an amd64 container to another amd64 node — but never
across. Live VM migration has the same wall unless the guest images are multi-arch (they usually
aren't).

So the cluster gave me a pretty unified console and VM mobility *within* an architecture, but the
cross-arch flexibility I'd imagined wasn't real. After a while the cluster's operational overhead
(keeping members healthy, certificate/trust wrangling) wasn't paying for itself given that
constraint.

## What I run today: standalone, per architecture

I dropped clustering and now run each host as a **standalone** Incus server. Split by architecture:

- **amd64 host** — heavier services, VMs that need x86.
- **arm64 host** — lightweight always-on LXC containers (DNS, small Go services, the homelab
  odds and ends).

I accept that a workload lives on one architecture and stays there. When I need it elsewhere, I
rebuild the image for that arch (or run a multi-arch container image) rather than migrating. That's
a small, predictable tax instead of a cluster to maintain for a benefit I couldn't fully use.

The lesson, in hindsight: clustering is worth it *if your nodes are homogeneous* (or your VMs are
truly portable). With a mixed amd64/arm64 fleet, standalone-per-arch is simpler and honestly just
as effective for a home lab.

## Things I'd tell myself at the start

- **Unprivileged is the default for a reason.** Only flip `security.privileged=true` when you have a
  concrete need (device passthrough, like a Wi-Fi dongle — see the [passthrough post](/posts/incus-wifi-dongle-passthrough/)).
  Most workloads never need it.
- **The Web UI is one command away** — don't suffer the CLI just because you assume Incus is
  headless.
- **Think in architectures.** Pick an image tag per host (`/amd64`, `/arm64`) and decide where a
  service lives up front; don't plan to migrate it across ISAs.
- **Snapshots and profiles are the real productivity win.** A good default profile (storage, the
  bridge, user SSH key) makes spinning up a useful container a one-liner, and `incus snapshot` has
  saved me more than once.

## Bottom line

Incus replaced Proxmox in my home lab because it's lighter, distro-agnostic, native on arm64, and
the Web UI closes the day-to-day UX gap. Clustering was a fun experiment that I retired once I hit
the cross-architecture wall — standalone per-arch is the setup that actually fits a mixed fleet.
If your nodes are same-arch, keep the cluster; if they're not, don't.
