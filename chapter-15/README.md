# Integrating Storage Solutions and Kubernetes
Decoupling state from applications and building your microservices to be as stateless as possible results in maximally reliable, manageable systems.

#### Every system that has any complexity has state 
However, nearly every system that has any complexity has state in the system somewhere, from the records in a database to the index shards that serve results for a web search engine. At some point, you have to have data stored somewhere.

#### Integrating this data with containers and orchestration solutions is the most complicated
This complexity largely stems from the fact that the move to containerized architectures is also a move toward decoupled, immutable, and declarative application development.

#### Example: consider setting up a ReplicaSet in MongoDB and then running an imperative command to identify the leader and the participants
As an example of this, consider setting up a ReplicaSet in MongoDB, which involves deploying the Mongo daemon and then running an imperative command to identify the leader, as well as the participants in the Mongo cluster. Of course, these steps can be scripted, but in a containerized world it is difficult to see how to integrate such commands into a deployment. Likewise, even getting DNS-resolvable names for individual containers in a replicated set of containers is challenging.

#### Evolution to the cloud means that storage is an externalized cloud service
In that context it can never really exist inside of the Kubernetes cluster.

#### This chapter covers a variety of approaches for integrating storage into containerized microservices in Kubernetes
1) How to import existing external storage solutions into Kubernetes. 

2) How to run reliable singletons inside of Kubernetes where you deployed storage solutions. 

3) StatefulSets, which are still under development.


## Importing External Services
You have an existing machine running with some of database running on it. In this situation you may not want to immediately move that database into containers and Kubernetes.

#### This legacy server and service are not going to move into Kubernetes
Regardless of the reasons for staying put, this legacy server and service are not going to move into Kubernetes. 

But it’s still worthwhile to represent this server in Kubernetes. 

#### You will get to take advantage
 - all of the built-in naming and service-discovery primitives provided by Kubernetes. 
 - this enables you to configure your applications so that it looks like the database that is running on a machine somewhere is actually a Kubernetes service. 
  - It is trivial to replace it with a database that is a Kubernetes service. 
  - In production, you may rely on your legacy database that is running on a machine, but for continuous testing you may deploy a test database as a container.

#### Representing both databases as Kubernetes services enables you to maintain identical configurations in both testing and production. 
High fidelity between test and production ensures that passing tests will lead to successful deployment in production.

To see how you maintain fidelity between development and production, remember that all Kubernetes objects are deployed into namespaces. Imagine that we have test and production namespaces defined. The test service is imported using an object like:

```
kind: Service
metadata:
    name: my-database
    # note 'test' namespace here
    namespace: test
...
```

The production service looks the same, except it uses a different namespace:

```
kind: Service
metadata:
    name: my-database
    # note 'prod' namespace here
    namespace: prod
...
```

#### When you deploy a Pod into the test namespace and it looks up the service named my-database
It will receive a pointer to my-database.test.svc.cluster.internal, which in turn points to the test database. 

#### When a Pod deployed in the prod namespace looks up the same name ( my-database ) 
It will receive a pointer to my-database.prod.svc.cluster.internal, which is the production database.

Thus, the same service name, in two different namespaces, resolves to two different services. 


### Services Without Selectors
When we introduced services, we talked about label queries and how they were used to identify the Pods.

#### External services, 
There is no such label query. Instead, you have a DNS that points to the server running the database. 

Example, let’s assume that this server is named database.company.com.

#### Import this external database service into Kubernetes
We start by creating a service without a Pod selector that references the DNS of the database server

```
kind: Service
apiVersion: v1
metadata:
    name: external-database
spec:
    type: ExternalName
    externalName: database.company.com
```

#### Typical Kubernetes service 
WHen it is created, an IP address is also created and the Kubernetes DNS service points to that IP address.

#### Service of type ExternalName
The Kubernetes DNS service points to the external name you specified (database.company.com in this case). 

When an application in the cluster does a DNS lookup for the hostname external-database.svc.default.cluster, the DNS protocol aliases that name to database.company.com. 

In this way, all containers in Kubernetes believe that they are talking to a service that is backed with other containers, when in fact they are being redirected to the external database.

#### And if you don’t have a DNS address for an external database service, just an IP address. 
In such cases, it is still possible to import this service as a Kubernetes service.

First, you create a Service without a label selector without the ExternalName type.

```
kind: Service
apiVersion: v1
metadata:
    name: external-ip-database
```

Given that this is an external service, the user is responsible for populating the endpoints manually with an Endpoints resource

```
kind: Endpoints
apiVersion: v1
metadata:
    name: external-ip-database
subsets:
    - addresses:
        - ip: 192.168.0.1
        ports:
        - port: 3306
```

If you have more than one IP address for redundancy, you can repeat them in the addresses array. 

Once the endpoints are populated, the load balancer will start redirecting traffic from your Kubernetes service to the IP address endpoint(s).


### Limitations of External Services: Health Checking
External services in Kubernetes have one significant restriction: they do not perform any health checking. The user is responsible for ensuring that the endpoint or DNS name supplied to Kubernetes is as reliable as necessary for the application.


## Running Reliable Singletons

#### The challenge of running storage solutions in Kubernetes 
 - ReplicaSet expect that every container is identical and replaceable
 - but for most storage solutions this isn’t the case. 

 - One option is to use Kubernetes primitives, but not replicate the storage. 
 - Instead, simply run a single Pod that runs the database. 
 - In this way the challenges of running replicated storage in Kubernetes don’t occur, since there is no replication.

