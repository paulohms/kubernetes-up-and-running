# ReplicaSets

Previously, we covered how to run individual containers as Pods. But these Pods are essentially one-off singletons. More often than not, you want multiple replicas of a
container running at a particular time. There are a variety of reasons for this type of replication:

### Redundancy
Multiple running instances mean failure can be tolerated.
### Scale
Multiple running instances mean that more requests can be handled.
### Sharding
Different replicas can handle different parts of a computation in parallel.

Pods managed by ReplicaSets are automatically rescheduled under certain failure conditions, such as node failures and network partitions

## Reconciliation Loops

The reconciliation loop is constantly running, observing the current state of the world and taking action to try to make the observed state match the desired state. For
instance, with the previous examples, the reconciliation loop would create a new kuard Pod in an effort to make the observed state match the desired state of three
replicas.

Reconciliation loop for ReplicaSets is a single loop, yet it handles user actions to scale up or scale down the ReplicaSet as well as node failures or nodes rejoining the cluster after being absent.

## Relating Pods and ReplicaSets

The relationship between ReplicaSets and Pods is loosely coupled. Though ReplicaSets create and manage Pods, they do not own the Pods they create.

Replica‐Sets use label queries to identify the set of Pods they should be managing. They then use the exact same Pod API that you used directly in Chapter 5 to create the Pods that
they are managing

ReplicaSets that create multiple Pods and the services that load-balance to those Pods are also totally separate, decoupled API objects.