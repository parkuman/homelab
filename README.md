# HomeLab

This repository contains all of the code and documentation for my homelab.

I started this project in January, 2025. It currently runs on a used Acer Chromebox CXI4 I picked up off of Facebook Marketplace.

## Goals

- use this as a way to learn about dev ops, self-hosting, networking
- observability via Grafana
- everything is managed via Gitops
- self host apps I use every day
- my website
- [AnyType](https://anytype.io/) - for note taking
- [Jellyfin](https://jellyfin.org/) - media streaming
- PiHole / AdGuard or similar
- Home Assistant

## Setup

- Proxmox
  - virtualize the one server i have into multiple machines to simulate a K3S cluster
  - some apps might be better suited as a VM (home assistant) - proxmox helps with this
- K3S Cluster
  - 2 debian nodes
    - 1 control plane node
    - 1 worker node (for now, plan to extend this via more VMs or more physical servers later if needed)

## Sops

Using [`sops`](https://github.com/getsops/sops) for secret management.

```bash
sops -e -i apps/<blah>/secret.yaml
```

## Repo structure

The current repo structure is close to the one outlined in the [Flux Documentation](https://fluxcd.io/flux/guides/repository-structure/), but more suited to a simple homelab setup.

## Plan moving forward

### TODO

- [ ] merge this repo with `.dotfiles`
- [ ] install nix on NAS
- [ ] get ssh working on NAS
- [ ] get ZFS working
- [ ] get samba working
- [ ] set up users for samba / unix so jill and i have one
- [ ] get DNS working .. ?
- [ ] get it working over tailscale
- [ ] buddy backup
- [ ] install immich
- [ ] immich cloudflare tunnel for sharing

...

### Hardware

- **Chromebox** — i5-10210U, 32 GB RAM. Currently runs Proxmox + K3s. Home Assistant as a VM.
- **Zimaboard 2 "NAS"** — 8 GB RAM, 6 TB HDD (data), 500 GB SSD (boot). Primary storage and core infrastructure.
- **Zimaboard 2 "Play"** — 8 GB RAM. Sandbox for experiments and game servers.

---

### Machine Roles

#### Zimaboard NAS (NixOS)

The foundation. If this is up, the home network works.

**Core infrastructure:**

- AdGuard Home. DNS + ad-blocking for the entire network.
- Caddy. reverse proxy with automatic TLS via Cloudflare DNS-01 challenge. Terminates `*.home.prowe.ca` for NAS-hosted services.
- Samba. SMB file shares from ZFS datasets, accessible from all PCs on LAN and over Tailscale.
- Tailscale. mesh VPN node.

**Storage + media:**

- ZFS. single 6 TB disk (no redundancy!).
- Immich. photo backup and management. Shared links with upload for friends (no account needed).
- Jellyfin. media server. N150 iGPU handles transcoding.
- Some way to access these files from my phone for me and for jill
  - nextcloud?
  - syncthing?
- NFS. exports for K3s if any workload needs NAS storage.

**Dev:**

- Forgejo. git forge. Repos live close to storage rather than over network
  - could also consider running this on the chromebox

**Networking:**

- cloudflared. Cloudflare Tunnel for public access to Immich (and maybe jellyfin?) at `photos.prowe.ca`. Outbound-only, no ports opened, home IP hidden. Use this mainly for sharing links to albums for friends and family to see and add photos.

**Management:**

- Node exporter. feeds metrics to Prometheus on the Chromebox.

**Backup:**

- restic. nightly encrypted backups to dad's QNAP over Tailscale.
- ZFS snapshots. local rollback capability.

#### Chromebox (NixOS host, Proxmox or libvirt for VMs)

Non-critical services and learning. If K3s breaks ideally nobody other than me notices.

**Home automation:**

- Home Assistant. runs as a dedicated VM (HAOS) as per their recommendation. No mission critical shit on here

**K8s playground:**

- Pi agent. runs headless as a daemon. Accessible via `pi.home.prowe.ca`.
- K3s with Flux GitOps. Learning/professional development environment.
- Traefik. ingress for K8s services.
- MetalLB. LoadBalancer IPs for K8s services.

**Apps (non-critical, in K8s):**

- Mealie
- Obsidian LiveSync (CouchDB)
- Grafana + Prometheus + Loki (monitoring for all machines)
- Forgejo runners (resource-hungry, 32 GB absorbs spikes without threatening NAS stability)

**Management:**

- Node exporter on the NixOS host.
- Prometheus scrapes all machines (NAS, Play, Chromebox itself).

#### Zimaboard Play (NixOS)

Fully disposable. Nothing depends on it.

- Game servers (Minecraft, etc.)
- Nix experiments and learning
- Could become a second K3s node or dedicated runner host later
- Node exporter for monitoring

---

### Networking & DNS

#### How DNS works

- AdGuard Home runs on the NAS as a NixOS service
- Router DHCP hands out the NAS IP as the DNS server for all LAN devices.
- DNS rewrites are declared in Nix. version-controlled, reviewed in PRs, not clicked in a UI.

#### DNS rewrite config (in NixOS)

```nix
services.adguardhome = {
  enable = true;
  mutableSettings = false;
  settings.dns.rewrites = [
    # NAS services
    { domain = "immich.home.prowe.ca";    answer = "<NAS_IP>"; }
    { domain = "jellyfin.home.prowe.ca";  answer = "<NAS_IP>"; }
    { domain = "forgejo.home.prowe.ca";   answer = "<NAS_IP>"; }
    { domain = "home.prowe.ca";           answer = "<NAS_IP>"; }
    { domain = "pi.home.prowe.ca";        answer = "<NAS_IP>"; }
    # Chromebox services
    { domain = "grafana.home.prowe.ca";   answer = "<METALLB_IP>"; }
    { domain = "mealie.home.prowe.ca";    answer = "<METALLB_IP>"; }
    { domain = "couchdb.home.prowe.ca";   answer = "<METALLB_IP>"; }
  ];
};
```

#### TLS certificates

- Caddy on the NAS handles TLS for NAS services via Cloudflare DNS-01
- cert-manager in K3s continues handling TLS for K8s services.

#### Domain layout

- `*.home.prowe.ca` — all internal services (resolved via AdGuard rewrites).
- `photos.prowe.ca` — public Immich access via Cloudflare Tunnel (friends don't need Tailscale).

#### Remote access via Tailscale

- Every machine runs Tailscale.
- Tailscale DNS config (admin console): add the NAS as a nameserver restricted to `home.prowe.ca`.
- From outside: phone/laptop → Tailscale → NAS resolves `*.home.prowe.ca` → routes to correct machine → Caddy/Traefik terminates TLS.
- Same URLs work on LAN and remotely.

---

### Immich Sharing (for friends)

- Immich has built-in shared links: create an album, generate a link, toggle "allow uploads," optionally set a password.
- Friends open the link in a browser. no account needed, no Tailscale needed.
- Public access via Cloudflare Tunnel at `photos.prowe.ca`. Outbound-only connection, no ports opened.

---

### Buddy Backup with Dad's QNAP

#### Tailscale setup

- Dad has his own free Tailscale account and tailnet.
- Tailscale installed on dad's QNAP (most models support it natively).
- Dad shares the QNAP device with your tailnet using Tailscale's "share a device" feature.
- Result: the QNAP appears in your tailnet as a reachable machine. Dad doesn't see my other devices, I don't see his.

#### Backup flow

- ZFS snapshots on the NAS (local rollback).
- restic encrypts + deduplicates → pushes to `<QNAP_TAILSCALE_IP>:/backup/parker/` via SFTP/rsync.
- Nightly NixOS systemd timer.
- healthchecks.io ping at my end of the cron job. alerts you if a backup doesn't complete.
- Dad can't read my data (restic encryption). You get point-in-time restores.

#### Reverse direction (optional)

- Dad backs up his QNAP to my NAS. I share your NAS device with his tailnet.

---

### Management

#### Day-to-day monitoring

- Grafana dashboards (on Chromebox K3s). CPU, RAM, disk for all machines via node-exporter.
- healthchecks.io? alerts for backup failures or service downtime.
- Accessible over Tailscale from anywhere.

#### Config changes

- All infrastructure declared in one git repo (Nix flake + K8s manifests).
- Push a commit → SSH into NAS → `nixos-rebuild switch --flake .#nas`.
- For other machines: `nixos-rebuild switch --flake .#play --target-host play.home.prowe.ca` from the NAS.
- Or set up a Forgejo webhook to auto-rebuild on push.
- Have some way to auto-rollback if some checks fail.

#### Management from phone

- Tailscale SSH + terminal app on phone.
- Pi mobile app or self-hosted Pi dashboard (`pi.home.prowe.ca`) connects over Tailscale.

---

### Repo Structure

One monorepo, combined with my `.dotfiles` repo.

```
infra/
├── flake.nix
├── flake.lock
├── hosts/
│   ├── nas/          # NixOS config: AdGuard, Caddy, Immich, Jellyfin, Forgejo, Samba, ZFS, cloudflared, Pi agent
│   ├── chromebox/    # NixOS host config: Home Assistant VM, K3s service, node-exporter
│   ├── play/         # NixOS config: game servers, experiments
│   └── */            # Other personal machine(s)
├── modules/
│   ├── common.nix    # Tailscale, users, SSH, base packages
│   ├── caddy.nix     # Reverse proxy module
│   ├── monitoring.nix# Node exporter, promtail
│   └── ...
├── home/             # home-manager configs (dotfiles, shell, editor)
└── k8s/              # Flux GitOps manifests (Mealie, Grafana, etc.)
    ├── apps/
    ├── infrastructure/
    └── monitoring/
```

- Flux points at the `k8s/` subdirectory.
- Deploy NixOS changes: `colmena apply` or something like `deploy-rs` pushes to all machines, with automatic rollback if a host doesn't come back.
- Dotfiles are home-manager configs. no separate repo.

---

### Homepage

- A dashboard linking to all services (Homarr or Homepage).

---

### Future / Later

- **Router stuff** — eventually replace ISP router with something you control (OPNsense, etc.).
- **Second drive for NAS**
- **Forgejo webhook → auto-rebuild** — push to main triggers `nixos-rebuild` on affected hosts.
- **Oracle Cloud VM** Cloudflare Tunnel replaces its "public endpoint" role. Could decommission or find a new use for it. Or just keep it running jellyfin

---

### Key Principles

- **NAS is the foundation** — DNS, storage, photos, media. If it's up, the home network works.
- **Chromebox is non-critical** — K3s playground, Home Assistant, monitoring. If this machine goes down I can still access my files and movies. Home automations and fun little apps stop working but the critical ones stay up.
- **Play box is disposable**. Break things freely so long as nobody is gaming on it or something.
- **Single disk = no redundancy** buddy backup with dad is not optional
- **Everything is code** Nix flake for machines, Flux for K8s, DNS rewrites in Nix, no UI clicking required.
- **One repo** all infrastructure in one place. one `git log` shows everything.
