---
title: Kubernetes Graph as Agent Context
status: building
order: 2
summary: System tables for self-managed data platforms on Kubernetes — a relationship graph of the cluster, exposed to AI agents over MCP so they can answer what owns what, where it runs, and what's at risk.
stack: [Python, Kubernetes, networkx, MCP]
repo: ""
articles: [graph-for-agents-part-1]
---

## The idea

Agents troubleshooting a cluster today see flat lists of pods and events. The relationships —
what owns what, where it runs, what it depends on — are what an operator actually reasons with,
and many of them appear in no single object.

## Where it stands

- Graph core built and tested: ownership, placement, placeholders for missing or unchecked objects,
  provenance on every edge.
- Runs from a live cluster or from fixture files, so it works without a cluster.
- MCP tool (look up a component and its relationships) in progress.

## Direction

- Keep the graph live and add workloads, Services and scheduling signals.
- Tools that find risks and blast radius, not just look things up.
- Data-platform providers (Spark, Kestra, Iceberg) that join pipeline runs to the pods they ran on.
- A benchmark: the same agent, the same injected failures, with and without the graph.