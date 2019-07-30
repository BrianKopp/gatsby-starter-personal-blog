---
title: Kubernetes Custom Metrics (part 2)
subTitle: All things custom
category: kubernetes
cover: blueprint.jpg
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

## Prometheus Operator

[Prometheus operator](https://github.com/coreos/prometheus-operator)
is a helm package that provides some syntatic sugar for
installing prometheus. It installs prometheus, grafana, alert-manager, and
a bunch of other metrics-related stuff. It's really a great tool to use in your
real-world cluster, so I'll use it here, even though it's a bit overkill for a
minikube install.

When you install prometheus-operator, it installs
[some custom resource definitions (CRDs)](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md#related-resources).
After installing, you can provide new manifests
to kubernetes with `kind: Prometheus` and `kind: ServiceMonitor`, which helps
you tailor your prometheus installation(s) to your specific use case. In this
article, we'll keep things simple, but you should know that you get a lot of
bang for your buck by using prometheus-operator.

### Installation

Here is the [helm install](https://github.com/helm/charts/tree/master/stable/prometheus-operator)
package for prometheus-operator. I'll be using mostly
the default configuration, with the exception of a few values.

```bash
helm install \
    -n prom-op \
    --set prometheus.service.type=NodePort \
    --namespace monitoring \
    stable/prometheus-operator
```

Here, I'm installing prometheus-operator in the `monitoring` namespace. It's good
to organize your kubernetes stuff into meaningful namespaces.

### Validate the Install

Let's make sure the installation worked. Grab the URL to access your
prometheus service. If you're using minikube, it's
`minikube service prom-op-prometheus-operato-prometheus --url --namespace monitoring`.
This could take a minute or two to show up.

Open prometheus in your browser and search search for `:node_memory_utilisation:`, then click
Execute and then Graph. All we're doing here is making sure that data is flowing from the
default services. We won't yet see our custom metrics, but that's what
we'll configure next.

### ServiceMonitor

Prometheus operator introduces the concept of service monitors. They work
basically like this: Prometheus is configured to select ServiceMonitors
based on labels. ServiceMonitors are configured to select Services based
on labels and namespaces. Pods are then identified via the Services, and
subsequently scraped for metrics. Check out
[this troubleshooting guide](https://github.com/coreos/prometheus-operator/blob/master/Documentation/troubleshooting.md#overview-of-servicemonitor-tagging-and-related-elements)
for some super sweet explanation.

Our Prometheus instance is already configured, by default, to select all
ServiceMonitors which have the label `release=prom-op`. You can inspect that via
`kubectl get prometheus -n monitoring -o yaml`, inspecting the `serviceMonitorSelector`
key. By default prometheus-operator creates several default ServiceMonitors that
Prometheus sees, but those ServiceMonitors aren't configured
to select our app's service. Let's make another ServiceMonitor which selects our
app's Service, and which is also discoverable by our Prometheus instance.

`gist:68bacf0baad08fd027ad4da1e4cd182e#service-monitor.yaml`

### Check Prometheus

Our Prometheus pod has a container which should update it's configuration,
so we shouldn't have to do anything for Prometheus to pick up our new ServiceMonitor.
Let's open the Prometheus URL and check out it's configuration under
Status->Configuration. We should be able to see our new
ServiceMonitor show up after a few minutes.

Now we should be able to see our `connection_count` custom metric in the
query search. Let's make sure it's showing up!

Sweet, Prometheus is doing everything it needs to do. Next, we need to
get this Prometheus custom metric over to the custom metrics API so our
Horizontal Pod Autoscaler can see it.

## Prometheus Adapter

[Prometheus adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter)
is a great plugin which connects the dots between
Prometheus and the kubernetes custom metrics API. Let's install it.

```bash
helm install -n prom-adpt \
    --namespace monitoring \
    --set prometheus.url=http://prom-op-prometheus-operato-prometheus.monitoring.svc \
    stable/prometheus-adapter
```

### Verifying the Install

Let's make sure the installation is correct. You should be able to call
`kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1` and see a boatload of
output.

RANCHER USERS READ THIS!

Read [this page](https://rancher.com/docs/rancher/v2.x/en/k8s-in-rancher/horitzontal-pod-autoscaler/manage-hpa-with-kubectl/),
especially the last part. The URL above is not the URL you will request
using kubectl. You have just saved many hours of self-hate and frustration.
Congratulations and you're welcome.

### Customizing Prometheus Adapter

Prometheus adapter grabs a handful of metrics from Prometheus by default.
It doesn't grab everything because that could be overwhelming. Instead,
it ships with a default configuration which grabs some metrics. It won't
grab our metrics out of the box.

Prometheus adapter has a bit of a learning curve with its configuration.
Unfortunately that is hard to avoid since it's such a flexible tool. Let's
hop in.

Let's check out the prometheus adapter configuration. Grab it using
`kubectl get cm prom-adpt-prometheus-adapter -n monitoring -o yaml > prometheus-adapter-config-old.yaml`.
Let's update the rules list by adding the following rule. Read
[this doc](https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config.md)
to understand what's going on.

`gist:68bacf0baad08fd027ad4da1e4cd182e#prometheus-adapter-new-rule.yaml`

Apply your new good config, `prometheus-adapter-config-good.yaml` using
`kubectl apply -f prometheus-adapter-config-good.yaml`.

### Confirm our Metrics

After a few minutes, you should be able to call
`kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/default/namespace/pods/*/connection_count`
and see our custom metric. You should be able to call
`kubectl get hpa` and see that the app's HorizontalPodAutoscaler can see
the current value.

## Conclusion

In these articles, we've set up an app which exposes a custom metric called
`connection_count` over a `/metrics` endpoint, a HorizontalPodAutoscaler
which watches for this custom metric to scale the app, a `prometheus-operator`
installation with a default `Prometheus` setup, a `ServiceMonitor` to make our
app's `Service` discoverable by `Prometheus`, and a `prometheus-adapter` installation
configured to select our custom metric from `Prometheus`.

There's a lot here, and unfortunately it's pretty complicated. This kubernetes
monster is super powerful, but has steep curves like this. I hope this post
has helped illuminate some of the concepts and configuration steps required to
make this sort of setup work for you in your cluster. You **can** do custom metrics,
it **will** take some learning, but it **probably** will be sweet when you're done.

I'd love to hear if this was helpful. If you have any questions, hit me up in the
kubernetes slack.
