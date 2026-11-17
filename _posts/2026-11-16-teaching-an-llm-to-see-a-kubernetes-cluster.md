---
layout: post
title: "Teaching an LLM to see a Kubernetes cluster"
date: 2026-11-16
series: "Building a Kubernetes graph for AI agents"
part: 1
summary: What I learned mapping Kubernetes relationships by hand, and the bugs my tests caught before any agent saw the data.
tags: [Kubernetes, MCP, Python, Testing]
project: kubernetes-graph-agent-context
canonical: ""
---

*Building a Kubernetes graph for AI agents, part 1: what I learned mapping relationships by hand, and the bugs my tests caught before any agent saw the data.*

I'm building an MCP server that gives AI agents a live graph of a Kubernetes cluster: what owns what, where it runs, and what depends on it. The idea is simple. Instead of an agent reading pages of raw `kubectl` output and guessing at connections, it asks the graph and gets back a small, connected answer.

This is the first post in a building-in-public series. It makes no claims about how well agents perform with the graph. That needs a benchmark, which comes later. This one is about the groundwork: what I had to learn about Kubernetes to model it honestly, and the mistakes my tests caught along the way.

I didn't start with code. I started by mapping relationships by hand.

## Every edge has to come from somewhere

Before writing a collector, I deployed Google's Online Boutique demo on a local kind cluster and traced how two services connect, field by field. The rule I set myself: every edge in the graph must point to the exact field that proves it.

That exercise reshaped the design more than any code did.

**Ownership lives on the child.** A pod knows its ReplicaSet through `metadata.ownerReferences`, and a ReplicaSet knows its Deployment the same way. The parent records nothing about its children. A collector learns ownership from the dependent object, never from the owner.

```json
{"apiVersion": "apps/v1", "kind": "ReplicaSet", "name": "frontend-7f5f76b5ff",
 "uid": "ea0a7824-...", "controller": true, "blockOwnerDeletion": true}
```

The `controller: true` flag matters. An object can have several owners, but at most one is the managing one. What `kubectl describe` shows as "Controlled By" is exactly that reference.

**Placement is set by the scheduler.** `spec.nodeName` is empty while a pod is Pending and filled in when it's scheduled. So a pod stuck without one is itself a signal. It's also a name-only reference, unlike ownerReferences, which carry a uid.

**Some relationships exist on neither side.** A Service selects pods by label, but the pod has no field saying "a Service selects me." The link only exists when you evaluate the selector. Change one label and the relationship disappears silently.

**Runtime calls appear in no field at all.** I knew the frontend calls the cart service. Kubernetes doesn't. The closest thing is an environment variable holding the address:

```text
CART_SERVICE_ADDR=cartservice:7070
```

That proves configuration, not traffic. The variable could be stale or unused. So the graph needs to tell the agent which edges are declared and which are observed. That became one of the core design rules: every edge records its source.

## Names lie, pods tell the truth

Three discoveries changed how the graph identifies and sources objects.

**Key by uid, not by name.** Delete and recreate a pod with the same name and it gets a new uid. A graph keyed by name would attach a dead pod's history to its replacement. So nodes are keyed by uid, with kind, namespace and name kept as readable labels.

**Old ReplicaSets don't go away.** I triggered a rollout with `kubectl rollout restart` and watched. The old ReplicaSet stayed, scaled to zero, and the new one got a higher `deployment.kubernetes.io/revision` annotation. Mid-rollout both have replicas, so replica count can't tell you which is current. The revision annotation can. Keeping old ReplicaSets in the graph turns out to be useful: "did this start after the last deploy?" becomes answerable.

**The template is intent; the pod is truth.** A Deployment's pod template had no volumes. The running pod had one, injected by the ServiceAccount admission controller: a projected volume with an API token, the cluster's CA certificate and the pod's namespace. Admission controllers change pods on the way in, the scheduler adds a node, and mid-rollout old pods still run the old template. So every pod-level edge is built from the live pod, never the template.

