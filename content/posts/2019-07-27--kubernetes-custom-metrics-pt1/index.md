---
title: Kubernetes Custom Metrics (part 1)
subTitle: All things custom
category: kubernetes
cover: 
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
`prometheus-adapter` comes to save the day. It's a drop-in plug in that takes
`prometheus` metrics and serves them in k8s via the `/apis/custom.metrics.k8s.io/v1beta1`
API, which HorizontalPodAutoscaler is happy to grab.

So what the heck? You're new to kubernetes, but you want to scale
using something other than CPU or memory, and I'm all these custom API names,
prometheus, prometheus-adapter, all the things. Why is this so hard? Buckle up and
deal with it. That's what's required, and that's where we're going.

But first, let's get our apps up and running.

## Example App - Socket-IO Server

`prometheus-adapter` and others have some great how-to's for
scaling based on http requests per second, so I'm not going to reinvent that wheel.
You can go read those, and you should. Seriously, the documentation is pretty dang
good, and you need to read it to understand what the flip you're doing.

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
in *prometheus* format (go read about it). Later, we'll set up our autoscaler to
scale the number of pods in this deployment to target an average value for this
number of connections.

I mentioned that `socket-io` in node thinks it's fun to blow up if it has
to serve too many connections. Super fun. I've got logic in the app to
simulate disconnecting socket clients after it reaching a maximum connection
limit. Read more about that in the readme of the app. Link TODO.

Gist TODO.

You can see that the `/metrics` endpoint exposes a metric called
`cm_connection_count`. I've prefixed the name with `cm_`. I'll use this
prefix later to configure `prometheus-adapter` to grab it from `prometheus`.
This will let me set up `prometheus-adapter` with a generic set of selectors
that will work for all my custom metrics, without having to update my configuration
each time I want to add a new metric. Instead, I can just later expose a
metric called `cm_foo`, and have it come through automatically.

Make a Dockerfile.

Gist TODO.

### Manifests

Ok, manifests. You've read the k8s docs. You know what they are. If you don't,
you should stop reading this and go read k8s docs.

Deployment gist TODO.

Service gist TODO.

Horizontal pod autoscaler gist TODO.

Deploy these three manifests, and confirm the pod is running.
Make sure you can access the service, either via minikube, or proxy your
service using kubectl. Request a `GET /connection` a few times and see
how the number increments. Request it more than 20 times, and you'll see
how it starts simulating closing client connections as a defense
mechanism.

Check the horizontal pod autoscaler. You'll see it has no idea what
the actual value is. That's because it's looking for the `/apis/custom.metrics.k8s.io`
API, which we haven't yet brought to the table. We'll get `prometheus-adapter`
up and running in the next post to serve that API with data from `prometheus`.
