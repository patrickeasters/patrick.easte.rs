---
title: "TLS for free in OpenShift"
date: 2018-05-03T22:48:00-04:00
draft: true
tags: ["openshift"]
categories: ["DevOps"]
---
There are fewer things more mundane in the world of software than TLS certificates and PKI in general. It's easy to get wrong and troubleshooting it can be frustrating at best. But in today's world where many of our apps are running in cloud networks we can't fully trust, using encryption-in-transit has become a necessity. As my team at Red Hat has been deploying our own applications inside of OpenShift, ensuring all our internal traffic is encrypted has been one of our architectural requirements. Even with just a handful of services, managing certificates for them across multiple environments can be a pain. What if I told you OpenShift can handle all of the certificate signing for you?
<!--more-->

## Meet Service Signing Certs

Hiding within the OpenShift docs is a feature called [Service Signing Certificates](https://docs.openshift.com/container-platform/3.10/dev_guide/secrets.html#service-serving-certificate-secrets). OpenShift already has a certificate authority it uses to sign certificates for masters and nodes, so this feature builds upon that and lets OpenShift sign certificates for your services as well. Simply add an annotation to your service, and a secret containing a certificate and private key will automatically be created. This cert can be used for traffic between pods to other internal services as well as for traffic between the OpenShift routers and your pods.

## Trying It Out

Let's try this thing out on some
