# Deployments

The Deployment object exists to manage the release of new versions. Deployments represent deployed applications in a way that transcends any particular version.
Additionally, deployments enable you to easily move from one version of your code to the next. This “rollout” process is specifiable and careful. It waits for a user configurable amount of time between upgrading individual Pods. It also uses health checks to ensure that the new version of the application is operating correctly, and
stops the deployment if too many failures occur.

The actual mechanics of the software rollout performed by a deployment is controlled by a deployment controller that runs in the Kubernetes cluster itself. This makes it easy to integrate deployments with numerous continuous delivery tools and services. Further, running server-side makes it safe to perform a rollout from places with poor or intermittent internet connectivity. Imagine rolling out a new version of your software from your phone while riding on the subway. Deployments make this possible and safe!

## Your First Deployment

A deployment can be represented as a declarative YAML object that provides the details about what you want to run. In the following case, the deployment is requesting a single instance of the kuard application
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: kuard
spec:
    selector:
        matchLabels:
            run: kuard
    replicas: 1
    template:
        metadata:
            labels:
                run: kuard
        spec:
            containers:
            - name: kuard
                image: gcr.io/kuar-demo/kuard-amd64:blue 
```

```bash 
kubectl create -f kuard-deployment.yaml
```

## Deployment Internals

Deployments manage ReplicaSets. As with all relationships in Kubernetes, this relationship is defined by labels and a label selector. You can see the label selector by looking at the Deployment object.

```bash 
kubectl get deployments kuard -o yaml
```

From this you can see that the deployment is managing a ReplicaSet with the label run=kuard.

```bash 
kubectl get replicasets --selector=run=kuard
```

Now let’s see the relationship between a deployment and a ReplicaSet in action. We can resize the deployment using the imperative scale command

```bash 
kubectl scale deployments kuard --replicas=2
```

Kubernetes is an online, self-healing system. The top-level Deployment object is managing this ReplicaSet. If you ever want to manage that ReplicaSet directly, you need to delete the deployment (remember to set --cascade to false, or else it will delete the ReplicaSet and Pods as well!)



