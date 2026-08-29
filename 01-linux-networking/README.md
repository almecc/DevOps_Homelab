# Module 1 — Linux & Networking

Environment: Fedora 44 Workstation, single laptop, 16GB RAM, NVMe/btrfs root.

Goal: establish a working baseline of the host — services, processes, storage, networking, DNS, firewall, and remote access — before building on top of it with Podman, k3s, and the rest of the stack. This module is intentionally condensed: it documents real findings from the actual machine rather than walking through Linux fundamentals from scratch.

## System services & processes

```
systemctl list-units --type=service --state=running
systemctl status NetworkManager
journalctl -b -p err
```

Core services running as expected: `NetworkManager`, `firewalld`, `dbus-broker`, `chronyd`, `crond`, `bluetooth`, `cups`, `avahi-daemon`. `NetworkManager` active and healthy since boot.

Reviewed boot-time errors (`journalctl -b -p err`) — all firmware/hardware-level noise, not actionable: ACPI BIOS method aborts (common on consumer laptop firmware), `kvm_amd` no-op (this is an Intel CPU), and cosmetic PAM/Bluetooth chatter. No real system issues at boot.

```
ps aux --sort=-%mem
free -h
```

15.7GB total RAM, ~4GB free, ~7GB in reclaimable buff/cache — comfortable headroom before container workloads start. Baseline load average low on an 8-core system.

## Users, groups, permissions

User `archie` (uid 1000), member of `wheel` (sudo). Also a member of `vboxusers` — legacy group memberships from tools not used going forward in this homelab.

## Storage & filesystems

```
lsblk
df -h
cat /etc/fstab
```

Single NVMe drive, 476.9G, partitioned as:
- `nvme0n1p1` — 600M, `/boot/efi` (vfat)
- `nvme0n1p2` — 2G, `/boot` (ext4)
- `nvme0n1p3` — 474.4G, btrfs, subvolumes for `/` and `/home`

Root filesystem is **btrfs** with zstd compression (`compress=zstd:1`) — relevant later, since container storage drivers (overlayfs on top of btrfs) behave differently than on ext4, and k3s/Podman storage decisions should account for this. 16% used on root (74G/475G), plenty of headroom.

## IP addressing & routing

```
ip addr show
ip route show
nmcli device status
```

Single active interface `wlp0s20f3` (wifi), DHCP-assigned `10.186.x.x/24`, default route via `10.186.x.x`. Loopback and standard link-local IPv6 present. No unexpected interfaces or stale routes.

This is the host-level network baseline that Podman's bridge networking and k3s's CNI will sit on top of — container traffic will NAT out through this interface.

## DNS

```
cat /etc/resolv.conf
dig fedoraproject.org +short
nslookup github.com
```

DNS resolution goes through `systemd-resolved`'s local stub resolver (`127.0.0.53`), which forwards to upstream DNS. Both `dig` and `nslookup` resolved correctly. Container DNS will differ from this once Podman/k3s are running — containers get their own resolv.conf pointing at the container runtime's embedded DNS (Podman) or CoreDNS (k3s), not directly at this host resolver.

## Firewall (firewalld)

```
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

`firewalld` running, single active zone `FedoraWorkstation` bound to `wlp0s20f3`. Allowed services: `dhcpv6-client`, `samba-client`, `ssh`.

**Finding:** the zone also has `ports: 1025-65535/udp 1025-65535/tcp` open — a much broader range than a typical Workstation default. This is effectively every non-privileged port open inbound. Not an immediate problem on a home network, but it should be tightened before Podman starts publishing container ports, so that exposed ports are explicit and intentional rather than covered by a blanket range. Flagged as a to-do for Module 2.

## SSH

```
sudo systemctl enable --now sshd
systemctl status sshd
sudo grep -E "^(PasswordAuthentication|PermitRootLogin|Port)" /etc/ssh/sshd_config
```

`sshd` was installed but disabled by default — enabled and started today, now listening on port 22 (both IPv4 and IPv6). Config currently at Fedora defaults (no explicit overrides for `PasswordAuthentication`/`PermitRootLogin`/`Port` found in the grep, meaning they're at their compiled-in defaults). Key-based auth and a proper `sshd_config` hardening pass (disable password auth, disable root login) is a follow-up item, not done yet.

## Troubleshooting toolkit

```
ss -tulnp
curl -I https://github.com
ping -c 3 8.8.8.8
traceroute github.com
```

`ss -tulnp` baseline: expected listeners only — `sshd` on 22, `cups` on 631, `systemd-resolved` on 53 (stub resolver), and a `rygel` DLNA/UPnP media server on a few ports (desktop service, not relevant to homelab work). No unexpected open ports.

`curl -I https://github.com` returned `HTTP/2 200` — confirms outbound HTTPS working cleanly with valid TLS.

`traceroute github.com` completed the first two hops (local gateway, ISP hop) then stopped responding to ICMP past hop 3 — expected behavior, most ISPs/backbone routers rate-limit or block ICMP traceroute probes without indicating an actual connectivity problem. `curl` succeeding confirms the path works even though traceroute couldn't fully map it.

## Findings & decisions

- **Firewall port range too broad.** `1025-65535` open on both protocols — needs tightening once container ports need to be explicit (Module 2 follow-up).
- **SSH enabled but not yet hardened.** Running with defaults; key-based auth + config hardening is a follow-up, not blocking for now.
- **Root filesystem is btrfs, not ext4** — worth remembering when reasoning about container/k3s storage drivers later.

## Why this matters going forward

- `journalctl` triage by boot/unit/priority is the same workflow used to debug Podman and k3s service failures.
- Host networking (interface, routes, DNS resolver) is the substrate Podman bridge networking and k3s CNI build on top of — understanding it now makes container networking issues much easier to diagnose later.
- The firewall and SSH findings are concrete, real to-dos carried forward into Module 2, not hypothetical checklist items.
