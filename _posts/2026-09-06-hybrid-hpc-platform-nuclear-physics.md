---
title: "A Hybrid HPC Platform for Dynamic Search in Large Combinatorial Spaces"
date: 2026-09-06 08:30:00 +0200
categories: [Research, HPC]
tags: [hpc, mpi, openmp, apache-arrow, parquet, hdf5, redis, grafana, nuclear-physics, thesis]
pin: true
toc: true
math: true
---

<!-- TODO: cover image → media/2026-09-06-hybrid-hpc-platform-nuclear-physics/portada.png,
     then add the `image:` front-matter block. -->

> **Status: draft.** This is the write-up of my BSc thesis, and the sections marked as pending are
> being filled in from the dissertation itself. Nothing here is estimated or reconstructed from
> memory — where a number is missing, it is missing on purpose.
{: .prompt-warning }

My BSc thesis in Computer Engineering at Universidad Carlos III de Madrid was carried out at the
**CSIC**, on the project *"Visualization of Nuclear Structure and Reactivity Based on the Standard
Model of Particles and NT-FS&L"*, under Dr. Pedro Noheda and Nuria Tabarés. It was graded **9.6/10**.

The engineering problem was not the physics. It was that the search space the physics implies is far
too large to enumerate, and the people who needed to explore it needed to do so *interactively* —
steering the search while it ran, rather than submitting a batch job and reading the results the
next morning. Those two requirements pull in opposite directions, and the thesis is largely about
the architecture that reconciles them.

The result was a **hybrid platform**: a distributed C++ search engine running on CSIC's **DRAGO**
supercomputer, coupled to a local interactive layer, with **Redis** as the synchronisation boundary
between them.

## 1. The problem: what is being searched, and why the space explodes

<!-- TODO: from the thesis —
       - what a candidate in the search space actually is (a configuration? a
         combination of nuclear states? state the object precisely)
       - the parameters that define the space and how it grows with them
       - the closed form or bound for the size of the space, so the growth claim
         below can be made concrete with real numbers and typeset with MathJax
         (math: true is already enabled in the front matter)
       - what makes a candidate "interesting", i.e. the evaluation criterion
     Do not approximate any of this. -->

The essential difficulty is the one common to combinatorial search: the space grows fast enough that
exhaustive enumeration stops being a strategy and becomes a category error, while the interesting
region is small, unevenly distributed, and not known in advance. Which means the search cannot be
planned ahead of time — it has to adapt to what it finds.

That last point is what forces the architecture. A search whose shape is decided at submission time
can be a batch job. A search whose shape depends on intermediate results cannot.

## 2. Why neither local nor HPC alone was sufficient

Two obvious architectures were available, and both fail for the same reason from opposite directions.

**Purely local** fails on throughput. The evaluation of candidates is expensive and embarrassingly
parallel in the parts that matter, and a workstation gives up several orders of magnitude of
capacity that the problem can genuinely use.

**Purely HPC** fails on interaction. A supercomputer is a batch environment: work is queued,
scheduled, and run without a human in the loop. That is exactly the right model for a computation
whose parameters are known in advance, and exactly the wrong one for a search that a domain expert
needs to redirect based on what has been found so far. Queue latency alone destroys the interaction
loop, before considering that compute nodes are not where you want a visualisation front-end to live.

The hybrid design takes the part of the problem that is throughput-bound to DRAGO, keeps the part
that is latency-bound and human-facing local, and accepts the cost of a synchronisation boundary
between them as the price of getting both.

## 3. Architecture

<!-- TODO: architecture diagram →
     media/2026-09-06-hybrid-hpc-platform-nuclear-physics/architecture.png
     Should show: the DRAGO-side MPI ranks and their OpenMP threads, the Redis
     boundary, the local interactive/visualisation layer, and the Arrow/Parquet/
     HDF5 data path. Image reference omitted until the file exists — htmlproofer
     fails the deploy on a missing image path. -->

The compute side is **C++ with MPI and OpenMP** — MPI for distribution across DRAGO's nodes, OpenMP
for shared-memory parallelism within each of them. The local side handles steering and
visualisation. Between them sits Redis.

<!-- TODO: from the thesis —
       - the work decomposition: how the space is partitioned across ranks, and
         whether that partitioning is static or rebalanced during the run
       - the MPI communication pattern (collectives? point-to-point? a master/worker
         arrangement?) and where the synchronisation points are
       - the OpenMP parallelisation within a rank, and the granularity
       - how a steering command from the local side actually reaches the ranks and
         takes effect mid-run -->

## 4. Storage decisions: Arrow, Parquet and HDF5

