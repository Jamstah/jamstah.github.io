---
layout: post
tags: kubernetes apiversion api deprecation
title: "Resources in Kubernetes have no API version"
---

OK, so its a bit of a click-bait title, and its not 100% accurate either, but its how
I have come to think about resources in Kubernetes. Once a resource has been pushed to
the API server and accepted, it no longer has an API version. Exactly how Kubernetes
stores its metadata is not my concern.

However, I often find people talking about checking the API version of
a resource by fetching it from the Kubernetes API server and taking a look. Because of
the nature of Kubernetes, people will often do exactly that, and believe the result
that comes back matches exactly what was pushed to the API server, but that's not true
and can lead to missed opportunities for discovering and validating bugs.

In this post I'm going to try to explain how API versions are used within Kubernetes, and
what can go wrong. At the end I'll include some examples and background if you want to dig
deeper.

## How API versions are used within the API server

When making calls to the Kubernetes API server, you need to include the API version that
your client understands. By enabling clients to specify their API version, the Kubernetes
API is able to change without breaking old clients by presenting the same resources in
multiple forms.

The API version is the schema that the Kubernetes API server uses to serialize and de-serialize
resources that you pass to it and that it passes back as responses.

There is no use case within Kubernetes where the a user should be directly accessing
resources within the etcd database, so from an end user's perspective, the stored object
has no API version.

This intentionally simple diagram skips a lot of the detail of how the API server works,
and doesn't even include etcd, but illustrates how all API versions for a resource type
are just different ways in to the same set of resources.

![The client connects to either of the API versions within the Ingress API, which itself is part of the API server.](/assets/kube-api-server-apiversions.svg){: .center }

## What could go wrong?

For a concrete example, imagine you have written an operator for Kubernetes that deploys
software for your users, and you want to know if that's going to work in Kubernetes 1.22.
Kubernetes 1.22 removes a number of API versions that have been deprecated for a long time,
so you need to check that you aren't using them.

The immediate reaction could be to pull the resources that your operator creates from the
Kubernetes cluster and look at them, so you run `kubectl get ingress -o yaml` to check that
you're using the right version of the ingress API. The result comes back with an apiVersion
of `networking.k8s.io/v1`, so you think great, I'm covered.

However, what you've checked here is not actually what version of the API your operator is
using, but what version of the API kubectl is using. Your application could have created its
Ingress resource using the `v1beta1` API, and the cluster would still present you with the
`v1` representation when using kubectl.

This test also doesn't take into account the API version your operator uses to get resources
from the cluster.

How do you do it right? Follow the advice from the [Kubernetes deprecation guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/#what-to-do).

## Example: Multiple versions of the Ingress API

Lets use the example of a very simple Ingress object, straight out of the
Kubernetes documentation. The ingress API had schema changes between `v1beta1`
and `v1`, so we can use it to demonstrate the runtime conversion performed by
the API server when serializing the resource.

We'll use this example straight from the Kubernetes documentation:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

First step, push that resource to the cluster:

```
[jammy@ibm007470 ~]$ kubectl create -f minimal-ingress.yaml
ingress.networking.k8s.io/minimal-ingress created
```

Notice that there is no API version in that output.

So, next up, get the resource back out:

```
[jammy@ibm007470 ~]$ kubectl get ingress minimal-ingress -o yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  creationTimestamp: "2021-09-15T13:44:02Z"
  generation: 1
  managedFields: ...
  name: minimal-ingress
  namespace: jammy
  resourceVersion: "14187394"
  uid: 29aec213-b79a-4b96-9b05-072e9bb48541
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: test
            port:
              number: 80
        path: /testpath
        pathType: Prefix
status:
  loadBalancer: {}
```

So the default api version used by kubectl is `v1`, and the spec is exactly what we put in.

How about if we ask kubectl to use a specific version?

```
[jammy@ibm007470 ~]$ kubectl get ingress.v1beta1.networking.k8s.io -o yaml minimal-ingress
Warning: networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
```
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  creationTimestamp: "2021-09-15T13:44:02Z"
  generation: 1
  managedFields: ...
  name: minimal-ingress
  namespace: jammy
  resourceVersion: "14187394"
  uid: 29aec213-b79a-4b96-9b05-072e9bb48541
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: test
          servicePort: 80
        path: /testpath
        pathType: Prefix
status:
  loadBalancer: {}
```

Now the spec has been serialized to the `v1beta1` schema by the API server
at runtime, you can see `serviceName` instead of `service.name` in the yaml. Clients
who are using the `v1beta1` schema can continue to successfully use it to interact
with the cluster, at least until the API version is removed in Kubernetes 1.22.

## Behind the scenes: How does Kubernetes actually store its metadata?

Kubernetes uses etcd to store cluster metadata, so its all key/value based. Reaching
in to an etcd associated with a Kubernetes cluster you can find keys like this:

```
/registry/deployments/kube-system/coredns
```

That's the key for the `coredns` Deployment in the `kube-system` namespace. The key
itself does not have an API version, because the API version is only used for serializing
resources.

If you extracted the value associated with that key, you would find a protobuf serialized
representation in whatever API version the server is using for storage. When presenting
that to a user, the API server will convert it to whatever API version is requested by the
client.