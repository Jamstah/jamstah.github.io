---
layout: post
tags: appconnect integration ace image volume
title: "IBM App Connect Enterprise containers using image volume mounts"
image:
  path: /assets/ace-image-volume.jpg
  alt: A shipping container stacked neatly inside a larger container, surrounded by other containers on a cargo dock.
---

You can use image volume mounts in an App Connect Runtime using the App Connect operator as part of Cloud Pak for Integration.
Most Cloud Pak for Integration users will have a local registry for storing their container images and will typically mirror software to ensure reliable image access.
Reusing this infrastructure for bar files makes deployment and reliability simple.

## Step 1: Put bar files into an image

Use a very simple `FROM scratch` Dockerfile to copy the bar files you want onto an image.

```Dockerfile
FROM scratch

COPY strings.bar .
```

If you don't want to install a full container client into your system, you can also use `crane` to build the image.

Build and push this image somewhere your cluster can pull it from:
```
podman build -t quay.io/jammy/strings-bar:latest .
podman push quay.io/jammy/strings-bar:latest
```

## Step 2: Configure your App Connect runtime to mount the image

Mount the image with your bar files into the containers where the App Connect runtime can pick them up.

In the standard image, you can use `/home/aceuser/initial-config/bars/`

```yaml
apiVersion: appconnect.ibm.com/v1beta1
kind: IntegrationRuntime
metadata:
  name: strings
  namespace: strings
spec:
  license:
    accept: false # Change this to true
    license: L-RJGW-BUAA2R
    use: CloudPakForIntegrationNonProductionFREE
  template:
    spec:
      containers:
        - name: runtime
          volumeMounts:
            - mountPath: /home/aceuser/ace-server/run
              name: strings
      volumes:
        - image:
            reference: 'quay.io/jammy/strings-bar:latest'
          name: strings
  version: '13.0'
```

## Test that your bar is loaded

Now in the logs of the App Connect runtime pod, you'll see the bar file being loaded:

```
2026-08-18 15:14:40.068456: Deploying bar files
2026-08-18 15:14:40.068567: List of bars to deploy: -a "/home/aceuser/initial-config/bars/strings.bar"
```

For most deployments you're also likely to want some more App Connect configuration like credentials.
You can add those to the same runtime in the same way as you would using the App Connect dashboard or other gitops approaches.
