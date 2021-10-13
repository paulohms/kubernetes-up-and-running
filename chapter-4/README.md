# Common kubectl Commands

The kubectl command-line utility is a powerful tool, and in the following chapters you will use it to create objects and interact with the Kubernetes API. Before that, however, it makes sense to go over the basic kubectl commands that apply to all Kubernetes objects.

## Namespaces

Kubernetes uses namespaces to organize objects in the cluster. You can think of each namespace as a folder that holds a set of objects. By default, the kubectl command-line tool interacts with the default namespace. If you want to use a different namespace, you can pass kubectl the `--namespace` flag. For example, kubectl `--namespace=mystuff` references objects in the mystuff namespace. If you want to interact with all namespaces —for example, to list all Pods in your cluster— you can pass the `--all-namespaces` flag.

## Contexts

If you want to change the default namespace more permanently, you can use a context. This gets recorded in a kubectl configuration file, usually located at *$HOME/.kube/config*. This configuration file also stores how to both find and authenticate to your cluster. For example, you can create a context with a different default namespace for your kubectl commands using:

```bash
$ kubectl config set-context my-context --namespace=mystuff
```

This creates a new context, but it doesn’t actually start using it yet. To use this newly created context, you can run:

```bash
$ kubectl config use-context my-context
```

Contexts can also be used to manage different clusters or different users for authenticating to those clusters using the --users or --clusters flags with the set-context command.

## Viewing Kubernetes API Objects

Everything contained in Kubernetes is represented by a RESTful resource. Throughout this book, we refer to these resources as Kubernetes objects. Each Kubernetes object exists at a unique HTTP path; for example, https://your-k8s.com/api/v1/namespaces/default/pods/my-pod leads to the representation of a Pod in the default namespace named my-pod. The kubectl command makes HTTP requests to these URLs to access the Kubernetes objects that reside at these paths.

The most basic command for viewing Kubernetes objects via kubectl is get . If you run kubectl get *<resource-name>* you will get a listing of all resources in the current namespace. If you want to get a specific resource, you can use kubectl get *<resource-name> <obj-name>*.

Another common task is extracting specific fields from the object. kubectl uses the JSONPath query language to select fields in the returned object. The complete details of JSONPath are beyond the scope of this chapter, but as an example, this command will extract and print the IP address of the specified Pod:

```bash
$ kubectl get pods my-pod -o jsonpath --template={.status.podIP}
```

## Creating, Updating, and Destroying Kubernetes Objects

Objects in the Kubernetes API are represented as JSON or YAML files. These files are either returned by the server in response to a query or posted to the server as part of an API request. You can use these YAML or JSON files to create, update, or delete objects on the Kubernetes server.
Let’s assume that you have a simple object stored in obj.yaml. You can use kubectl to create this object in Kubernetes by running:

```bash
$ kubectl apply -f obj.yaml
```

If you want to see what the apply command will do without actually making the changes, you can use the `--dry-run` flag to print the objects to the terminal without actually sending them to the server.

The apply command also records the history of previous configurations in an annotation within the object. You can manipulate these records with the edit-last-applied , set-last-applied , and view-last-applied commands. For example:

```bash
$ kubectl apply -f myobj.yaml view-last-applied
```

When you want to delete an object, you can simply run:

```bash
$ kubectl delete -f obj.yaml
```

It is important to note that kubectl will not prompt you to confirm the deletion. Once you issue the command, the object will be deleted.
Likewise, you can delete an object using the resource type and name:

```bash
$ kubectl delete <resource-name> <obj-name>
```

## Labeling and Annotating Objects

Labels and annotations are tags for your objects. We’ll discuss the differences in Chapter 6, but for now, you can update the labels and annotations on any Kubernetes object using the annotate and label commands. For example, to add the color=red label to a Pod named bar , you can run:

```bash
$ kubectl label pods bar color=red
```

By default, label and annotate will not let you overwrite an existing label. To do this, you need to add the `--overwrite` flag.

If you want to remove a label, you can use the *<label-name>-* syntax:

```bash
$ kubectl label pods bar color-
```

## Debugging Commands

kubectl also makes a number of commands available for debugging your containers. You can use the following to see the logs for a running container:

```bash
$ kubectl logs <pod-name>
```

If you have multiple containers in your Pod, you can choose the container to view using the `-c` flag.

You can also use the exec command to execute a command in a running container:

```bash
$ kubectl exec -it <pod-name> -- bash
```

This will provide you with an interactive shell inside the running container so that you can perform more debugging.

If you don’t have bash or some other terminal available within your container, you can always attach to the running process:

```bash
$ kubectl attach -it <pod-name>
```

You can also copy files to and from a container using the cp command:

```bash
$ kubectl cp <pod-name>:</path/to/remote/file> </path/to/local/file>
```

If you want to access your Pod via the network, you can use the port-forward command to forward network traffic from the local machine to the Pod. This enables you to securely tunnel network traffic through to containers that might not be exposed anywhere on the public network. For example, the following command:

```bash
$ kubectl port-forward <pod-name> 8080:80
```

Finally, if you are interested in how your cluster is using resources, you can use the top command to see the list of resources in use by either nodes or Pods. This command:

```bash
kubectl top nodes
```

will display the total CPU and memory in use by the nodes in terms of both absolute units (e.g., cores) and percentage of available resources (e.g., total number of cores). Similarly, this command:

```bash
kubectl top pods
```

## Alternative Ways of Viewing Your Cluster

In addition to kubectl , there are other tools for interacting with your Kubernetes cluster.

For example, there are plug-ins for several editors that integrate Kubernetes and the editor environment, including:
* Visual Studio Code
* IntelliJ
* Eclipse