---
title: "Deploying memcached in a StatefulSet with OpenShift"
date: 2018-05-03T22:48:00-04:00
draft: false
tags: ["kubernetes","openshift"]
categories: ["DevOps"]
---
Over the past few months at Red Hat, I've been working with my team on streamlining our CI/CD process and migrating some of our applications into [OpenShift](https://www.openshift.com/). As we've been slowly moving apps, it's been a great opportunity to revisit some of the basics of our architecture and look at ways we can better use OpenShift to . What may have worked well in a VM-based deployment doesn't necessarily translate well into a container-based deployment. For the sake of this post, I'll be showing how we use a recently stable feature of OpenShift (and Kubernetes) to deploy [memcached](https://memcached.org/) for one of our Ruby apps on the Red Hat Customer Portal.
<!--more-->

## The Problem

In our VM-based deployment, we co-located an instance of memcached on each VM that ran our app (that's the textbook use-case!). Our app was then configured to connect to each app server and shard keys across the memcached instances. This worked fine for us in an environment where hostnames didn't change all-too-often. If you've ever deployed an app in OpenShift or Kubernetes, you've probably noticed that whenever you rollout a new deployment you end up with a bunch of pods with funky hostnames like **app-8-m6t5v**. You probably never even cared, since resources like routes and services abstract us from needing to care about those hostnames. In an OpenShift deployment like this where instances of our app come and go fairly often, it's not feasible to configure our app to connect to memcached instances by pod name.

## Potential Solutions

### One Big Pod

Memcached was designed to take advantage of unused memory resources distributed across web servers. We don't quite have that same problem now that OpenShift will . Instead of using, say, (8) 256MB instances of memcached, we can use just use one 2GB instance, right?

![It's a trap!](https://patrickeasters.com/wp-content/uploads/2018/05/itsatrap-300x169.jpg)

Well, you could, but having only one replica of anything is usually a bad idea. In our environment, our nodes get rebuilt at-minimum once a week due to things like updates and autoscaling. If our pod is on a node getting rebuilt, there will at least be a minute or two where it will be unavailable while it's being rescheduled on a new node. Losing all of our cache each time that happens would be less than optimal. While most objects aren't cached very long and our app gracefully handles cache being unavailable, it's still not super great for performance or our backend services. Let's see if we can find a better way to deploy this while still keeping it distributed.

### Creating multiple services and deployments

One way to approach this would be to create a [deployment](https://docs.openshift.com/container-platform/3.9/dev_guide/deployments/how_deployments_work.html) and [service](https://docs.openshift.com/container-platform/3.9/architecture/core_concepts/pods_and_services.html#services) for each instance of memcached we want. A deployment ensures that we have a pod running and a service provides a static hostname we can use for accessing the pod. We would ultimately need to create a deployment and service for each instance of memcached (e.g. memcached-a, memcached-b, memcached-c, etc). This would work fine,  but it causes some management overhead since we have to configure each individual instance instead of configuring one resource to define them all.

### StatefulSets

Building on top of the previous approach, a newer OpenShift feature called [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) allows us to use a single controller to manage all of our memcached pods. The Kubernetes docs give a good overview of all the things you can do with them, but since memcached doesn't need any persistent storage or have any dependencies on which pods come up first, we'll mainly take advantage of the stable pod identities provided by StatefulSets. The StatefulSet resource will allow us to specify a desired number of replicas, and create those pods with stable names (e.g. memcached-0, memcached-1, memcached-3, etc). Whenever a pod is terminated, a new one will replace it with the same pod name. This means we can configure our app with a list of memcached shards and expect it work even when pods come and go.

## Let's Deploy It

Before we get started, I'm going to assume to you're logged into an OpenShift or Kubernetes cluster and have the `oc` or `kubectl` CLI tools installed. Though I'll probably mention OpenShift more often, all the concepts in this article will translate over to a vanilla Kubernetes cluster as well.

### Create the service

Since our apps shard keys across all memcached instances in the cluster, the typical load-balanced service isn't all that useful. Instead, we'll create a headless service where no load balancing or proxying takes place. This service simply allows endpoints to be looked up via DNS or via the API.

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  labels:
    component: memcached
  name: memcached
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    component: memcached
  ports:
    - name: memcached
      port: 11211
      protocol: TCP
      targetPort: 11211
EOF
```

### Create the StatefulSet

Now that we have the service in place, let's spin up some memcached instances. For this example, lets create 3 shards/replicas with 64MB of memory each. Since we don't care about what order pods spin up, we're also specifying the parallel `podManagementPolicy`. This still ensures our pods get their unique names, but doesn't limit us to spinning up one pod at a time. By specifying `serviceName` here, we also get a unique hostname for each pod based on the pod name and service name (e.g. `memcached-0.memcached.default.svc.cluster.local`, `memcached-1.memcached.default.svc.cluster.local`, etc)

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  labels:
    component: memcached
  name: memcached
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      component: memcached
  serviceName: memcached
  podManagementPolicy: Parallel
  template:
    metadata:
      labels:
        component: memcached
    spec:
      containers:
        - name: memcached
          args: ["memcached", "-m", "64"]
          image: memcached:1.5
          ports:
            - containerPort: 11211
              name: memcached
              protocol: TCP
          livenessProbe:
            tcpSocket:
              port: memcached
          readinessProbe:
            tcpSocket:
              port: memcached
          resources:
            limits:
              cpu: "1"
              memory: 96Mi
            requests:
              memory: 64Mi
EOF
```

At this point, we should have a ready-to-go memcached cluster. Let's check things out and make sure we see the pods and that they're discoverable. Feel free to skip ahead.

```bash
# Confirm we see 3 memcached pods running
$ kubectl get po -l component=memcached
NAME READY STATUS RESTARTS AGE
memcached-0 1/1 Running 0 1d
memcached-1 1/1 Running 0 1d
memcached-2 1/1 Running 0 1d

# Let's spin up a pod with a container running digso we can confirm DNS entries
$ kubectl run net-utils --restart=Never --image=patrickeasters/net-utils
pod "net-utils" created

# The service hostname should return with a list of pods
# Don't forget to replace default with the name of your OpenShift project
$ kubectl exec net-utils dig +short memcached.default.svc.cluster.local
10.244.0.17
10.244.0.15
10.244.0.16

# Pod IPs can still change, so it's best to configure any apps with hostnames instead
# Looking up SRV records for the service hostname should give us the pod FQDNs
$ kubectl exec net-utils dig +short srv memcached.default.svc.cluster.local
10 33 0 memcached-2.memcached.default.svc.cluster.local.
10 33 0 memcached-1.memcached.default.svc.cluster.local.
10 33 0 memcached-0.memcached.default.svc.cluster.local.

# Let's clean up this pod now that we're done with our validation
# (Or you can keep running queries... that's cool too)
$ oc delete po net-utils
pod "net-utils" deleted</pre>
```

## Connecting Our App

Now it's time to point our app to our memcached cluster. Instead of configuring a static list of pods, we'll take advantage of the built-in DNS service discovery. All we have to do is provide the hostname of our service, and our app can discover all the pods in the memcached cluster on its own. If you speak Ruby (or at least pretend to, like me), feel free to glean from the below example from one my team's apps.

```ruby
memcached_hosts = []
Resolv::DNS.new.each_resource('memcached.default.svc.cluster.local', Resolv::DNS::Resource::IN::SRV) { |rr|
  memcached_hosts << rr.target.to_s
}
config.cache_store = :dalli_store, memcached_hosts</pre>
```

## Closing Thoughts

Hopefully now you have a working memcached cluster and are on your way to configuring your app to take advantage of some of service discovery greatness we get for free from OpenShift and Kubernetes. Let me know in the comments or on Twitter if you were able to try this out for yourself. Happy caching!