The data path uses **Apache Arrow**, **Parquet** and **HDF5** rather than the two defaults most
projects reach for, and the reasoning is worth stating because it is a decision people get wrong by
inertia.

**Why not CSV.** CSV is a text format with no schema, no types, no compression and no structure that
a reader can exploit. Every read is a full parse, every numeric column is a string until proven
otherwise, and the cost scales with the size of the output — which, for a combinatorial search, is
the thing that grows. It is a fine interchange format and an actively harmful storage format at
this scale.

**Why not a relational database.** The access pattern is wrong for one. This is bulk columnar
analysis over append-only results: write a great many records, then read entire columns of them.
There are no updates, no transactions worth the name, no joins in the hot path, and no need for the
machinery that a relational engine spends its complexity budget on. A DBMS would add an operational
dependency and a serialisation bottleneck in exchange for guarantees the workload does not use.

**What the three do instead.** Arrow gives a columnar in-memory representation that can be handed
between processes and languages without a serialise/deserialise round trip — which matters precisely
because the pipeline crosses from C++ to Python. Parquet gives the same columnar layout on disk,
compressed and with the metadata to read a subset without touching the rest. HDF5 covers the
structured numerical arrays that do not fit a flat table cleanly.

<!-- TODO: from the thesis —
       - which format holds which artefact (results? checkpoints? intermediate state?)
       - the Parquet partitioning / row-group sizing actually used
       - whether writes were parallel, and if so how they were coordinated
       - measured comparison against a CSV baseline, if the thesis has one -->

## 5. Coordination through Redis

Redis is the boundary between the HPC side and the local side: the compute ranks publish results and
progress into it, and the local layer both reads that and writes steering decisions back.

The reason this works is that it decouples the two sides' lifetimes. The HPC job does not need the
local client to be connected, and the local client does not need to be running when the job starts or
survive when it ends. Each side talks to Redis, not to the other, which means neither is blocked on
the other's availability — the essential property when one side is a batch job on a shared machine
and the other is a person with a browser.

<!-- TODO: from the thesis —
       - which Redis structures were used, and for what (pub/sub? streams? lists? hashes?)
       - what exactly crosses the boundary in each direction, and at what rate
       - how consistency was handled when several ranks write concurrently
       - what happens on a Redis outage or a disconnect mid-run
       - whether Redis became a bottleneck, and at what scale -->

## 6. Results and performance

<!-- TODO: this section is empty on purpose, pending the thesis PDF. Required:
       - size of the combinatorial space explored
       - speedup and parallel efficiency, with the node/rank/thread counts they
         were measured at, and the baseline they are relative to
       - strong and/or weak scaling behaviour, and where it stops scaling
       - wall-clock times for representative runs
       - throughput of the data path (records/s, MB/s written)
     No figure goes in this section until it comes from a measurement. -->

The platform was run on **DRAGO**, CSIC's supercomputer, across multiple nodes and **192+ threads**.

## 7. Observability with Grafana

Monitoring was built with Python and **Grafana**, giving a live view of the search while it ran.

This was not instrumentation added at the end for the write-up. In a steered search it is part of the
control loop: the operator's decision about where to direct the search next is made from what the
dashboard shows, so the observability layer is functionally the user interface. A batch job can be
profiled after the fact; a job a human is steering has to be legible while it runs.

<!-- TODO: from the thesis —
       - the metrics actually exposed, and how they were fed to Grafana
       - the instrumentation overhead, if measured
       - a dashboard screenshot →
         media/2026-09-06-hybrid-hpc-platform-nuclear-physics/grafana.png -->

## 8. What I would do differently today

Some of this I already thought at the time; some of it only became clear after a year of writing
production systems and then coming back to research.

**I would put the durability boundary somewhere explicit.** Redis as the coordination layer is the
right shape, but "the results are in Redis until something writes them out" is a design where the
point of no return is implicit. I would now be able to say precisely what is lost when each component
dies, and I could not have said that then.

**I would treat the storage layer as a first-class experiment.** The Arrow/Parquet/HDF5 choice was
argued from properties rather than from measurements on this workload. The reasoning holds up, but I
now think a project whose output volume *is* the scaling problem should measure its own I/O path
rather than reason about it — which is, not coincidentally, close to what I work on now.

**I would design for resumption from the start.** A long search on a shared machine will be
interrupted — by a walltime limit, a queue policy, or a failure. Checkpoint-and-resume added later is
a much worse thing than checkpoint-and-resume designed in.

<!-- TODO: add anything from the thesis's own future-work section that you still
     agree with, plus whatever the ARCOS work has since made obvious. -->

---

*The full dissertation is available on request. See the [Publications](/publications/) page.*
