---
layout: post
tags: kubernetes OpenShift MQ Keycloak http2 redirect loop cp4i integration
title: "Fixing an IBM MQ web console redirect loop on OpenShift"
excerpt_separator: <!--more-->
---

I've seen a number of Cloud Pak for Integration installations where the MQ web console is inaccessible because of a redirect loop attempting to log in.
Typically, this happens because HTTP/2 is pretty aggressive when it comes to reusing connections for performance reasons.
It can only happen when the MQ web console presents a certificate that is also valid for the Keycloak instance, which is more common than you might think.

<!--more-->

There are two fixes for this.

## Fix 1: Use more precise certificates

This problem occurs when the MQ web console presents a certificate that is also valid for the Keycloak instance, which typically means it's a wildcard certificate valid for `*.apps.<cluster>`.
Because it's valid for both hostnames, and both hostnames resolve to the same load balancer, HTTP/2 will attempt to reuse the connection, sending requests meant for Keycloak to MQ.

To stop the browser attempting to coalesce the connections, you can use a specific certificate for just the MQ web console domain instead of reusing the wildcard certificate for the cluster apps domain.
The browser will then correctly send requests for Keycloak on a new connection to the OpenShift router.

## Fix 2: Turn off HTTP/2 on the MQ web console

You can also turn off HTTP/2 (use HTTP/1.1) on the MQ web console, which means the browser will not attempt to coalesce the connections.
To do this, configure the MQ web console liberty server using the override mechanism in the MQ operator.

1.  Create a `ConfigMap` containing the configuration:
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: mq-web-config-without-http2
    data:
      mqwebuser.xml: |
        <?xml version="1.0" encoding="UTF-8"?>
        <server>
          <httpEndpoint id="defaultHttpEndpoint" protocolVersion="http/1.1"> </httpEndpoint>
        </server>
    ```

2.  Reference the `ConfigMap` in your `QueueManager` resources:
    ```yaml
    ...
    spec:
      web:
        manualConfig:
          configMap:
            name: mywebconfig
    ...
    ```

## Why is it happening?

The root cause of this issue is that HTTP/2 is more aggressive in terms of connection reuse and will attempt to coalesce any hostname/IP/certificate combination that has overlap into a single connection.
Daniel Stenberg of cURL fame wrote about [HTTP/2 Connection Coalescing](https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/) in his blog.

Typically, coalescing works on an OpenShift cluster as long as the TLS is terminated at the OpenShift router.
The browser will connect to the OpenShift router once and will reuse that connection for any other hostnames pointing at the OpenShift router that are covered by the same wildcard certificate.
The OpenShift router will correctly identify the hostname associated with each individual request over that coalesced connection and forward them on to the right backend services.
If a request for a hostname set to use Passthrough comes through to the router on that same connection, it will return a [421 Misdirected Request](https://httpwg.org/specs/rfc7540.html#MisdirectedRequest) response, allowing the browser to try again with a new connection that can be properly terminated at the MQ web console pod instead.

In this case, the MQ web console is using Passthrough mode for handling TLS connections, so TLS is terminated at the MQ web console.
The OpenShift router then looks at the SNI header and tunnels the TCP connection from the browser directly to the MQ web console (i.e. Passthrough), and cannot see the individual requests.
When the browser then sees that the user is being redirected to Keycloak, it determines that it can coalesce that connection because the hostname/IP/certificate combination has overlap, and it attempts to reuse that connection.
The MQ web console doesn't check the hostname being used (it doesn't know the load balancer hostname, so it can't validate it anyway), so it assumes the request is for a page in the console.
Because the user isn't logged in, the MQ web console then redirects the user back to the login page over and over again because the coalesced connection means the browser never reaches Keycloak.
