---
title: "Incus in my home lab: why I dropped Proxmox (and then clustering)"
date: 2026-08-19
description: "A year of running Incus for LXC containers and VMs across amd64 and arm64 — what sold me, where clustering broke down, and what I'd tell myself starting out."
---

For most of my home-lab life the default answer was Proxmox. It's a good product. But over the last
year I moved my containers and VMs to **Incus**, and I want to write down why, including the one place
it didn't work out the way I hoped.

## Why Incus over Proxmox

Incus is what came out of LXD, kept alive by the original LXC team at Linux Containers. Same mental
model: one daemon (`incusd`) you talk to with a client (`incus`), running both system containers (LXC,
sharing the host kernel, very light) and virtual machines (QEMU/KVM, full guests, stronger isolation).
If you've touched LXD you already know most of it; the commands even match (`incus launch`, `incus
exec`, `incus config device add`).

What pulled me over was that Incus doesn't insist on being the whole OS. You drop the package onto the
distro you already run, run `incus admin init`, and you're set. Proxmox wants to own the machine. Incus
is happy being a layer on top of one. That difference matters more in practice than it sounds on paper.

## The Web UI caught me off guard

Proxmox ships a polished web UI and that's a real reason people like it for day-to-day clicking. Incus
doesn't include one by default, but the Incus Web UI (the `incus-ui-canonical` frontend) is genuinely
good once you switch it on:

```
incus config set core.https_address :8443
incus webui
```

Point a browser at the host on `:8443`, accept the cert, and you get a clean console for launching,
shelling into, snapshotting, and networking instances, VMs and LXCs alike. I do most routine management
there now instead of the CLI, which surprised me because I expected to live in the terminal. For a home
lab, that UI is the thing that makes Incus feel as turnkey as Proxmox.

## Running on two architectures

My lab isn't a single box. I've got amd64 and arm64 hosts, and the arm64 ones are quiet and cheap, so
they carry the always-on services. Incus handles both natively, and the image server tags architectures
cleanly (`images:debian/12/arm64`, `images:debian/12/amd64`), so picking the right image per host is
trivial. That mixed fleet is where Proxmox has historically lagged, and it's a real reason Incus fits
my setup better.

## Where it broke down: clustering across architectures

I started the way you'd expect: stand up an **Incus cluster**, join the nodes, get one unified view,
live-migrate VMs between hosts. The clustering story is arguably nicer than Proxmox's Corosync-based
one, with no hard node limit and the ability to rebalance VMs off a busy node.

Then I ran into the architecture wall, and the detail here changed between Incus versions so it's worth
getting right. Copying or moving an instance across architectures used to be refused outright, because
Incus did an architecture check on `incus copy`/`incus move`. That check was removed in **Incus 0.6**
(early 2024), so today you can copy an arm64 container onto an amd64 host and vice versa. What still
fails is *running* it there: a container's binaries are built for one ISA, so an amd64 container won't
start on an arm64 host, and `incus start` throws an architecture mismatch. VMs under KVM are the same
story, the guest OS has to target the host's architecture to boot, even though the migration itself
succeeds.

So the cluster gave me a tidy console and the ability to relocate instances across arch, which is
handy for storage and backups or for starting them later on a same-arch node. What it did not give me
was "run anything anywhere regardless of CPU." After a while the operational overhead, keeping members
healthy, wrangling certificates and trust, rebalancing that only helped VMs, stopped paying for itself
against that hard limit. If you're on an older Incus and hit `Requested architecture isn't supported
by this host` on a copy, that's the pre-0.6 behaviour; upgrade and the copy goes through, just don't
expect to start it on the wrong arch.

## Two more reasons it beat Proxmox

Both come from the same root: Incus is a daemon on a normal distro, not an OS you boot into.

It runs OCI images directly. Since **Incus 6.3** you can register Docker Hub as an OCI remote and
launch application containers natively:

```bash
incus remote add docker https://docker.io --protocol=oci
incus launch docker:nginx my-nginx
```

No Docker engine required, Incus runs the image itself. Be clear-eyed about it, though. Incus's main
model is the system container, a long-lived, VM-like full OS, while OCI/Docker images are application
containers, ephemeral and task-focused. The Incus team added OCI support but treat it as a secondary
path, not the headliner, and it's newer, so rough edges remain (GPU passthrough into OCI, local-image
import, that sort of thing). For a throwaway `nginx` or a quick experiment it's slick. For a stack of
interdependent services I still reach for Docker Compose.

It also sits alongside Docker and Podman on the same host without complaint. Because `incusd` is just
another service on the distro, I can `apt install docker.io` or `podman` next to it and run OCI
containers as a peer of Incus, not nested inside it. That's the clean way to do it: Incus owns the
system containers and VMs, Docker/Podman owns the ephemeral app workloads, both sharing the host kernel
through different mechanisms. What you don't want is the Docker engine running inside an unprivileged
Incus container, nested containers need privilege and cgroup fiddling and tend to surprise you on
network isolation. Keep them side by side on the host and they get along.

```bash
# On the Incus host, as a peer, not inside a container:
apt install -y docker.io
docker run -d -p 8080:80 nginx
```

## What I run today

I dropped clustering and now run each host as a standalone Incus server, split by architecture:

```
 amd64 host (incusd)          arm64 host (incusd)
 ┌────────────────────┐       ┌─────────────────────┐
 │ VMs (x86 guests)   │       │ lightweight LXCs    │
 │ heavier x86 svc    │       │ (DNS, Go services,  │
 │                    │       │  homelab odds/ends) │
 └─────────┬──────────┘       └───────────┬─────────┘
           │ incusbr0                     │ incusbr0
           │                              │
         [     trusted LAN 10.10.1.0/24     ]──[ switch ]──[ router ]
```

The amd64 host carries the heavier services and the VMs that need x86. The arm64 host carries
lightweight always-on LXC containers, DNS, small Go services, the odds and ends. I've made peace with a
workload living on one architecture and staying there. When I need it elsewhere I rebuild the image for
that arch, or run a multi-arch container image, rather than migrating. That's a small, predictable tax
instead of a cluster I maintain for a benefit I couldn't fully use.

In hindsight the lesson is simple: clustering is worth it when your nodes are the same architecture, or
when your VMs really are portable. With a mixed amd64 and arm64 fleet, running standalone per arch is
just simpler, and for a home lab it's honestly as effective.

## What I'd tell myself at the start

Unprivileged is the default for a reason. Flip `security.privileged=true` only when you have a concrete
need, like device passthrough for a Wi-Fi dongle (there's a whole post on that). Most workloads never
need it. And don't suffer the CLI out of pride, the Web UI is one command away.

Plan around architectures from day one. Pick an image tag per host, `/amd64` or `/arm64`, and decide
where a service lives before you build it. You can copy or move a container to a different-arch host
now that Incus 0.6 dropped that block, but you can't start it there, so design workloads per ISA
instead of promising yourself you'll migrate later. The real productivity win, though, is snapshots and
profiles: a decent default profile carrying storage, the bridge, and your SSH key turns a useful
container into a one-liner, and `incus snapshot` has bailed me out more than once.

## Bottom line

Incus replaced Proxmox in my home lab because it's lighter, distro-agnostic, and native on arm64, and
the Web UI closes the everyday UX gap. Clustering was a fun experiment I retired once I hit the
cross-architecture wall. If your nodes share an architecture, keep the cluster. If they don't, don't.
