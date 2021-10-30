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
