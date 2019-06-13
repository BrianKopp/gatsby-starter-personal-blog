---
title: Kubernetes Done Cheap
subTitle: Surprisingly cheap on GKE
cover: gke.png
category: kubernetes
---

I've been wanting to set up my own personal kubernetes
cluster running in the cloud, but I have always been dissuaded by
the costs associated with it. On AWS if you want to run EKS,
that's a minimum of about 70 smackers a month, plus compute.
Of course if you have a commercial offering, this isn't that big of
a deal, but for a personal sandbox, it's a big price tag!

If you run your own EC2 instances and configure your own cluster,
that is less than ideal since you have to configure everything on
your own. At that point, you're talking about a minimum of $20 or so
for a few EC2 instances.

I came across a few articles talking about how GKE can be really cheap
to set up a small kubernetes cluster. I'll outline in this article how
I set it up.

## Prerequisites

You'll need the `gcloud` cli installed and set up on your computer.
You'll also need to have a google cloud project initialized. I called
mine `cheap-kubernetes`. You will need `kubectl` installed. If you don't
already have it installed, you can use gcloud to install it using
`gcloud components install kubectl`.

## Outline

I like to do most of my commands via the command line. I'll perform the
following steps.

* Create my cluster
* Access cluster using kubectl
* Deploy first container
* Set up nginx for HTTP proxying

## Creating the Cluster

We're going to create a 1-node auto-scaling cluster using `n1-standard-1`
node types. They have 1 vCPU and 3.75GB of memory, which is well-suited
for running several containers. You could also chooes `f1-micro` or `g1-small`
types, which have lower memory requirements. I will likely have a few memory-heavy
containers running, so I want to make sure memory isn't too sparse. As of this
writing, here are the specs of these types:

| Instance Type | vCPU | Memory (GB) | Preemptible Price ($/mo) |
| ------------- |:----:|:-----------:| ------------------------:|
| n1-standard-1 | 1 | 3.75 | $7.30 |
| g1-small | 1 | 1.70 | $5.11 |
| f1-micro | 1 | 0.60 | $2.56 |

Here is the command I used to create my cluster.

```bash
gcloud compute clusters create cheap-kubernetes \
    --disk-size=20 \
    --machine-type="n1-standard-1" \
    --maintenance-window=09:00 \
    --num-nodes=1 \
    --preemptible \
    --enable-autoscaling \
    --max-nodes=5 \
    --min-nodes=1 \
    --zone=us-central1-a
```

I chose `disk-size` to be 20GB per node. I don't really need much space,
so I don't want to pay for it. `maintenance-window` is the time that GKE will
perform any maintenance on the nodes (time in UTC, which is about 3AM for me).
I'm choosing 1 node to start with, with autoscaling between 1 and 5 nodes.
I chose to use preemptible nodes because they're way cheaper, and for my
personal playground, I'm ok if I have infrequent interruptions.

Execute this command, and you'll see a handful of warnings before it begins
creating the cluster. Cluster creation takes about 5 minutes. When you're done,
you should see something like this:

```text
Creating cluster cheap-kubernetes in us-central1-a... Cluster is being health-checked (master is healthy)...done.
Created [https://container.googleapis.com/v1/projects/cheap-cluster/zones/us-central1-a/clusters/cheap-kubernetes].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-central1-a/cheap-kubernetes?project=cheap-cluster
kubeconfig entry generated for cheap-kubernetes.
NAME              LOCATION       MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
cheap-kubernetes  us-central1-a  1.12.8-gke.6    12.345.67.890  n1-standard-1  1.12.8-gke.6  1          RUNNING
```

## Accessing the Cluster

Once your cluster is up and running, you'll want to inspect it with kubectl.
Let's get the credentials...

```bash
gcloud container clusters get-credentials cheap-kubernetes
```

This command sets up your local kubectl credentials information. Once it's
complete run `kubectl config current-context` to confirm you're pointing
to the right cluster. Then check out `kubectl get nodes` to see the
single node running.

```text
>kubectl get nodes
NAME                                              STATUS    ROLES     AGE       VERSION
gke-cheap-kubernetes-default-pool-8c8ca16c-8t7x   Ready     <none>    8m        v1.12.8-gke.6
```

## First Deployment

Let's run something! I'll use an existing docker container for simplicity.
We'll make a new deployment by creating a new file called `first-deploy.yaml`.

`gist:65c22cfdb517b21538ca3368900bc836#first-deploy.yaml`

Deploy the app using `kubectl apply -f first-deploy.yaml`. After
a few seconds, try the following commands to inspect your deployment.

```text
>kubectl get deployments
NAME          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-world   1         1         1            1           1m

>kubectl get services
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
hello-world-service   ClusterIP   10.59.245.250   <none>        8080/TCP   2m
kubernetes            ClusterIP   10.59.240.1     <none>        443/TCP    21m

>kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
hello-world-5585c5ccd7-24fld   1/1       Running   0          2m
```

Next, make sure your service is responding by proxying requests
from your local machine. In one command line, run the following command:

```bash
kubectl port-forward services/hello-world-service 3000:8080
```

This will take requests made to localhost on port 3000 and send them
to the hello-world-service service on port 8080. In another command
prompt, test your connection with cURL `curl 127.0.0.1:3000`. You should
see something like:

```text
Hello, world!
Version: 1.0.0
Hostname: hello-world-5585c5ccd7-24fld
```

## Set up nginx

Google provides a load balancer out of the box, but I don't really
want to pay for that, so I'll just make my own HTTP proxy
using nginx. Make a new file called nginx-config.yaml and populate
it with the following:

`gist:22abf31fee82f2e0f9746c81816a0188#nginx-config.yaml`

Apply this configuration using `kubectl apply -f nginx-config.yaml`.
Once they are created, your node (and each subsequent one) will
be listening on port 80 for HTTP and passing to the hello-world-service. Run `kubectl get node -o yaml`
and look for ExternalIP in the output.

Before you can send an HTTP request to your node, you need
to add a firewall rule to allow http access to your node. Run
the following command to add that firewall rule.

```bash
gcloud compute firewall-rules create allowhttp \
    --action=allow \
    --rules=tcp:80,tcp:443
```

Note! This will open up all HTTP traffic to the resources
in your project. If you have sensitive resources, please restrict
this command to only apply to your kubernetes nodes.

Next, try to curl your node's IP address to see the hello message:
`curl 12.345.67.890/hello`. Other paths should return 404's.

## Next Steps

There are a few more last items to wrap up this project.
The easy one is to make a prefix on your domain point to the
cluster. Add an alias (A) record in your DNS provider pointing
your website's subdomain at the cluster using the IP you got in
the previous step. I updated my site's `api.` prefix to point
at the cluster.

After that, you'll want to set up a deployment on the cluster
to keep your DNS up to date. If you're using Cloudflare, I'd
recommend using [this repository](https://github.com/calebdoxsey/kubernetes-cloudflare-sync). The author goes into more detail
on [his blog](https://www.doxsey.net/blog/kubernetes--the-surprisingly-affordable-platform-for-personal-projects),
and this post is largely based on his blog post.

Lastly, you'll want to update your nginx proxy with TLS certificate
to enable secure HTTPS communication.

Thanks for checking out this post! Let me know what you think!
