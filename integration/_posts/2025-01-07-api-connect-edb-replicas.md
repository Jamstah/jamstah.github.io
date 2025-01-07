---
layout: post
tags: kubernetes OpenShift EDB PDB API_Connect cp4i integration
title: "IBM API Connect EDB Replica configuration"
excerpt_separator: <!--more-->
---

I spin up a lot of API Connect instances for testing, which means I want to use the slimmest profile.
I'm not going to have a lot of production traffic and I want it to be quick to get going.

A problem I hit regularly is that the slimmest profile only includes one database replica.
To minimise disruptions, there is an automatic [pod disruption budget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets) configured to ensure the database always has at least one replica.
This stops the node the database is on from draining, preventing maintenance such as cluster upgrades.

<!--more-->

To fix this, you can manually configure the number of database replicas for any profile.

When using the top-level CR of kind `APIConnectCluster`, you can add this snippet to your CR:

```yaml
spec:
  template:
    - name: mgmt-edb-cluster
      replicaCount: 2
```

When using sub-system CRs to build your API Connect cluster, you can add this snippet to your `ManagementCluster` CR:

```yaml
spec:
  template:
    - name: edb-cluster
      replicaCount: 2
```

The operator will automatically increase the number of EDB replicas allowing your nodes to drain without violating the pod disruption budget.