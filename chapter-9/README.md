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

## Adopting Existing Containers

ReplicaSets are decoupled from the Pods they manage, you can simply create a ReplicaSet that will “adopt” the existing Pod, and scale out additional copies of those containers. In this way, you can seamlessly move from a single imperative Pod to a replicated set of Pods managed by a ReplicaSet.

## Quarantining Containers

In these situations, while it would work to simply kill the Pod, that would leave your developers with only logs to debug the problem. Instead, you can modify the set of labels on the sick Pod.

## Designing with ReplicaSets

ReplicaSets are designed to represent a single, scalable microservice inside your architecture. The key characteristic of ReplicaSets is that every Pod that is created by the ReplicaSet controller is entirely homogeneous. ReplicaSets are designed for stateless (or nearly stateless) services. The elements created by the ReplicaSet are interchangeable; 
when a ReplicaSet is scaled down, an arbitrary Pod is selected for deletion. Your application’s behavior shouldn’t change because of such a scale-down operation

## ReplicaSet Spec

ReplicaSets are defined using a specification. All ReplicaSets must have a unique name (defined using the metadata.name field), a spec section that describes the number of Pods (replicas) that should be running clusterwide at any given time, and a Pod template that describes the Pod to be created when the defined number of replicas is not met.

```yml
    apiVersion: extensions/v1beta1
    kind: ReplicaSet
    metadata:
        name: kuard
    spec:
        replicas: 1
        template:
            metadata:
                labels:
                    app: kuard
                    version: "2"
            spec:
                containers:
                    - name: kuard
                    image: "gcr.io/kuar-demo/kuard-amd64:green"
 ```

## Pod Templates
when the number of Pods in the current state is less than the number of Pods in the desired state, the ReplicaSet controller will create new Pods. The Pods are created using a Pod template that is contained in the ReplicaSet specification. The Pods are created in exactly the same manner as when you created a Pod from a YAML file.

```yml
template:
    metadata:
        labels:
            app: helloworld
            version: v1
    spec:
        containers:
            - name: helloworld
            image: kelseyhightower/helloworld:v1
            ports:
            - containerPort: 80
```

## Labels
ReplicaSets monitor cluster state using a set of Pod labels. Labels are used to filter Pod listings and track Pods running within a cluster

## Creating a ReplicaSet
```bash
kubectl apply -f kuard-rs.yaml
```

Once the kuard ReplicaSet has been accepted, the ReplicaSet controller will detect that there are no kuard Pods running that match the desired state, and a new kuard Pod will be created based on the contents of the Pod template

```bash
kubectl get pods
```

## Inspecting a ReplicaSet
If you are interested in further details about a ReplicaSet, the describe command will provide much more information about its state.

```bash
kubectl describe rs kuard
```

You can see the label selector for the ReplicaSet, as well as the state of all of the replicas managed by the ReplicaSet.

## Finding a ReplicaSet from a Pod
You may wonder if a Pod is being managed by a ReplicaSet. To enable this kind of discovery, the ReplicaSet controller adds an annotation to every Pod that it creates. The key for the annotation is ownerReferences. If you run the following, look for the ownerReferences entry in the annotations section

```bash
kubectl get pods <pod-name> -o yaml
```

## Finding a Set of Pods for a ReplicaSet

You can also determine the set of Pods managed by a ReplicaSet. First, you can get the set of labels using the kubectl describe command. To find the Pods that match this selector,
use the --selector flag or the shorthand -l

```bash 
kubectl get pods -l app=kuard
```

This is exactly the same query that the ReplicaSet executes to determine the current number of Pods