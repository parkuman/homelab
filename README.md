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

## Repo structure

The current repo structure is close to the one outlined in the [Flux Documentation](https://fluxcd.io/flux/guides/repository-structure/), but more suited to a simple homelab setup.
