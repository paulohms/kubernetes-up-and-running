# Deploying Real-World Applications1

The previous chapters described a variety of API objects that are available in a Kubernetes cluster and ways in which those objects can best be used to construct reliable distributed systems. However, none of the preceding chapters really discussed how you might use the objects in practice to deploy a complete, real-world application. That is the focus of this chapter

We’ll take a look at four real-world applications:
    • Jupyter, an open source scientific notebook
    • Parse, an open source API server for mobile applications
    • Ghost, a blogging and content management platform
    • Redis, a lightweight, performant key/value store

## Jupyter

The Jupyter Project is a web-based interactive scientific notebook for exploration and visualization. It is used by students and scientists around the world to build and explore data and data visualizations. Because it is both simple to deploy and interesting to use, it’s a great first service to deploy on Kubernetes.
We begin by creating a namespace to hold the Jupyter application:

```bash
kubectl create namespace jupyter
```

And then create a deployment of size one with the program itself:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: jupyter
  name: jupyter
  namespace: jupyter
spec:
  replicas: 1
  selector:
    matchLabels:
      run: jupyter
  template:
    metadata:
      labels:
        run: jupyter
    spec:
      containers:
      - image: jupyter/scipy-notebook:abdb27a6dfbb
        name: jupyter
      dnsPolicy: ClusterFirst
      restartPolicy: Always
 ```

```bash
  kubectl create -f jupyter.yaml
```

```bash
watch kubectl get pods --namespace jupyter
```

Once the Jupyter container is up and running, you need to obtain the initial login token. You can do this by looking at the logs for the container:

```bash
kubectl logs --namespace jupyter ${pod_name}
``` 

You should then copy the token (it will look something like /?token=0195713c8e65088650fdd8b599db377b7ce6c9b10bd13766).

Next, set up port forwarding to the Jupyter container:

```bash 
kubectl port-forward -n jupyter ${pod_name} 8888:8888
```

Finally, you can visit http://localhost:8888/?token=<token>, inserting the token that you copied from the logs earlier. You should now have the Jupyter dashboard loaded in your browser. You can find tutorials to get oriented to Jupyter if you are so inclined on the Jupyter project site.

## Parse

### requiriments 

```bash
 kubectl apply -f mongo-simple.yaml
 ```