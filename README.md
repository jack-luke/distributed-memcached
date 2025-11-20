# Distributed Memcached Kubernetes Manifests

Kubernetes manifests that demonstrate various architectures for deploying a distributed memcached system.

## Description
This project consists of manifests for deploying memcached as a distributed cache system. 
When using memcached as a distributed system, configuration of the replication & partitioning of cache data is done via the [memcached proxy](https://docs.memcached.org/features/proxy/). 

The manifests that implement the proxy show some possible ways to configure the system that may improve cache availability, hit-rate, or latency.

## Manifests
All proxy configurations assume the cache servers are the ones described in [Cache StatefulSet](#cache-statefulset) deployed in the `cache` namespace.

### Cache StatefulSet
A StatefulSet of memcached servers to be leveraged as a distributed cache.
```bash
kubectl apply -f cache-statefulset.yaml
```
The headless service allows pods to be referenced individually for consistent hashing.

This setup deploys 3 servers, at most 1 per Kubernetes node, making it suitable for replicating data across nodes. 
However, the pod/node affinity and topology spread constraints should be tuned for the use case.

Memory limits for the memcached server and the pods should be carefully selected.
In this example, each server has up to 256MB of cache, so memory requests and limits are set to 300MiB and 400MiB respectively to allow for slab overhead.

### Proxy DaemonSet
A DaemonSet of memcached proxies to provide a simple, cluster-wide access point for a sharded cache.

<img src="./images/daemonset.svg" width=80% alt="Diagram showing how an app uses the memcached proxy daemonset to access a sharded cache across two Kubernetes nodes." >

```bash
kubectl apply -f proxy-daemonset.yaml
```
Pods exposed through a loadbalancer service on standard memcached port `11211`.

A ConfigMap configures the proxies to partition keys across StatefulSet pods, assumed to be in the `cache` namespace. 
This effecively pools the memory of all cache servers, meaning clients have a much larger cache available than with standalone cache servers.

### Proxy Sidecar
Implements memcached proxy as a sidecar to an application, to simplify application configuration.

<img src="./images/proxy-sidecar.svg" width=80% alt="Diagram showing how an app utilises a memcached proxy as a sidecar to access a replicated cache across two Kubernetes nodes.">

```bash
kubectl apply -f proxy-sidecar.yaml
```
The app interacts with the proxy as if it were a normal memcached server, on localhost port 11211.

In this example, the proxy is configured to replicate cache entries across each cache server pod, and cache lookups return the fastest available response. 
This means that any cache node going down will not result in lost cache entries, however the cache size available is the same as with standalone servers.

### Cache Sidecar with Cluster Failover
Implements a small, local cache and proxy as sidecars, which fails over to a cluster-wide cache pool.

<img src="./images/proxy-sidecar-cluster-failover.svg" width=80% alt="Diagram showing how an app can use a sidecar memcached proxy to access a local cache sidecar, as well as a cluster-wide cache." >

```bash
kubectl apply -f sidecar-cluster-failover.yaml
```
The app interacts with the proxy as if it were a normal memcached server, on localhost port 11211.

In this example, the proxy [prioritises a pool using local zones](https://docs.memcached.org/features/proxy/configure/#zones). 
This means that cache reads first query a cache container running in the same pod as the app for lowest latency.
If this misses, the request polls a larger, sharded cache pool. Writes are synchronously written to both local and pooled cache for consistency.

> [!WARNING]
> Only the cluster will be updated with writes from other app clients, so long-living entries in the local cache sidecar may become inconsistent.

The local cache container is configured with 64MB of cache.

## Help & Resources
* [Configuring memcached proxy](https://docs.memcached.org/features/proxy/configure/)
* [Memcached proxy configuration examples](https://docs.memcached.org/features/proxy/examples/)