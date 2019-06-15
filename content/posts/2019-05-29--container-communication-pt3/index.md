---
title: Communication Between Containers (pt. 3)
subTitle: Making services which talk to each other
category: "containers"
cover: container.jpg
---

This post is the last part of a three article series
on making containers communicate with one another, especially
in the context of kubernetes. In this last article,
we'll be deploying our *Foo* and *Bar* apps onto Minikube
and letting them interact with one another using services.
It is assumed you have a kubernetes setup and are of the basic
concepts of k8s. There are plenty of great resources for
getting started with k8s, so there's no need to reinvent the
wheel on that. Let's go ahead and get started.

The source code for this article can be found in
[its github repo](https://github.com/briankopp/container-communication).

## Setting up the Environment

If you're working on minikube, it may be helpful to segregate
this project into its own namespace to keep track of resources.

Make sure your minikube system is up and running with
`minikube start`. Then check your namespaces using
`kubectl get namespaces --show-labels`. Right now, I don't have
a namespace for this project, so let's make one. Create a new
file in your project root called `namespace.yaml`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: container-communication
  labels:
    name: container-communication
```

Now let's add it.

```bash
kubectl create -f namespace.yaml
```

Now when you call `kubectl get namespaces --show-labels`, you should
see your new namespace. Let's set up the kubectl context
to use this namespace by default for the duration of this article.

```bash
kubectl config set-context $(kubectl config current-context) --namespace=container-communication
```

When we're all done, you'll want to switch the current context back to default.

```bash
kubectl config set-context $(kubectl config current-context) --namespace=default
```

## Create Bar Service

Ok, let's get started by making the bar service. We'll need to create
a kubernetes deployment object in file `bar/bar-deployment.yaml` to
describe the pods.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bar-deployment
  namespace: container-communication
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bar
  template:
    metadata:
      labels:
        app: bar
    spec:
      containers:
      - name: bar
        image: briankopp/container-communication-bar:1.0.0
        ports:
        - containerPort: 3000
```

Let's throw this on the cluster with `kubectl apply -f bar/bar-deployment.yaml`.

Next, we'll want to expose the pods using a service. Make a new file
called `bar/bar-service.yaml`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bar-service
  namespace: container-communication
  labels:
    service: bar
spec:
  ports:
  - port: 3000
    protocol: TCP
  selector:
    app: bar
  type: NodePort
```

Make sure it's of type NodePort so it's publicly available. We'll
change this later to make *Bar* a private service, but for now,
let's make sure we can hit the service. Apply this with
`kubernetes apply -f bar/bar-service.yaml`.

Next, you'll need to get the URL to your cluster. If you're using
minikube, you can execute the following command to get the
URL to your service.

```bash
minikube service bar-service --url --namespace=container-communication
```

In my local setup, my service's URL is `http://192.168.99.100:32267`. If
I hit that URL using `curl 192.168.99.100:32267`, I see the response
`Hello from BAR`.

Great, it's working now! Now we're going to make it a *private* service,
one that can't be accessed outside the cluster. Remove the `type: NodePort`
line from the `bar/bar-service.yaml` file. Now, in order to switch a service from
NodePort to ClusterIp in Kubernetes, you either have to delete and re-create the
serivce, or force the change. We'll do the latter, executing the following command
`kubectl apply -f bar/bar-service.yaml --force`.

Now, when you try to get the service's IP address, you won't be able to.

```bash
minikube service bar-service --url --namespace=container-communication
# empty output
```

## Create the Foo Service

Moving on to the foo service. Let's create the foo deployment and service files.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: foo-deployment
  namespace: container-communication
spec:
  replicas: 1
  selector:
    matchLabels:
      app: foo
  template:
    metadata:
      labels:
        app: foo
    spec:
      containers:
      - name: foo
        image: briankopp/container-communication-foo:1.0.0
        ports:
        - containerPort: 3000
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: foo-service
  namespace: container-communication
  labels:
    service: foo
spec:
  ports:
  - port: 3000
    protocol: TCP
  selector:
    app: foo
  type: NodePort
```

Apply these two files and get the foo service's IP address.

```bash
kubectl apply -f foo/foo-deployment.yaml
kubectl apply -f foo/foo-service.yaml
minikube service foo-service --url --namespace=container-communication
```

In my setup, the URL is `http://192.168.99.100:31611`. Hit it using
`curl 192.168.99.100:31611`. You should get an error message saying
`error in BAR request`. Let's check out what's happening by inspecting
the pod's logs. Run `kubectl get pods`.

```text
NAME                              READY   STATUS    RESTARTS   AGE
bar-deployment-dd858597d-n6zc7    1/1     Running   0          10m
foo-deployment-6866fdcb49-59fzj   1/1     Running   0          109s
```

Note the foo-deployment pod name and then run
`kubectl logs foo-deployment-6866fdcb49-59fzj`.
I see the following output:

```text
> foo@1.0.0 start /usr/app
> node index.js

express "foo" app listening on port 3000
Error: options.uri is a required argument
    at Request.init (/usr/app/node_modules/request/request.js:231:31)
    at new Request (/usr/app/node_modules/request/request.js:127:8)
    at request (/usr/app/node_modules/request/index.js:53:10)
    at Function.get (/usr/app/node_modules/request/index.js:61:12)
    at app.get (/usr/app/app.js:6:13)
    at Layer.handle [as handle_request] (/usr/app/node_modules/express/lib/router/layer.js:95:5)
    at next (/usr/app/node_modules/express/lib/router/route.js:137:13)
    at Route.dispatch (/usr/app/node_modules/express/lib/router/route.js:112:3)
    at Layer.handle [as handle_request] (/usr/app/node_modules/express/lib/router/layer.js:95:5)
    at /usr/app/node_modules/express/lib/router/index.js:281:22
```

It's complaining about the uri not being supplied. Remember in
the previous article when we ran the apps using docker-compose,
we passed in the BAR_URL environment variable to the foo service.
Let's make sure we add that to the `foo/foo-deployment.yaml` file.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: foo-deployment
  namespace: container-communication
spec:
  replicas: 1
  selector:
    matchLabels:
      app: foo
  template:
    metadata:
      labels:
        app: foo
    spec:
      containers:
      - name: foo
        image: briankopp/container-communication-foo:1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: BAR_URL
          value: http://bar-service.container-communication.svc.cluster.local:3000
```

Here, the internal ClusterIP service `bar-service` can be reached using
the DNS `http://bar-service.container-communication.svc.cluster.local:3000`, following
the pattern `service-name.namespace-name.svc.cluster.local:<PORT>`.

Apply and try again. Now you should get the `Hello from BAR from inside FOO`
response!

## Conclusion

In this article, we walked through creating two services on a minikube cluster.
One service was a ClusterIP internal-only service, while the other service was
a NodePort service allowing external connections. We made sure the external-facing
service could still communicate with the internal-only service, and configured
communication using the canonical service DNS within the kubernetes cluster.

Clean up your resources by running `kubectl delete namespace container-communication`.

Thanks for stopping by!
