---
title: "Passing a Wi-Fi dongle into an Incus LXC container"
date: 2026-08-19
description: "Why and how to give a system container its own physical Wi-Fi radio in Incus, and what it costs you in isolation."
---

Every now and then I want a container to be a real client on a Wi-Fi network, not just another
thing hidden behind the host's bridge. That means the container needs the actual wireless device,
not a virtual NIC with someone else's IP stacked on top. Incus can do this, but it's one of those
jobs where the "how" is short and the "why it works this way" is the part worth understanding.

## Why a bridge won't do it

Incus puts containers behind a virtual bridge (usually `incusbr0`) that NATs them onto the host's
network. Great for wired access. What it can't do is make the container *authenticate* to a Wi-Fi
access point as its own client, because Wi-Fi association is a property of the radio itself. A
bridge is layer 2 and 3; Wi-Fi client auth is layer 1 with a handshake. So if you want the container
to show up on the AP as its own device, you hand it the physical dongle and let it do the talking.

```
 HOST                                    CONTAINER (privileged)
 ┌───────────────────────────┐          ┌───────────────────────────┐
 │ USB Wi-Fi dongle          │          │ wlan0  (nictype=physical) │
 │   phy0  ── passthrough ────┼─────────►│   wpa_supplicant           │
 │                           │          │   dhclient → AP            │
 │ (no host Wi-Fi client)    │          │                           │
 └───────────────────────────┘          └─────────────┬─────────────┘
                                                     │
                                              [ Wi-Fi AP / router ]
```

Once passed through, the dongle vanishes from the host's network stack and reappears inside the
container as `wlan0`. The host can't use that radio while it's gone, which is exactly the point:
the container owns it.

## The trade-off you can't skip

Attaching a physical device requires the container to run **privileged**
(`security.privileged=true`). That loosens how much the container shares with the host's kernel
namespaces, and it visibly weakens the isolation that makes unprivileged containers worth using in
the first place. Incus runs unprivileged by default for a reason. I'm fine with this on a box I
own, running workloads I trust. I would not do it for anything I didn't. If the code inside that
container is untrusted, this is the wrong tool and you should stop here.

## What actually happens, step by step

The mechanics are simple once you see the shape of it. Plug the dongle in, then find what the
kernel called it:

```
ip link show        # look for a wlan*/wl* entry that appeared when you plugged it in
iw dev              # note the phy# (e.g. phy0) it's attached to
```

The thing people get wrong is the next line. You attach the device with `nictype=physical`, and
the `parent` is the **phy index** from `iw dev` (phy0, phy1, …), not the `wlan` name. Mix those up
and the device simply never shows up in the container, which is the most common way this whole
exercise fails:

```
incus config set <container> security.privileged=true
incus config device add <container> wlan0 nic nictype=physical parent=phy0 name=wlan0
```

Inside, the device is a Wi-Fi radio, so I name it `wlan0` to match what `wpa_supplicant` expects
rather than leaving it as a generic `eth0`.

There's a chicken-and-egg moment: before the dongle can authenticate, the container needs internet
to install the supplicant tools. I borrow the host bridge just long enough to get them, then pull it
back out:

```
incus config device add <container> tempnet nic network=incusbr0
incus exec <container> bash
  dhclient eth0
  apt update && apt install -y wpasupplicant wireless-tools iw
incus config device remove <container> tempnet
```

Then it's just standard Wi-Fi from inside. Write the network config, bring the supplicant up on the
passed-through interface, and ask the router for a lease:

```
wpa_passphrase "Your_WiFi_Name" "Your_WiFi_Password" > /etc/wpa_supplicant/wpa_supplicant.conf
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
ip addr flush dev wlan0
dhclient -v wlan0
iw dev wlan0 link      # confirm you're associated
```

Moving to another network later is the same dance with a fresh `wpa_passphrase` and a
`pkill wpa_supplicant` before you restart it.

## Things that will bite you

- `parent=` is the **phy** index, not the interface name. This is the number one reason the device
  "doesn't appear" in the container.
- A privileged container holding a physical device is, for that radio, effectively standing on the
  host's side of the network. Keep it on networks you trust.
- None of this survives a host reboot on its own. After a restart you re-run the supplicant steps,
  or you wire them into the container's own init if you want it to come back automatically.

It's a small amount of config for a genuinely useful capability: a container that is a first-class
Wi-Fi citizen instead of a guest behind the host. Just remember you paid for it with isolation, and
spend that privilege where it's actually safe.
