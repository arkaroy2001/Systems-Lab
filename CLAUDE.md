# Backend Systems Deep-Dive Plan

Source: https://docs.google.com/document/d/1pVPpgd3NxclO2gntKl3wmGLZiXVdwp27G4secb96-zo/edit?tab=t.0

## Overview
This roadmap guides full-stack engineers transitioning to backend/infrastructure roles at top tech companies through four progressive projects designed to demonstrate systems reasoning, failure thinking, and implementation depth.

## The Four Projects

### Project 1: Mini-Kafka (Replicated Log) — `broker/`
Building a Kafka-like system covering append-only logs, replication, consumer groups, and partition assignment. The foundation teaches "everything event-based in modern backend systems."

### Project 2: Streaming Engine — `stream-engine/`
A Flink-like processor handling event-time processing, watermarks, windowed aggregations, and checkpointing. Critical for understanding "exactly-once semantics, which are among the hardest problems in distributed systems."

### Project 3: Real-Time Anomaly Detector — `anomaly-detector/`
Transaction fraud detection using stateful windowed rules (velocity checks, amount spikes, geo-impossible travel). Demonstrates streaming skills on interview-relevant use cases.

### Project 4: Distributed Task Scheduler — `task-scheduler/`
Fault-tolerant task distribution across workers using consistent hashing, leader election, and exactly-once execution. Covers coordination and consistency gaps.

## Shared Directories
- `common/` — Shared utilities and code used across projects
- `docker/` — Docker configuration files
- `scripts/` — Deployment and utility scripts

## Timeline
- **Full pace**: ~30 weeks (7-8 months) at 10-15 hours/week
- **Compressed**: ~22 weeks (5-6 months) with scoped cuts
- Projects 1-2 are non-negotiable; Projects 3-4 can be condensed if needed

## Interview Preparation
- Practice systems design after completing Projects 1-2
- Dedicate 3-4 hours weekly to algorithm practice (graphs, trees, dynamic programming)
- Complete 5+ mock interviews before real loops
- Read *Designing Data-Intensive Applications* and *Streaming Systems* alongside projects

## Success Criteria
A complete portfolio allows peers to: clone the repo, run the system via Docker, execute chaos tests demonstrating recovery, and understand design invariants from the README.
