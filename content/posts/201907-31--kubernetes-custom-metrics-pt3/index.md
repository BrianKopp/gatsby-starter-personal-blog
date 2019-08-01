---
title: Kubernetes Custom Metrics (part 3)
subTitle: More custom metrics
category: kubernetes
cover: blueprint.jpg
---

In the previous articles, we set up a minikube cluster which scaled
our app based on our custom metric. This showed how to use a
non-standard metric to scale pods. In this article, we'll walk through
how to set up an app to scale based off **some other** metric.

The use case I'll outline is the following. Suppose you have a message
queue that has items that should be processed. The number of messages
that get put on the queue increases and decreases throughout the day,
and you'd like to scale your queue workers to respond to the message
queue size. In this case, you want to scale the pods based on
**the average number of messages per worker**, rather than just the
message queue size.

So what we want is a metric that represents our queue size, and then
we want to scale our pods based on the queue size, divided by the
number of workers running.

Due to an unfortunate bug, we will have to represent this metric
as an `external` custom metric, as opposed to just a custom `object`
metric. The bug prevents us from scaling with an *average value* target
for `object` metrics. Not to worry. Everything is fine, I just might have
preferred to not use the `external` metric type.

Ok, let's hop in.

## Metric Emitter

Let's make an app that will expose our custom metric. This app will
*watch our message queue* and report the size of the queue to prometheus.
For demo purposes, this app doesn't actually go out and query the
message queue API to get the message queue size. Instead, it exposes
a REST API which allows the *message queue size* to be set.

TODO Gist

Run this app using `npm run start`, and then hit it with curl.

```bash
curl localhost:3000/
curl localhost:3000/metrics
curl localhost:3000/value
curl -X PUT --header "Content-Type: application/json" \
     --data '{"value": 500}' \
     localhost:3000/value
```

Let's make two manifest files. One for the deployment, and one for the service.
In our previous articles, we set up prometheus to scrape node apps using
a custom ServiceMonitor resource definition. Check in prometheus to see
if our metric `custom_external_metric` shows up.

TODO Prometheus pic.

## Worker that Scales

We'll make a minimal node app which represents our queue worker. Since
it's not relevant to this article how exactly your message queue app
looks, my app basically logs some stuff and then sits around.

TODO Gist.

Next, we'll make a couple manifest files. The deployment and service manifests
are straightforward.

TODO Gists.

Next, let's take a look at the manifest for the horizontal pod autoscaler.

TODO Gist.

Here, we are specifying that our worker deployment should scale based off an
**external** metric, and that we will have an *average target value*, which means
that the metric will be divided by the number of replicas in our deployment,
and then compared to this target average value.

## Configuring prometheus-adapter

We didn't configure prometheus-adapter last time to use external metrics.
prometheus-adapter is capable of serving the `/apis/external.metrics.k8s.io`
api, but it needs to be enabled.

```bash
helm delete --purge prom-adpt
helm install -n prom-adpt --namespace monitoring \
    -f prom-adpt-value.yaml \
    stable/prometheus-adapter
```

TODO prom-adpt-value.yaml GIST.

Now you should be able to validate the `/apis/external.metrics.k8s.io`
endpoint using `kubectl get --raw /apis/external.metrics.k8s.io` and
see that the custom_external_metric item is being piped through.

## Confirm Autoscaling Works

TODO

## Conclusion

In this article, we walked through how to set up autoscaling of
a deployment based on some arbitrary,
external metric, averaged over the number of pods available.

Hopefully across the past several articles, you are able to find
some examples that help you accomplish your custom autoscaling
needs! Thanks for reading! Please reach out to me on the kubernetes
slack! I'd love to hear if you have questions or if this helped!
