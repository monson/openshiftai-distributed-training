# Introduction

This example demostrate how to use quota allocation with Kueue with preemption.

## Overview 
In this example, there are 2 teams that work in their own namespace:

1. Team A and B belongs to the same [cohort](https://kueue.sigs.k8s.io/docs/concepts/cluster_queue/#cohort)
1. Both teams share a quota
1. Team A has access to GPU while team B does not
1. Team A has higher priority and can prempt others

### Kueue Configuration 

There are 2 `ResourceFlavor` that manages the [CPU/Memory](default-flavor.yaml) and [GPU](gpu-flavor.yaml) resources. The GPU `ResourceFlavor` tolerates nodes that have been tainted. 

Both teams have their invididual cluster queue that is associated with their respective namespace.

| Name                        | CPU | Memory (GB) | GPU 
| --------------------------- | --- | ----------- | ---
| [Team A cq](team-a-cq.yaml) | 0   | 0           | 2 
| [Team B cq](team-b-cq.yaml) | 0   | 0           | 0
| [Shared cq](shared-cq.yaml) | 10  | 64          | 0   

A local queue is defined in their namespace to associate the cluster queue. E.g.

``` yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: local-queue
  namespace: team-a
spec:
  clusterQueue: team-a-cq
```

When a Ray cluster is defined, it is submitted to the local queue with the associated priority.

``` yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  labels:  
    kueue.x-k8s.io/queue-name: local-queue
    kueue.x-k8s.io/priority-class: dev-priority
```

### Ray cluster configuration

> The shared quota is only up to 10 CPU for both teams.

| Name                                   | CPU | Memory (GB) | GPU 
| -------------------------------------- | --- | ----------- | ----
| [Team A](team-a-ray-cluster-prod.yaml) | 10  | 24          | 2 
| [Team B](team-b-ray-cluster-dev.yaml)  | 6   | 16          | 0


### Premption

Team A cluster queue has preemption defined that can `borrowWithinCohort` of a lower priority which Team B belongs to.

``` yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: team-a-cq
spec:
  preemption:
    reclaimWithinCohort: Any
    borrowWithinCohort:
      policy: LowerPriority
      maxPriorityThreshold: 100
    withinClusterQueue: Never
```

Team A will preempt team B because it has insufficient resources to run. 


## Setting Up the Demo

1. Ensure [prerequisite](../../) is already configured. 

2. Run the makefile target

```
make setup-kueue-examples
```

## Running the example


1. Create a ray cluster for team B. Wait for the cluster to be running.
```
oc create -f yaml/preemption/team-b-ray-cluster-dev.yaml
```

```bash
$ oc get rayclusters -A
NAMESPACE   NAME             DESIRED WORKERS   AVAILABLE WORKERS   CPUS   MEMORY   GPUS   STATUS   AGE
team-b      raycluster-dev   2                 2                   6      16G      0      ready    70s

$ oc get po -n team-b
NAME                                           READY   STATUS    RESTARTS   AGE
raycluster-dev-head-zwfd8                      2/2     Running   0          45s
raycluster-dev-worker-small-group-test-4c85h   1/1     Running   0          43s
raycluster-dev-worker-small-group-test-5k9j5   1/1     Running   0          43s
```

2. Create a Ray cluster for team A. 
```bash
oc create -f yaml/preemption/team-a-ray-cluster-prod.yaml
```

3. Observe team B cluster is suspended and team A cluster is running because of preemption.

```bash
$ oc get rayclusters -A
NAMESPACE   NAME              DESIRED WORKERS   AVAILABLE WORKERS   CPUS   MEMORY   GPUS   STATUS      AGE
team-a      raycluster-prod   2                 2                   10     24G      4      ready       75s
team-b      raycluster-dev    2                                     6      16G      0      suspended   3m46s
```





