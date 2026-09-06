---
title: "A Two-Node Bare-Metal K3s Cluster as a Systems Lab"
date: 2026-09-06 10:00:00 +0200
categories: [Systems, Homelab]
tags: [kubernetes, k3s, longhorn, flux-cd, gitops, traefik, cert-manager, storage, self-hosting]
pin: false
toc: true
---

I run a two-node Kubernetes cluster on hardware I can physically touch. It hosts things I actually
depend on — mail, photos, notes, media — but that is not really why it exists. It exists because
distributed storage, scheduling and failure recovery are the subjects I work on, and reading about
them is not the same as being woken up by them.

This post is about what the cluster teaches, not about how to build one.

> This is the successor to [Nextcloud on RAID with SSL and Domain](/posts/Nextcloud-on-RAID-with-SSL-and-Domain/),
> which I wrote in 2024. That setup — one box, one RAID array, Apache, certificates renewed by hand —
> is comprehensively superseded by what is described here, and I have left it up only as a record of
> where this started.
{: .prompt-info }

## Why real hardware and not a VM

The honest answer is that virtual machines are too well behaved.

A hypervisor gives you disks that never develop bad sectors, a network with no jitter and no
renegotiation, and a storage controller that is a software abstraction pretending to be silicon. If
you build a replicated storage system on top of that, you get to observe the happy path in high
resolution and almost nothing else. The failure modes that make distributed storage hard — a disk
that gets slow before it gets dead, a link that drops for four seconds, a controller with its own
opinions about write caching — simply do not occur.

Real hardware supplies them for free, on its own schedule. That is the entire point. The cluster is
useful to me precisely in the moments it misbehaves, because those are the moments that correspond
to the assumptions I would otherwise be making silently in a paper.

There is a second reason, less philosophical: on bare metal you own the whole stack. There is no
layer below you that someone else is tuning. When something is slow, the answer is somewhere between
the application and the platters, and it is all yours to find.

## The topology: compute and storage, deliberately split

Two nodes, with different jobs:

- **`n150`** — a mini-PC, the compute node. Small, quiet, low power, no meaningful local storage.
- **`t320`** — a Dell PowerEdge T320, the storage node. A PERC H710 RAID controller, a Mercusys
  2.5G NIC, Debian, networking managed with `systemd-networkd`.

Splitting them is not a capacity decision — it is a way of making the interesting boundary visible.
When compute and storage live in the same box, every I/O path is a memory copy and the cost of
getting data to the code is invisible. Separate them across a link and the boundary becomes something
you can measure, saturate and reason about. Which is the same boundary that dominates parallel I/O on
a real cluster, only smaller and cheaper to break.

The 2.5G NIC exists for that reason. It is a deliberate constraint: fast enough that the system is
usable, slow enough that replication traffic and workload traffic actually contend, so I can see the
contention rather than assume it away.

The PERC H710 is its own lesson. It is a hardware RAID controller from an era that assumed it knew
better than the operating system, and a modern software-defined storage layer would rather talk to
disks directly. Making those two worldviews coexist is not a configuration detail; it is a small
worked example of what happens when a storage abstraction is imposed from below by firmware you do
not control.

## Longhorn and the three StorageClasses

