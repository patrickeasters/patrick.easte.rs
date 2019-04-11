---
title: "Using Traefik with TLS on Kubernetes"
date: 2016-08-16T00:00:00-04:00
tags: ["kubernetes"]
categories: ["DevOps"]
---
Over the past few months, I’ve been working with [Kubernetes](http://kubernetes.io/) a lot as Ayetier has been making the shift towards container orchestration. As easy as it was to create and scale services, it was a bit frustrating to see how most reverse proxy solutions seemed kludgy at best.

That’s why I was pretty intrigued when I first read about [Traefik](https://traefik.io/) — a modern reverse proxy supporting dynamic configuration from several orchestration and service discovery backends, including Kubernetes.
<!--more-->

Traefik is still relatively new and doesn’t fully support Kubernetes’ TLS configuration for the ingress, so it took a bit of manual configuration. Much trial-and-error was involved, so I thought I’d share the process here.

## First Things First

I’m going to assume you already have a working Kubernetes cluster and have the `kubectl` tool installed to manage your cluster. I’m also on Google Container Engine (GKE), so depending on your cloud provider, a few things like LoadBalancer service types may be different.

Any code I use is also in [this GitHub repo](https://github.com/patrickeasters/traefik-k8s-tls-example), so feel free to clone it and follow along. (It’s more fun that way)

```bash
git clone https://github.com/patrickeasters/traefik-k8s-tls-example.git
```

## Deploy Backend Services
For the purposes of this post, I made a pretty simple web service in Go that will aid in testing. You could also [make your own service to display cat facts](https://catfact.ninja/) if you really want. Let’s go ahead and deploy 3 replication controllers and services from the [backend.yaml](https://raw.githubusercontent.com/patrickeasters/traefik-k8s-tls-example/master/backend.yaml) file.

```bash
kubectl create -f backend.yaml
```

## Secure All The Things
We’re just going to generate a self-signed certificate for this tutorial, but any certificate/key pair will work. Run the following command to generate your certificate and dump the certificate and private key.

```bash
openssl req
        -newkey rsa:2048 -nodes -keyout tls.key
        -x509 -days 365 -out tls.crt
```

Now that we have the certificate, we’ll use `kubectl` to store it as a [secret](http://kubernetes.io/docs/user-guide/secrets/). We’ll use this so our pods running Traefik can access it.

```bash
kubectl create secret generic traefik-cert
        --from-file=tls.crt
        --from-file=tls.key
```

## Configure Traefik
As I mentioned earlier, due to lack of native support for TLS with Kubernetes ingresses, we’ll have to do a bit of manual configuration on the pods running Traefik.The only real points of interest here are setting up the HTTP to HTTPS redirect and then setting the certificate to be used for TLS.

```toml
# traefik.toml
defaultEntryPoints = ["http","https"]
[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
      entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
      [[entryPoints.https.tls.certificates]]
      CertFile = "/ssl/tls.crt"
      KeyFile = "/ssl/tls.key"
```

Now let’s take this configuration and store it in a ConfigMap to be mounted as a volume in the Traefik pods.

```bash
kubectl create configmap traefik-conf --from-file=traefik.toml
```

## Deploy Traefik
Now we finally get to deploy the replication controller for Traefik. I’m going to use 2 pods, but this can be scaled out as desired. I’m also using a LoadBalancer service, but you can change it to NodePort and configure your external load balancer as required if your cloud provider doesn’t natively support it.

Here are a few things to note in the pod spec from [traefik.yaml](https://raw.githubusercontent.com/patrickeasters/traefik-k8s-tls-example/master/traefik.yaml), which contains the RC and service.

* The **traefik-cert** secret is mounted as a volume to **/ssl**, which allows the `tls.crt` and `tls.key` files to be read by the pod
* The **traefik-conf** ConfigMap is mounted as a volume to **/config**, which lets Traefik read the `traefik.conf` file
* The log level is set to debug, which is great when you’re troubleshooting or getting started, but it may be more manageable if you set it to something less chatty before going into production with it.

```yaml
spec:
  terminationGracePeriodSeconds: 60
  volumes:
  - name: ssl
    secret:
      secretName: traefik-cert
  - name: config
    configMap:
      name: traefik-conf
  containers:
  - image: traefik
    name: traefik-ingress-lb
    imagePullPolicy: Always
    volumeMounts:
    - mountPath: "/ssl"
      name: "ssl"
    - mountPath: "/config"
      name: "config"
    ports:
    - containerPort: 80
    - containerPort: 443
    args:
    - --configfile=/config/traefik.toml
    - --kubernetes
    - --logLevel=DEBUG
```

Now let’s go ahead and create this RC and service in the cluster.

```bash
kubectl create -f traefik.yaml
```

## Configuring the Ingress
Now that Traefik is up and running, we need to configure an ingress so it has actual rules. We’re going to set up 2 route prefixes, **/s1** and **/s2**, pointing to **svc1** and **svc2** respectively. Anything else will be sent to **svc3**. The pods running Traefik are watching the API for any changes made to the ingress configuration

**A note for any GKE users:** To prevent the default L7 load balancer ingress controller from picking up this configuration, I set the **kubernetes.io/ingress.class** annotation to **traefik**. Google’s ingress controller will ignore any ingresses whose class is not set to **gcp**.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-web-app
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  tls:
    - secretName: traefik-cert
  rules:
  - host:
    http:
      paths:
      - path: /s1
        backend:
          serviceName: svc1
          servicePort: 8080
      - path: /s2
        backend:
          serviceName: svc2
          servicePort: 8080
      - path: /
        backend:
          serviceName: svc3
          servicePort: 8080
```

Behold, our last configuration.

```bash
kubectl create -f ingress.yaml
```

## Testing It Out

At this point, you can test your routes and see your shiny new config doing its magic. In my output below, you can see each service identify itself.

**$ curl -k https<span></span>://myhost/s1**
```
Hi, I'm the svc1 service!
Hostname: svc1-on0mm
```

**$ curl -k https<span></span>://myhost/s2**
```
Hi, I'm the svc2 service!
Hostname: svc2-i485q
```

**$ curl -k https<span></span>://myhost/**
```
Hi, I'm the svc3 service!
Hostname: svc3-iict9
```

**$ curl -k https<span></span>://myhost/lolcat**
```
Hi, I'm the svc3 service!
Hostname: svc3-iict9
```

## Final Thoughts
Hopefully at this point you have a working Traefik reverse proxy setup. Hit me up in the comments or on Twitter if you have any questions.
