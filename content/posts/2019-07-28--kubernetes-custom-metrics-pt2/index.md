---
title: Kubernetes Custom Metrics (part 2)
subTitle: All things custom
category: kubernetes
cover: 
---

In the previous article, I went over how to set up an application to expose
a custom metric that we want to use to scale the application, as well as
created kubernetes manifests which included a HorizontalPodAutoscaler to
scale the app up and down based on that metric. I left off with the app
running, but the HPA couldn't find the custom metric, because k8s doesn't
yet know about it. In this post, I'll go over setting up your minikube cluster
to use `prometheus` and `prometheus-adapter` to get the custom metric from
the app and to kubernetes' custom metrics API so the HPA can see the metric
and scale the application.

Let's hop in.

## Environment

I'll be using `minikube` in this post, for simplicity. Some things will be
different in a real-world cluster, but hopefully after going through this
article, you'll have the tools you need to diagnose why your real-world
cluster isn't working (if these exact steps don't work for your real-world cluster).

```bash
# If you want to start with a clean slate and delete your existing minikube cluster
minikube delete

# Start (or create) up your cluster
minikube start

# Make sure you're using your minikube context.
kubectl config current-context

# Set up helm. If you don't have it set up, then set it up.
helm init
```

## Prometheus Operator

Prometheus operator is a helm package that provides some syntatic sugar for
installing prometheus. It installs prometheus, grafana, alert-manager, and
a bunch of other metrics-related stuff. It's really a great tool to use in your
real-world cluster, so I'll use it here, even though it's a bit overkill for a
minikube install.

When you install prometheus-operator, it installs some custom
resource definitions (CRDs). After installing, you can provide new manifests
to kubernetes with `kind: Prometheus` and `kind: ServiceMonitor`, which helps
you tailor your prometheus installation(s) to your specific use case. In this
article, we'll keep things simple, but you should know that you get a lot of
bang for your buck by using prometheus-operator.

### Installation

Here is the helm install package for prometheus-operator. I'll be using mostly
the default configuration, with the exception of a few values.

```bash
helm install \
    -n prom-op \
    --set \
    --namespace monitoring \
    stable/prometheus-operator
```

Here, I'm installing prometheus-operator in the `monitoring` namespace. It's good
to organize your kubernetes stuff into meaningful namespaces.

### Validate the Install

Let's make sure the installation worked. Grab the URL to access your
prometheus service. If you're using minikube, it's

```
minikube service prometheusTODO --url --namespace monitoring
```

Open prometheus in your browser and search search for TODO, then click
Graph. All we're doing here is making sure that data is flowing from the
default services. We won't yet see our custom metrics, but that's what
we'll configure next.

### ServiceMonitor

Prometheus operator introduces the concept of service monitors. They work
basically like this: Prometheus is configured to select ServiceMonitors
based on labels. ServiceMonitors are configured to select Services based
on labels and namespaces. Pods are then identified via the Services, and
subsequently scraped for metrics. Check out this troubleshooting guide for
some super sweet explanation.

Our Prometheus instance is already configured, by default, to select all
ServiceMonitors which have the label TODO. By default prometheus-operator creates
a ServiceMonitor that Prometheus sees, but that ServiceMonitor isn't configured
to select our app's service. Let's make another ServiceMonitor which selects our
app's Service, and which is also discoverable by our Prometheus instance.

TODO ServiceMonitor.

### Check Prometheus

Our Prometheus pod has a container which should update it's configuration,
so we shouldn't have to do anything for Prometheus to pick up our new ServiceMonitor.
Let's open the Prometheus URL and check out it's configuration. We should be able
to see our new ServiceMonitor show up.

Now we should be able to see our cm_connection_count custom metric in the
query search. Let's make sure it's showing up!

Sweet, Prometheus is doing everything it needs to do. Next, we need to
get this Prometheus custom metric over to the custom metrics API so our
Horizontal Pod Autoscaler can see it.

## Prometheus Adapter

Prometheus adapter is a great plugin which connects the dots between
Prometheus and the kubernetes custom metrics API. Let's install it.

```bash
helm install -n prom-adpt \
    --set TODO \
    --namespace monitoring \
    stable/prometheus-adapter
```

### Verifying the Install

Let's make sure the installation is correct. You should be able to call
`kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1` and not get an
error message. OMG IF YOU'RE USING RANCHER LIKE I WAS, THIS IS NOT THE URL
YOU WILL USE.

### Customizing Prometheus Adapter

Prometheus adapter grabs a handful of metrics from Prometheus by default.
It doesn't grab everything because that could be overwhelming. Instead,
it ships with a default configuration which grabs some metrics. It won't
grab our metrics out of the box.

Prometheus adapter has a bit of a learning curve with its configuration.
Unfortunately that is hard to avoid since it's such a flexible tool. Let's
hop in.

TODO configure prometheus adapter.

### Confirm our Metrics

Finally, you should be able to call
`kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/default/namespace/pods/*`
and see our custom metric. You should be able to call
`kubectl get hpa` and see that the app's HorizontalPodAutoscaler can see
the current value.

## Conclusion

In these articles, we've set up an app which exposes a custom metric called
`cm_connection_count` over a `/metrics` endpoint, a HorizontalPodAutoscaler
which watches for this custom metric to scale the app, a `prometheus-operator`
installation with a default `Prometheus` setup, a `ServiceMonitor` to make our
app's `Service` discoverable by `Prometheus`, a `prometheus-adapter` installation
configured to select our custom metric from `Prometheus`.

There's a lot here, and unfortunately it's pretty complicated. This kubernetes
monster is super powerful, but has steep curves like this. I hope this post
has helped illuminate some of the concepts and configuration steps required to
make this sort of setup work for you in your cluster. You **can** do custom metrics,
it **will** take some learning, but it **probably** will be sweet when you're done.

I'd love to hear if this was helpful. If you have any questions, hit me up in the
kubernetes slack.
