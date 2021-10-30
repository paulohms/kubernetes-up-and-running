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