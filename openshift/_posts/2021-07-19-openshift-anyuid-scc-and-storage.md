---
layout: post
tags: OpenShift SCC anyuid Storage fsGroup SecurityContextConstraint
title:  "The OpenShift anyuid SCC and its effects on storage"
---

# The OpenShift anyuid SCC and its effects on storage

Working for IBM on the Cloud Pak for Integration, recently I've been spending a lot of time
with operators, and how to make running software on OpenShift as easy as possible. I came
across a situation where when running under the `restricted` SCC, our pods behave, but under
the less restrictive `anyuid` SCC, they can fail to access storage. This post is an exploration
of why that is, and a suggestion of how to address it.

## Using the restricted SCC

Best practice when creating an OpenShift pod is to use the `restricted` SCC. The restricted
SCC adds a number of security features to your running pods, including running as a random
user and group ID that cannot clash with other namespaces in your cluster.

To take advantage of those features, when creating pods you do not specify the user and group
ID so that OpenShift can assign one. This all works perfectly when the `restricted` SCC is in
use, because the assignment will be done on pod admission so the metadata is all there for
Kubernetes to use when mounting storage volumes.

When Kubernetes mounts storage volumes to pods, it will (caveat: not for all provisioners!)
`chmod` and `chown` the volume recursively to allow the pod to access the files. It does this
based (partly) on the `fsGroup` provided in the pod's `securityContext`. Under the restricted
SCC, if you don't specify the `fsGroup` yourself, one will be provided based on the allocated
range for the namespace.

## What's different with anyuid?

The `anyuid` SCC enables an important use case for OpenShift - running a pod under the user
and group defined in the image. In a plain Kubernetes distribution, you can do this by not
specifying your own values in the pod spec. In OpenShift, if you don't specify values in the
pod spec and the `restricted` SCC is applied, it will provide defaults, so the image definition
will not be used.

Generally, this is a good thing, and a more secure way to run the containers, but there are
situations where you need to run using the image definition. To achieve this, you can apply
the `anyuid` SCC to the pod, which has a higher priority, and doesn't provide defaults to the
pod when it is applied.

Unfortunately, if your pod is relying on defaults to be applied to successfully `chmod` and
`chown` your volumes on mount, then applying the `anyuid` SCC to a pod designed for the
`restricted` SCC can mean you can no longer access mounted storage.

## What can we do about it?

Ideally, we would know in advance which SCC would be applied, and could modify our resources
to specify metadata if we knew OpenShift wasn't going to do it for us, but unfortunately, there's
no easy way for the operator to know which SCC will be applied to a pod in advance, and it could
change over time anyway.

When admitting pods under the `restricted` SCC, if the `fsGroup` is not already set, the first
group in the range defined by the `openshift.io/sa.scc.supplemental-groups` annotation on the
namespace will be inserted into the pod definition. This is the part that doesn't happen with
the `anyuid` SCC.

To make our pod compatible with both SCCs, what we can do is examine the annotation ourselves,
and set the `fsGroup` for our pod to the first group in the range. This will satisfy the
`restricted` SCC, because the group is in range, and will provide the metadata for Kubernetes
to prepare the mount under the `anyuid` SCC.

## Can you back that up?

I ran some tests, and you can too, using my handy test repo:

[https://github.com/Jamstah/write-test-container](https://github.com/Jamstah/write-test-container)

I ran these tests on a Red Hat OpenShift on IBM Cloud cluster using `ibmc-block-gold` storage with
OpenShift 4.6.36. The important thing to note is that the volume is always writable where the fsGroup
has been defined within the range of groups for the namespace.

The `scc-nonroot-no-context` job is an example of a pod designed for the `restricted` SCC only.

The `scc-anyuid-no-context` job shows what happens when a pod designed for the `restricted` SCC is
run using the `anyuid` SCC.

The `scc-nonroot-fsgroupproject` and `scc-anyuid-fsgroupproject` jobs show how adding an `fsGroup`
within the correct range can solve the problem under both SCCs.

Here are my results:

| Job | Job spec fsGroup | Pod Admitted | Effective SCC | Pod spec fsGroup | UID | GID | Groups | Volume UID | Volume GID | Volume perms | Writable | Written UID | Written GID | Written perms |
| === | ================ | ============ | ============= | ================ | === | === | ====== | ========== | ========== | ============ | ======== | =========== | =========== | ============= |
| scc-anyuid-fsgroup0 | 0 | Yes | anyuid | 0 | uid=1001(1001) | gid=0(root) | groups=0(root) | root | root | drwxrwsr-x. | Yes | 1001 | root | -rw-r--r--. |
| scc-anyuid-fsgroupproject | 1000670000 | Yes | anyuid | 1000670000 | uid=1001(1001) | gid=0(root) | groups=0(root),1000670000 | root | 1000670000 | drwxrwsr-x. | Yes | 1001 | 1000670000 | -rw-r--r--. |
| scc-anyuid-no-context |  | Yes | anyuid |  | uid=1001(1001) | gid=0(root) | groups=0(root) | root | root | drwxr-xr-x. | No |
| scc-default-fsgroup0 | 0 | No |
| scc-default-fsgroupproject | 1000670000 | Yes | restricted | 1000670000 | uid=1000670000(1000670000) | gid=0(root) | groups=0(root),1000670000 | root | 1000670000 | drwxrwsr-x. | Yes | 1000670000 | 1000670000 | -rw-rw-r--. |
| scc-default-no-context |  | Yes | restricted | 1000670000 | uid=1000670000(1000670000) | gid=0(root) | groups=0(root),1000670000 | root | 1000670000 | drwxrwsr-x. | Yes | 1000670000 | 1000670000 | -rw-rw-r--. |
| scc-nonroot-fsgroup0 | 0 | Yes | nonroot | 0 | uid=1001(1001) | gid=0(root) | groups=0(root) | root | root | drwxrwsr-x. | Yes | 1001 | root | -rw-r--r--. |
| scc-nonroot-fsgroupproject | 1000670000 | Yes | restricted | 1000670000 | uid=1000670000(1000670000) | gid=0(root) | groups=0(root),1000670000 | root | 1000670000 | drwxrwsr-x. | Yes | 1000670000 | 1000670000 | -rw-r--r--. |
| scc-nonroot-no-context |  | Yes | restricted | 1000670000 | uid=1000670000(1000670000) | gid=0(root) | groups=0(root),1000670000 | root | 1000670000 | drwxrwsr-x. | Yes | 1000670000 | 1000670000 | -rw-r--r--. |