That injected volume taught one more lesson. Almost every pod mounts the same `kube-root-ca.crt` ConfigMap. Add it as an ordinary edge and that ConfigMap connects to everything, so every "what's related to X" query returns the whole cluster. Hub nodes like that get tagged and excluded from traversal by default.

## Teaching the graph to say "I don't know"

An agent is only as honest as its context. If the graph presents guesses as facts, the agent will too. So every node carries an `exists` field with three values:

| `exists` | Meaning | Example |
| --- | --- | --- |
| `true` | Verified: the collector listed it | A ReplicaSet returned by the API |
| `false` | Referenced, and missing from a kind we did list | A pod's owner ReplicaSet that wasn't in the list |
| `null` | Referenced, but never checked | A Deployment, before Deployments are collected |

The distinction between `false` and `null` is the whole point: "we looked and it's gone" versus "we never looked." An agent that says a `null` object is healthy is making something up.

Referenced-but-missing objects become explicit placeholder nodes. A pod pointing at a deleted ConfigMap may be running fine today, because its config was read at startup, and fail on its next restart. That kind of time bomb is exactly what a graph can surface and a single `kubectl get` can't.

**Static pods were the strangest case.** Pods like `kube-apiserver` on a kind control-plane node are defined as files on the node and run by the kubelet. The API shows a read-only "mirror" of them, whose owner reference points to the Node. The Node isn't really managing anything; it stands in for the kubelet on that machine. For an agent this matters a lot: `kubectl edit` or `kubectl delete` won't fix a static pod, because the kubelet recreates it from the file. So mirror pods get a `static: true` flag, derived from the `kubernetes.io/config.mirror` annotation.

## The bugs my tests caught

None of these crashed anything. Every one would have produced a graph that looked fine and was quietly wrong, which is the worst kind of bug to hand to an agent.

### 1. One edge silently overwrote another

A static pod has two relationships to the same Node: it's owned by it and it runs on it. My first graph was a networkx `DiGraph`, which allows only one edge between any pair of nodes. The second edge replaced the first:

```python
import networkx as nx

g = nx.DiGraph()
g.add_edge("pod", "Node/w1", type="OWNED_BY")
g.add_edge("pod", "Node/w1", type="RUNS_ON")
print(g.number_of_edges())          # 1
print(g.edges["pod", "Node/w1"])    # {'type': 'RUNS_ON'}  (ownership is gone)

m = nx.MultiDiGraph()
m.add_edge("pod", "Node/w1", key="OWNED_BY", type="OWNED_BY")
m.add_edge("pod", "Node/w1", key="RUNS_ON", type="RUNS_ON")
print(m.number_of_edges())          # 2
```

The fix was a `MultiDiGraph` with the edge type as the key. Re-adding the same relationship updates it instead of duplicating it, which matters once the graph is kept live.

### 2. `add_edge` invents nodes

In networkx, adding an edge to a node that doesn't exist creates that node, with no attributes. A pod whose owner wasn't listed would get an edge to a nameless, kindless node. The graph still "worked." The fix: add every referenced-but-unlisted object as an explicit placeholder first, and a test helper that checks invariants on every graph:

```python
def assert_graph_invariants(graph):
    for key, data in graph.nodes(data=True):
        assert "kind" in data and "exists" in data, key
    for u, v, k, data in graph.edges(keys=True, data=True):
        assert data.get("type") == k and "source" in data, (u, v, k)
```

### 3. The test that couldn't fail

When I switched to `MultiDiGraph`, I passed `key=OWNED_BY` instead of `type=OWNED_BY`, rather than both. Every lookup used keys, so every test passed. One test checked that unscheduled pods have no `RUNS_ON` edge by looking at the `type` attribute. With `type` missing everywhere, that check could never fail.

The invariant helper caught it immediately:

```text
AssertionError: edge pod-uid->owner-uid key OWNED_BY != type None
E           assert None == 'OWNED_BY'
E            +  where None = <built-in method get of dict object at 0x000001B845B718C0>('type')
E            +    where <built-in method get of dict object at 0x000001B845B718C0> = {'controller': False, 'source': 'metadata.ownerReferences'}.get
```