The cluster runs [Longhorn](https://longhorn.io/) for distributed block storage, with **three
StorageClasses** rather than one. That is the design decision I find most instructive, because it
forces a per-workload answer to a question most setups answer once and forget.

The question is: *what is this data worth, and what are you willing to pay to keep it?*

Replication is not free. Every additional replica multiplies write amplification across the link,
consumes capacity, and adds a participant that has to acknowledge before a write is durable. Paying
that cost for data you could regenerate from scratch in an afternoon is waste. Not paying it for data
that exists nowhere else is negligence. Between those two extremes is a spectrum, and a single
default StorageClass quietly places every workload at the same point on it.

So the classes differ in the replication guarantee they offer, and each workload is assigned to the
one that matches what its data actually is:

- Irreplaceable state — mail, photos, notes — where the write cost is worth paying.
- Reproducible state — caches, media that can be re-acquired, derived artefacts — where it is not.
- State whose access pattern makes locality matter more than redundancy.

The general lesson transfers directly upward. Parallel file systems make exactly this trade — stripe
width, replication, and where the durability boundary sits — and they make it at a scale where
getting it wrong is expensive. Making the same decision three times on hardware I own, and then
living with the consequences, is a cheap way to develop an intuition for it.

## GitOps with Flux: the cluster as a reproducible artefact

The entire cluster state is declared in Git and reconciled by [Flux CD](https://fluxcd.io/). Nothing
is configured by logging into a node and typing. Ingress is [Traefik](https://traefik.io/), TLS
certificates are issued and renewed automatically by [cert-manager](https://cert-manager.io/), and
both are described in the same repository as everything else.

The practical benefit is that recovery becomes an operation with a defined outcome. If a node is
lost, rebuilding it is not an exercise in remembering what past-me did — it is a checkout and a
reconcile. But the benefit I care about more is *reproducibility as a property of the system rather
than of my memory*, which is the same property I want from an experimental setup.

This is worth stating plainly, because it is the part of research infrastructure that is most often
handled badly: an experiment whose environment cannot be reconstructed is an experiment whose results
cannot be trusted. A cluster that is a Git repository is a cluster whose configuration can be
diffed, reviewed and rolled back. Running my own infrastructure that way is practice for running
experiments that way.

## What actually runs on it

The workload matters because it is what generates the load and the failures. It is a deliberately
mixed bag, which is the point — different services stress different parts of the system:

| Service | What it stresses |
|---|---|
| **Stalwart** mail server | Availability and durability; mail loss is not recoverable |
| **Immich** | Large-object storage, sustained write throughput |
| **Jellyfin** (with a GPU pinned to the pod) | Device scheduling, hardware passthrough |
| **Jellyseerr** and the *arr stack | Service-to-service dependencies, restart ordering |
| **Copyparty** | Bulk transfer, filesystem semantics over HTTP |
| **CouchDB** (Obsidian Self-hosted LiveSync) | Small, frequent, latency-sensitive writes |

Backups run through **K8up**, using `rclone serve restic` to target WebDAV storage on a separate
Nextcloud instance. That indirection — a restic repository served over WebDAV by a translation layer
— is uglier than it sounds on paper, and it is a fair illustration of what integrating heterogeneous
storage interfaces actually costs in practice.

Pinning a GPU to the Jellyfin pod deserves a mention of its own. It is the smallest possible instance
of a scheduling problem that is very large in HPC: an exclusive, non-divisible accelerator that a
scheduler has to allocate, that cannot be oversubscribed, and whose availability constrains where the
work can run at all.

## What carries over into the research

The line from here to what I do at ARCOS is shorter than it looks.

**Parallel and distributed storage.** Longhorn's replication is a small, legible version of the
question every parallel file system answers: where does the durability boundary sit, who acknowledges
a write, and what does the system do when a participant stops responding. Having tuned that by hand
against a link I can saturate, the trade-offs in a parallel I/O paper read differently.

**Dependable systems.** Failure recovery on paper is a state machine. Failure recovery on real
hardware is a state machine plus incomplete information plus whatever the disk decided to do. The gap
between those two is where the interesting research questions live, and running something I depend on
is how I expect to keep meeting it.

**Reproducibility.** A GitOps cluster and a reproducible experiment are the same discipline applied
to different artefacts. Practising it on infrastructure I own means it is already a habit when it
matters for results.

**Scheduling under constraint.** One GPU, one pod, no oversubscription. Scale that up and it is a
resource manager on a real machine — the same problem, with more zeros.

None of this replaces working on actual HPC systems. What it does is keep the abstractions honest:
it is harder to write breezily about replication cost or failure recovery when there is a machine in
the next room whose behaviour will eventually disagree with you.
