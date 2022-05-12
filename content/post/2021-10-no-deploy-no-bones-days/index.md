---
title: Making No Bones About Kubernetes Admission Control
date: 2021-10-31T00:00:00-04:00
draft: false
tags: ["kubernetes","golang"]
categories: ["DevOps","Kubernetes"]
---
Fewer things raise will raise as many pitchforks amongst the DevOps community as the mention of [No-Deploy Fridays](https://charity.wtf/2019/05/01/friday-deploy-freezes-are-exactly-like-murdering-puppies/)&mdash;and for good reason. One of the end goals of CI/CD is making deployments uneventful and boring. But in the current state of our world, I think we can all agree that production deployments are best saved for [bones days](https://knowyourmeme.com/memes/bones-day-no-bones-day).

I've been diving deeper into Kubernetes lately and wanted to see what it'd take to actually enforce this new mantra of "No Bones, No Deploys." Enter Kubernetes [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/). Let's see what it takes to build an admission webhook that only lets us create and update deployments on bones days.
<!--more-->

## What is an admission webhook anyway?

Admission webhooks are custom HTTP callback endpoints that are called when resources are created/modified/destroyed in Kubernetes. There are two types of webhooks that are called during the lifecycle of an API request:

* **Mutating admission webhooks** are called first and can modify resources. This is useful if you want to set some default values, add a sidecar container to pods, or otherwise change resources before they are applied.
* **Validating admission webhooks** are called after any modifications are complete for any final validations. This comes in handy if you have specific compliance requirements or want to do something silly like restrict changes during a full moon (or a "no bones day" in our case).

Webhooks give you a lot of power without needing to write a full-fledged controller or operator.

## Writing our webhook

For our needs, we are going to implement a validating admission webhook. Go is my programming language of choice these days, but a webhook could be implemented with anything that speaks HTTP.

Our webhook needs to do two things:
1. Accept a HTTP POST request with an `AdmissionReview` request in its body
2. Return an HTTP 200 response with a body containing an `AdmissionReview` JSON object. Most importantly it needs to include the UID from the request and a boolean indicating if the admission is allowed.

### Anatomy of a request