The lesson: an absence test passes trivially when the thing it looks for can't exist. Pair every "X is absent" test with a check that X would be present in the right case.

### 4. Tests that passed on empty lists

My first integration test against a disposable kind cluster asserted that the collector returned lists. It would have passed with a typo in the namespace, because listing a namespace that doesn't exist returns an empty list, not an error. Useful tests assert content: CoreDNS pods exist and are owned by a CoreDNS ReplicaSet.

### 5. "Ready" is inside "NotReady"

The same test once checked node health with `assert "Ready" in output`. That passes for `NotReady` too. Parse structured output instead of searching text.

After the fixes, the suite looks like this:

```text
tests/test_collector.py::test_fetch_objects_returns_pods_and_replicasets PASSED
tests/test_collector.py::test_fetch_objects_with_empty_namespace_raises_value_error[] PASSED
tests/test_collector.py::test_fetch_objects_with_empty_namespace_raises_value_error[  ] PASSED
tests/test_collector.py::test_fetch_objects_with_empty_namespace_raises_value_error[\t] PASSED
tests/test_collector_kind.py::test_collector_reads_kind_cluster SKIPPED (Set RUN_KIND_TESTS=1 to...)
tests/test_graph.py::test_build_graph_adds_missing_owner_placeholder PASSED
tests/test_graph.py::test_build_graph_keeps_collected_replicaset_as_existing_owner PASSED
tests/test_graph.py::test_build_graph_adds_owner_edge_from_replicaset PASSED
tests/test_graph.py::test_build_graph_adds_runs_on_edge_for_scheduled_pod PASSED
tests/test_graph.py::test_build_graph_does_not_add_runs_on_edge_for_unscheduled_pod PASSED
tests/test_graph.py::test_build_graph_bare_pod_has_no_owned_by_edges PASSED
tests/test_graph.py::test_node_key[Node-worker-1-None-Node/worker-1] PASSED
tests/test_graph.py::test_node_key[Pod-web-pod-pod-uid-pod-uid] PASSED
tests/test_graph.py::test_node_key[Deployment-web-deployment-uid-deployment-uid] PASSED
tests/test_graph.py::test_node_key_raises_when_non_node_has_no_uid PASSED
tests/test_graph.py::test_build_graph_static_pod_owner_and_node_resolve_to_one_node PASSED
tests/test_graph.py::test_build_graph_preserves_non_controller_owner PASSED
tests/test_loading.py::test_load_objects_from_fixture PASSED
tests/test_queries.py::test_describe_component_returns_node_and_relationships PASSED
tests/test_queries.py::test_describe_component_returns_not_found_for_missing_node PASSED
tests/test_server.py::test_server_status_tool PASSED
tests/test_server.py::test_failing_server_status_tool PASSED

21 passed, 1 skipped in 2.42s
```

The skipped test is the opt-in integration test against a disposable kind cluster; the default run stays fast and needs no cluster.

## What I'd tell myself at the start

- **Map it by hand first.** An hour of `kubectl get -o yaml` saved days of guessing where edges come from.
- **Make every edge cite its source.** Declared, computed and observed relationships deserve different levels of trust, and the agent needs to know which is which.
- **Model uncertainty explicitly.** `exists: null` is more honest than a confident answer, and more useful.
- **Treat silent wrongness as the main risk.** The bugs that matter don't crash. Invariant checks catch whole classes of them at once.
- **Test objects and server fixtures serve different consumers.** Tests use Python objects directly; a server launched by an MCP host is a separate process, so its no-cluster mode reads JSON files, and one round-trip test proves that bridge holds.

## What's next

Next, the graph will feed a small MCP server. The first tool, which looks up a component and returns what owns it, where it runs and what depends on it, is in progress. After that come a tool that finds risks like single points of failure and dangling references, and then a benchmark: the same agent, the same injected failures, with and without the graph. That's the post where I'll find out whether any of this actually helps.

If you run data workloads on Kubernetes and have opinions about what an agent should be able to answer, I'd like to hear them.
