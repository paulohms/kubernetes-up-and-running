# Service Discovery
Kubernetes is a very dynamic system. The system is involved in placing Pods on nodes, making sure they are up and running, and rescheduling them as needed.

There are ways to automatically change the number of Pods based on load (such as horizontal Pod autoscaling [see “Autoscaling a ReplicaSet” on page 110]). The API- driven nature of the system encourages others to create higher and higher levels of automation.

While the dynamic nature of Kubernetes makes it easy to run a lot of things, it creates problems when it comes to finding those things. Most of the traditional network infrastructure wasn’t built for the level of dynamism that Kubernetes presents.

## What Is Service Discovery
The general name for this class of problems and solutions is service discovery. Service discovery tools help solve the problem of finding which processes are listening at which addresses for which services.

## The Service Object
Real service discovery in Kubernetes starts with a Service object.

A Service object is a way to create a named label selector. As we will see, the Service object does some other nice things for us, too.

Just as the kubectl run command is an easy way to create a Kubernetes deployment, we can use kubectl expose to create a service. Let’s create some deployments and services so we can see how they work:

```bash
kubectl create deployment alpaca-prod --image=gcr.io/kuar-demo/kuard-amd64:blue --replicas=3 --port=8080
kubectl expose deployment alpaca-prod

kubectl create deployment bandicoot-prod --image=gcr.io/kuar-demo/kuard-amd64:green --replicas=2 --port=8080
kubectl expose deployment bandicoot-prod

kubectl get services -o wide
```

After running these commands, we have three services. The kubernetes service is automatically created for you so that you can find and talk to the Kubernetes API from within the app.

If we look at the SELECTOR column, we see that the alpaca-prod service simply gives a name to a selector and specifies which ports to talk to for that service.

Furthermore, that service is assigned a new type of virtual IP called a cluster IP. This is a special IP address the system will load-balance across all of the Pods that are iden tified by the selector.

To interact with services, we are going to port forward to one of the alpaca Pods. Start and leave this command running in a terminal window. You can see the port forward working by accessing the alpaca Pod at http://localhost:48858:

```bash
ALPACA_POD=$(kubectl get pods -l app=alpaca-prod -o jsonpath='{.items[0].metadata.name}')
echo $ALPACA_POD
kubectl port-forward $ALPACA_POD 48858:8080
```

### Service DNS
Because the cluster IP is virtual, it is stable, and it is appropriate to give it a DNS address. All of the issues around clients caching DNS results no longer apply. Within a namespace, it is as easy as just using the service name to connect to one of the Pods identified by a service.

Kubernetes provides a DNS service exposed to Pods running in the cluster.

The Kubernetes DNS service provides DNS names for cluster IPs.

You can try this out by expanding the “DNS Query” section on the kuard server status page. Query the A record for alpaca-prod.

The full DNS name here is alpaca-prod.default.svc.cluster.local. . Let’s break this down:

alpaca-prod:

The name of the service in question.

default:

The namespace that this service is in.

svc:

Recognizing that this is a service. This allows Kubernetes to expose other types of things as DNS in the future.

cluster.local.:

The base domain name for the cluster. This is the default and what you will see for most clusters. Administrators may change this to allow unique DNS names across multiple clusters.


## Readiness Checks
Often, when an application first starts up it isn’t ready to handle requests. 

One nice thing the Service object does is track which of your Pods are ready via a readiness check

Let’s modify our deployment to add a readiness check that is attached to a Pod, as we discussed in Chapter 5:

```bash
kubectl edit deployment/alpaca-prod
```

This command will fetch the current version of the alpaca-prod deployment and bring it up in an editor.

Add the following section:

```bash
spec:
  ...
  template:
    ...
    spec:
      containers:
        ...
        name: alpaca-prod
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 2
          initialDelaySeconds: 0
          failureThreshold: 3
          successThreshold: 1
```

This sets up the Pods this deployment will create so that they will be checked for readiness via an HTTP GET to /ready on port 8080. This check is done every 2 seconds starting as soon as the Pod comes up. If three successive checks fail, then the Pod will be considered not ready. However, if only one check succeeds, the Pod will again be considered ready.

Updating the deployment definition like this will delete and recreate the alpaca Pods. As such, we need to restart our port-forward command from earlier:

```bash
ALPACA_POD=$(kubectl get pods -l app=alpaca-prod -o jsonpath='{.items[0].metadata.name}')
echo $ALPACA_POD
kubectl port-forward $ALPACA_POD 48858:8080
```

Point your browser to http://localhost:48858 and you should see the debug page for that instance of kuard . Expand the “Readiness Probe” section. You should see this page update every time there is a new readiness check from the system, which should happen every 2 seconds.

In another terminal window, start a watch command on the endpoints for the alpaca-prod service

```bash
kubectl get endpoints alpaca-prod --watch
```

Now go back to your browser and hit the “Fail” link for the readiness check. You should see that the server is now returning 500s. After three of these, this server is removed from the list of endpoints for the service. Hit the “Succeed” link and notice that after a single readiness check the endpoint is added back.

This readiness check is a way for an overloaded or sick server to signal to the system that it doesn’t want to receive traffic anymore.


## Looking Beyond the Cluster
So far, everything we’ve covered in this chapter has been about exposing services inside of a cluster. Oftentimes, the IPs for Pods are only reachable from within the cluster. At some point, we have to allow new traffic in!

The most portable way to do this is to use a feature called NodePorts

```bash
kubectl delete services,deployments -l app
kubectl create deployment alpaca-prod --image=gcr.io/kuar-demo/kuard-amd64:blue --replicas=3 --port=8080
kubectl expose deployment alpaca-prod --type=NodePort --name=ex-alpaca-prod-service --port=8080
kubectl get services
minikube service --url ex-alpaca-prod-service
```

Now we can hit any of our cluster nodes on that port to access the service. If you are sitting on the same network, you can access it directly. If your cluster is in the cloud someplace, you can use SSH tunneling with something like this:

$ ssh <node> -L 8080:localhost:32711

```bash
kubectl delete services,deployments -l app
```
