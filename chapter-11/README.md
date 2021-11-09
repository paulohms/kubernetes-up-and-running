# DaemonSets

A DaemonSet ensures a copy of a Pod is running across a set of nodes in a Kubernetes cluster. DaemonSets are used to deploy system daemons such as log collectors and monitoring agents, which typically must run on every node. DaemonSets share similar functionality with ReplicaSets; both create Pods that are expected to be long-running services and ensure that the desired state and the observed state of the cluster match.

Given the similarities between DaemonSets and ReplicaSets, it’s important to understand when to use one over the other. ReplicaSets should be used when your application is completely decoupled from the node and you can run multiple copies on a given node without special consideration. DaemonSets should be used when a single copy of your application must run on all or a subset of the nodes in the cluster.

You should generally not use scheduling restrictions or other parameters to ensure that Pods do not colocate on the same node. If you find yourself wanting a single Pod per node, then a DaemonSet is the correct Kubernetes resource to use. Likewise, if you find yourself building a homogeneous replicated service to serve user traffic, then a ReplicaSet is probably the right Kubernetes resource to use.

## DaemonSet Scheduler

By default a DaemonSet will create a copy of a Pod on every node unless a node selector is used, which will limit eligible nodes to those with a matching set of labels. DaemonSets determine which node a Pod will run on at Pod creation time by specifying the nodeName field in the Pod spec. As a result, Pods created by DaemonSets are ignored by the Kubernetes scheduler.

## Creating DaemonSets

DaemonSets are created by submitting a DaemonSet configuration to the Kubernetes API server. The DaemonSet in `fluentd.yaml` will create a fluentd logging agent on every node in the target cluster.

DaemonSets require a unique name across all DaemonSets in a given Kubernetes namespace. Each DaemonSet must include a Pod template spec, which will be used to create Pods as needed. This is where the similarities between ReplicaSets and DaemonSets end. Unlike ReplicaSets, DaemonSets will create Pods on every node in the cluster by default unless a node selector is used.

Once you have a valid DaemonSet configuration in place, you can use the kubectl apply command to submit the DaemonSet to the Kubernetes API. In this section we will create a DaemonSet to ensure the fluentd HTTP server is running on every node in our cluster:

```bash
kubectl apply -f fluentd.yaml
```

Once the fluentd DaemonSet has been successfully submitted to the Kubernetes API, you can query its current state using the kubectl describe command:

```bash
kubectl describe daemonset fluentd
```

This output indicates a fluentd Pod was successfully deployed to all three nodes in our cluster. We can verify this using the kubectl get pods command with the -o flag to print the nodes where each fluentd Pod was assigned:

```bash
kubectl get pods -o wide
```

With the fluentd DaemonSet in place, adding a new node to the cluster will result in a fluentd Pod being deployed to that node automatically.

This is exactly the behavior you want when managing logging daemons and other cluster-wide services. No action was required from our end; this is how the Kubernetes DaemonSet controller reconciles its observed state with our desired state.

## Limiting DaemonSets to Specific Nodes

The most common use case for DaemonSets is to run a Pod across every node in a Kubernetes cluster. However, there are some cases where you want to deploy a Pod to only a subset of nodes. For example, maybe you have a workload that requires a GPU or access to fast storage only available on a subset of nodes in your cluster. In cases like these, node labels can be used to tag specific nodes that meet workload requirements.

### Adding Labels to Nodes

The following command adds the ssd=true label to a single node:

```bash
kubectl label nodes minikube ssd=true
```

Using a label selector, we can filter nodes based on labels. To list only the nodes that have the ssd label set to true , use the kubectl get nodes command with the --selector flag:

```bash
kubectl get nodes --selector ssd=true
```

### Node Selectors

Node selectors can be used to limit what nodes a Pod can run on in a given Kubernetes cluster. Node selectors are defined as part of the Pod spec when creating a DaemonSet. The DaemonSet configuration in `nginx-fast-storage.yaml` limits NGINX to running only on nodes with the ssd=true label set.

Let’s see what happens when we submit the nginx-fast-storage DaemonSet to the Kubernetes API:

```bash
kubectl apply -f nginx-fast-storage.yaml
```

Since there is only one node with the ssd=true label, the nginx-fast-storage Pod will only run on that node.

```bash
kubectl get pods -o wide
```

## Updating a DaemonSet

DaemonSets are great for deploying services across an entire cluster, but what about upgrades? Prior to Kubernetes 1.6, the only way to update Pods managed by a DaemonSet was to update the DaemonSet and then manually delete each Pod that was managed by the DaemonSet so that it would be re-created with the new configuration. With the release of Kubernetes 1.6, DaemonSets gained an equivalent to the Deployment object that manages a DaemonSet rollout inside the cluster.

## Rolling Update of a DaemonSet

DaemonSets can be rolled out using the same RollingUpdate strategy that deployments use. You can configure the update strategy using the spec.updateStrategy.type field, which should have the value RollingUpdate . When a DaemonSet has an update strategy of RollingUpdate, any change to the spec.template field (or subfields) in the DaemonSet will initiate a rolling update.

There are two parameters that control the rolling update of a DaemonSet:
* spec.minReadySeconds , which determines how long a Pod must be “ready” before the rolling update proceeds to upgrade subsequent Pods;
* spec.updateStrategy.rollingUpdate.maxUnavailable , which indicates how many Pods may be simultaneously updated by the rolling update;

You will likely want to set spec.minReadySeconds to a reasonably long value, for example 30–60 seconds, to ensure that your Pod is truly healthy before the rollout proceeds.

The setting for spec.updateStrategy.rollingUpdate.maxUnavailable is more likely to be application-dependent. Setting it to 1 is a safe, general-purpose strategy, but it also takes a while to complete the rollout (number of nodes × minReadySeconds). Increasing the maximum unavailability will make your rollout move faster, but increases the “blast radius” of a failed rollout.

## Deleting a DaemonSet

Deleting a DaemonSet is pretty straightforward using the kubectl delete command. Just be sure to supply the correct name of the DaemonSet you would like to delete:

```bash
kubectl delete -f fluentd.yaml
```

Deleting a DaemonSet will also delete all the Pods being managed by that DaemonSet. Set the --cascade flag to false to ensure only
the DaemonSet is deleted and not the Pods.