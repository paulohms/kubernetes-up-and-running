# Extending Kubernetes
From the beginning, it was clear that Kubernetes was going to be more than its core set of APIs; once an application is orchestrated within the cluster, there are countless other useful tools and utilities that can be represented and deployed as API objects in the Kubernetes cluster. 


#### The challenge
How to embrace this explosion of objects and use cases without having an API that sprawled without bound.

#### To resolve 
To resolve this tension between extended use cases and API sprawl, significant effort was put into making the Kubernetes API extensible. 

#### WHat is extensibility 
Extensibility meant that cluster operators could customize their clusters with the additional components that suited their needs.

This extensibility enables people to augment their clusters themselves, consume community-developed cluster add-ons, and even develop extensions that are bundled and sold in an ecosystem of cluster plug-ins.

Regardless of whether you are building your own extensions or consuming operators from the ecosystem, understanding how the Kubernetes API server is extended and how extensions can be built and delivered is a key component to unlocking the complete power of Kubernetes and its ecosystem. 


### What It Means to Extend Kubernetes
It adds new functionality to a cluster or limits the ways that users can interact with their clusters.

There is a rich ecosystem of plug-ins that cluster administrators can use to add additional services and capabilities to their clusters.

#### It’s important noting that extending the cluster is a very high-privilege thing to do. 
Not everyone should do, because cluster administrator privileges are required to extend a cluster. 

Even cluster administrators should be careful when installing third-party tools. Some extensions, like admission controllers, can be used to view all objects being created in the cluster, and could easily be used to steal secrets or run malicious code.


## Patterns for Custom Resources

Not all custom resources are identical. There are a variety of different reasons for extending the Kubernetes API surface area, and the following sections discuss some general patterns you may want to consider.

### Just Data
 - The easiest pattern for API extension is the notion of “just data.”
 - In this pattern, you are simply using the API server for storage and retrieval of information for your application.

An example use case for the “just data” pattern might be configuration for canary deployments of your application
 - for example, directing 10% of all traffic to an experimental backend. While in theory such configuration information could also be stored in a ConfigMap, ConfigMaps are essentially untyped, and sometimes using a more strongly typed API extension object provides clarity and ease of use.


### Compilers
A more complicated pattern is the “compiler” or “abstraction” pattern. In this pattern the API extension object represents a higher-level abstraction that is “compiled” in a combination of lower-level Kubernetes objects. 


### Operators
While compiler extensions provide easy-to-use abstractions, extensions that use the “operator” pattern provide online, proactive management of the resources created by the extensions.

These extensions provide a higher-level abstraction (for example, a database) that is compiled to a lower-level representation, but they also provide online functionality, such as snapshot backups of the database, or upgrade notifications when a new version of the software is available. 

To achieve this, the controller not only monitors the extension API to add or remove things as necessary, but also monitors the running state of the application supplied by the extension (e.g., a database) and takes actions to remediate unhealthy databases, take snapshots, or restore from a snapshot if a failure occurs. 