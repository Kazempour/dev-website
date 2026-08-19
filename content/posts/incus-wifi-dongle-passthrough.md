---
title: "Passing a Wi-Fi dongle into an Incus LXC container"
date: 2026-08-18
description: "How to give a system container direct access to a physical Wi-Fi interface with Incus, and connect it to a network with wpa_supplicant."
---

I run a few Incus LXC containers on my home-lab hosts, and occasionally I want a container to
join a Wi-Fi network **directly** — not through the host bridge, but with its own physical radio.
That means handing the container the actual Wi-Fi device. Here's the procedure that works for me,
plus the caveats you should know before you do it.

## Why a physical device (and not a bridge)

Incus normally connects containers through a virtual bridge (`incusbr0`) that gets NAT'd to the
host's network. That's perfect for wired/LAN access. But if you specifically need the container to
associate with a Wi-Fi SSID as its own client, the container needs the real wireless interface —
bridges don't do Wi-Fi client auth. So we pass the dongle through as a **physical** NIC.

## The security trade-off (read this first)

To attach a physical device, the container must be **privileged** (`security.privileged=true`).
A privileged container shares the host's kernel namespaces far more loosely than the default
unprivileged one, which weakens isolation. For a trusted home-lab workload on hardware you own,
that's an acceptable risk — but don't do this for untrusted code. Incus runs **unprivileged by
default** for good reason; flip it only when you have a concrete reason (like this).

## Part 1: Host configuration

1. Plug the Wi-Fi USB dongle into the host.

2. Find the interface name:

   ```
   ip link show
   ```

   Look for a `wlan*` / `wl*` entry that appeared when you plugged it in.

3. Find the **physical device** the interface maps to (you'll need this for the passthrough):

   ```
   iw dev
   ```

   Note the `phy#` (e.g. `phy0`) and the interface name it's attached to.

4. Make the container privileged:

   ```
   incus config set <container_name> security.privileged=true
   ```

5. Attach the physical device to the container. Replace `phy0` with your actual `phy` index and
   `wlan0` with the host interface name:

   ```
   incus config device add <container_name> wlan0 nic nictype=physical parent=phy0 name=wlan0
   ```

   > The original notes used `parent=phy0 name=eth0`, but inside the container the device is a
   > Wi-Fi radio, not Ethernet — I name it `wlan0` to match what `wpa_supplicant` expects.

## Part 2: Get the container online once (temporary)

Before the dongle can authenticate to Wi-Fi, the container needs *some* internet to install the
supplicant tools. Borrow the host bridge temporarily:

1. Add a temporary bridge interface:

   ```
   incus config device add <container_name> tempnet nic network=incusbr0
   ```

2. Shell in:

   ```
   incus exec <container_name> bash
   ```

3. Pull an address on the temp interface (usually `eth0`):

   ```
   dhclient eth0
   ```

4. Install the Wi-Fi tools:

   ```
   apt update && apt install -y wpasupplicant wireless-tools iw
   ```

5. Exit, then remove the temporary bridge from the host:

   ```
   incus config device remove <container_name> tempnet
   ```

   (Reference: the [Linux Containers forum thread](https://discuss.linuxcontainers.org/t/lxc-container-on-same-network-as-host-with-internet-access/12038) on container networking.)

## Part 3: Connect to Wi-Fi

Shell back in and drive the dongle directly:

1. Generate the supplicant config (quote the SSID and password):

   ```
   wpa_passphrase "Your_WiFi_Name" "Your_WiFi_Password" > /etc/wpa_supplicant/wpa_supplicant.conf
   ```

2. Start `wpa_supplicant` on the passed-through interface in the background:

   ```
   wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
   ```

3. Clear any stale addresses and request a DHCP lease from your router:

   ```
   ip addr flush dev wlan0
   dhclient -v wlan0
   ```

4. Confirm you're associated:

   ```
   iw dev wlan0 link
   ```

## Part 4: Switching to another network

1. Regenerate the config for the new SSID:

   ```
   wpa_passphrase "New_WiFi_Name" "New_WiFi_Password" > /etc/wpa_supplicant/wpa_supplicant.conf
   ```

2. Stop and restart the supplicant:

   ```
   pkill wpa_supplicant
   wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
   ```

3. Re-lease the address:

   ```
   ip addr flush dev wlan0
   dhclient -r wlan0
   dhclient -v wlan0
   ```

## Gotchas

- The `parent=` value is the **phy** index from `iw dev` (`phy0`, `phy1`, …), not the `wlan`
  interface name. Getting this wrong is the #1 reason the device "doesn't appear" in the container.
- A privileged container with a physical device is effectively on the host's side for that radio.
  Keep it on trusted networks only.
- After a host reboot, re-run Part 3 (or wire it into the container's own init) — this setup isn't
  persistent across reboots by itself.