#### This might seem to run counter to the principles of building reliable distributed systems
But in general, it is no less reliable than running your database or on a single virtual or physical machine. 

In reality, if you structure the system properly the only thing you are sacrificing is potential downtime for upgrades or
in case of machine failure. 

While for large-scale or mission-critical systems this may not be acceptable.
For smaller-scale applications this may be acceptable.

If this is not true for you, feel free to skip this section and either import existing services as described in the previous section, or move on to Kubernetes-native StatefulSets.


### Running a MySQL Singleton
 - How to run a reliable singleton instance of the MySQL as a Pod in Kubernetes;
 - How to expose that singleton to other applications in the cluster.

To do this, we are going to create three basic objects:

1) A persistent volume to manage the lifespan of the on-disk storage independently from the lifespan of the running MySQL application
2) A MySQL Pod that will run the MySQL application
3) A service that will expose this Pod to other containers in the cluster

### persistent volumes
It is a storage location that has a lifetime independent of any Pod or container. This is useful when the on-disk of a database should survive even if the containers running the database application crash, or move to different machines. If the application moves to a different machine, the volume should move with it, and data should be preserved.

To begin, we’ll create a persistent volume for our MySQL database to use. This example uses NFS for maximum portability.

```
apiVersion: v1
kind: PersistentVolume
metadata:
    name: database
    labels:
        volume: my-volume
spec:
    accessModes:
    - ReadWriteMany
    capacity:
        storage: 1Gi
    nfs:
        server: 192.168.0.1
        path: "/exports"
```

This defines an NFS PersistentVolume object with 1 GB of storage space.

We can create this persistent volume as usual with:

```
$ kubectl apply -f nfs-volume.yaml
```

Now that we have a persistent volume created, we need to claim that persistent volume for our Pod. We do this with a PersistentVolumeClaim object.

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
    name: database
spec:
    accessModes:
    - ReadWriteMany
    resources:
        requests:
            storage: 1Gi
    selector:
        matchLabels:
            volume: my-volume
```

The selector field uses labels to find the matching volume we defined previously.

Now that we’ve claimed our volume, we can use a ReplicaSet to construct our singleton Pod.

```
apiVersion: extensions/v1
kind: ReplicaSet
metadata:
    name: mysql
    # labels so that we can bind a Service to this Pod
    labels:
        app: mysql
spec:
    replicas: 1
    selector:
        matchLabels:
            app: mysql
    template:
        metadata:
            labels:
                app: mysql
        spec:
            containers:
            - name: database
                image: mysql
                resources:
                    requests:
                        cpu: 1
                        memory: 2Gi
                env:
                # Environment variables are not a best practice for security,
                # but we're using them here for brevity in the example.
                # See Chapter 11 for better options.
                - name: MYSQL_ROOT_PASSWORD
                    value: some-password-here
                livenessProbe:
                    tcpSocket:
                        port: 3306
                ports:
                - containerPort: 3306
                volumeMounts:
                    - name: database
                        # /var/lib/mysql is where MySQL stores its databases
                        mountPath: "/var/lib/mysql"
        volumes:
        - name: database
            persistentVolumeClaim:
                claimName: database

```

Once we create the ReplicaSet it will, in turn, create a Pod running MySQL using the persistent disk we originally created. The final step is to expose this as a Kubernetes service.

```
apiVersion: v1
kind: Service
metadata:
    name: mysql
spec:
    ports:
    - port: 3306
        protocol: TCP
    selector:
        app: mysql
```

Now we have a reliable singleton MySQL instance running in our cluster and exposed as a service named mysql , which we can access at the full domain name mysql.svc.default.cluster.


### Dynamic Volume Provisioning
With dynamic volume provisioning, the cluster operator creates one or more StorageClass objects.

Now you can refer to this storage class in your persistent volume claim, rather than referring to any specific persistent volume. When the dynamic provisioner sees this storage claim, it uses the appropriate volume driver to create the volume and bind it to your persistent volume claim.


## Kubernetes-Native Storage with StatefulSets
When Kubernetes was first developed, there was a heavy emphasis on homogeneity for all replicas in a replicated set. 

### No replica had an individual identity
While this approach provides a great deal of isolation for the orchestration system, it also makes it quite difficult to develop stateful applications. 

After significant input from the community StatefulSets were introduced in Kubernetes version 1.5.

### Properties of StatefulSets
StatefulSets are replicated groups of Pods, similar to ReplicaSets. But unlike a ReplicaSet, they have certain unique properties:

1) Each replica gets a persistent hostname with a unique index (e.g., database-0 ,database-1 , etc.).

2) Each replica is created in order from lowest to highest index, and creation will block until the Pod at the previous index is healthy and available.

3) When a StatefulSet is deleted, each of the managed replica Pods is also deleted in order from highest to lowest.

### StatefulSets are valuable for applications that require one or more of the following.

Stable, unique network identifiers.
Stable, persistent storage.
Ordered, automated rolling updates.

In the above, stable is synonymous with persistence across Pod (re)scheduling. If an application doesn't require any stable identifiers or ordered deployment, deletion, or scaling, you should deploy your application using a workload object that provides a set of stateless replicas. Deployment or ReplicaSet may be better suited to your stateless needs.