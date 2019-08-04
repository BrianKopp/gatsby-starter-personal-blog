---
title: Kubernetes Custom Metrics (part 1)
subTitle: All things custom
category: kubernetes
cover: blueprint.jpg
---

I've been digging a lot into kubernetes at work, which has been a lot of fun.
What's been less than fun, though, is a few long days of face smashing trying to
figure out how to do some custom things. However, some hills have been taken,
some battles have been won, and I have lived to tell the tale. I want to capture
my lessons learned and provide a how-to on some things that are less than standard.

Autoscaling. It's a thing. You *probably* need to be doing it in production. But what
if you don't want to scale based on CPU or memory? Something else? Maybe http requests
per second? Enter: kubernetes custom metrics. Cool thing, kubernetes only defines the
API, but doesn't implement it. So if you want your autoscalers to scale your pods
based on custom metrics, it's a Bring-Your-Own-API sort of affair. Fortunately,
[prometheus-adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter)
comes to save the day. It's a plug in that takes
[prometheus](https://prometheus.io/)
metrics and serves them in k8s via the `/apis/custom.metrics.k8s.io/v1beta1`
API, which HorizontalPodAutoscaler is happy to grab.

So what the heck? You're new to kubernetes, but you want to scale
using something other than CPU or memory, and I'm all these custom API names,
prometheus, prometheus-adapter, all the things. Why is this so hard? Buckle up and
deal with it. That's what's required, and that's where we're going.

But first, let's get our app up and running. Follow along in my
[github repository](https://github.com/BrianKopp/kubernetes-custom-metrics)
with the example code for these posts.

Parts:

* Part 1 - introduction (this post)
* [Part 2 - prometheus configuration](https://blog.codekopp.com/kubernetes-custom-metrics-pt2/)
* [Part 3 - (sort of) external metrics](https://blog.codekopp.com/kubernetes-custom-metrics-pt3/)

## Example App - Socket-IO Server

prometheus-adapter and others have some great how-to's for
scaling based on
[http requests per second](https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/walkthrough.md),
so I'm not going to reinvent that wheel.
You can go read those, and you should. Seriously,
[the documentation](https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config.md)
is pretty dang good, and you need to read it to understand what the flip you're doing.

Instead, I'm going to show a slightly different take on their example apps and
make something different. I'll pretend I need to serve socket-io connections
from 1-N docker containers. Since node sometimes blows up and kicks off a super
fun death spiral if you give it too many socket-io connections to maintain, we'll
set up our app to scale based on the number of connections it has. This way,
our node services don't get overwhelmed, and they can be stable and healthy.

This is similar to the http per second example, but will let me show how
to take those https req/s examples and tailor them to do exactly what
I need them to do.

Let's jump in.

### The Node App

We'll make a basic express app that will simulate what our servers will experience
serving socket-io connections. `GET /connection` will increment our
app's # of connections it is *having to serve*. It will hold onto a
`connectionCount`, and serve that **metric** up through a `/metrics` endpoint,
in *prometheus* format
([go read about it](https://github.com/prometheus/docs/blob/master/content/docs/instrumenting/exposition_formats.md)).
Later, we'll set up our autoscaler to
scale the number of pods in this deployment to target an average value for this
number of connections.

I mentioned that `socket-io` in node thinks it's fun to blow up if it has
to serve too many connections. Super fun. I've got logic in the app to
simulate disconnecting socket clients after it reaching a maximum connection
limit. Read more about that in the
[readme of my sample app](https://github.com/BrianKopp/kubernetes-custom-metrics/tree/master/examples/connections-metric).

`gist:68bacf0baad08fd027ad4da1e4cd182e#index.js`

You can see that the `/metrics` endpoint exposes a metric called
`connection_count`. This metric will be picked up by prometheus, and
then later prometheus-adapter, which will serve it via the kubernetes
custom metrics API for our horizontal-pod-autoscaler.

Make a Dockerfile.

`gist:68bacf0baad08fd027ad4da1e4cd182e#Dockerfile`

## Set Up Minikube

I'll be using `minikube` in this post for simplicity. Some things will be
different in a real-world cluster, but hopefully after going through this
article, you'll have the tools you need to diagnose why your real-world
cluster isn't working (if these exact steps don't work for your real-world cluster).

```bash
# If you want to start with a clean slate and delete your existing minikube cluster
minikube delete

# Start (or create) up your cluster
minikube start # optional "--memory 4096 --cpus 4" for more juice

# Make sure you're using your minikube context.
kubectl config current-context

# Set up helm. If you don't have it set up, then set it up.
helm init
```

## Deploy Sample App

Ok, manifests. You've read the k8s docs. You know what they are. If you don't,
you should stop reading this and go read k8s docs.

`gist:68bacf0baad08fd027ad4da1e4cd182e#deployment.yaml`

`gist:68bacf0baad08fd027ad4da1e4cd182e#service.yaml`

`gist:68bacf0baad08fd027ad4da1e4cd182e#horizontal-pod-autoscaler.yaml`

```bash
kubectl apply -f examples/connections-metric/manifests/deployment.yaml
kubectl apply -f examples/connections-metric/manifests/service.yaml
kubectl apply -f examples/connections-metric/manifests/horizontal-pod-autoscaler.yaml
```

Confirm the sample app pod is running. Make sure you can access
the service. Grab its URL using
`minikube service connections-metric-svc --url`.
Run the following command to start hitting the app with "connections".

```bash
for run in {1..60}; do curl <URL>/connection && sleep 1; done
```

You should see how the number doesn't go too much higher
than 20. That is because the app is simulating closing connections
as a defense mechanism.

## Wrapping Up This Post

Check the horizontal pod autoscaler using `kubectl get hpa`.
You'll see it has no idea what
the actual value is. That's because it's looking for the `/apis/custom.metrics.k8s.io`
API, which we haven't yet brought to the table. We'll get `prometheus-adapter`
up and running in the
[next post](https://blog.codekopp.com/kubernetes-custom-metrics-pt1/)
to serve that API with data from `prometheus`.
