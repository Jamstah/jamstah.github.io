---
layout: post
tags: appconnect integration ace openshift gitops argocd cp4i
title: "Resolving Out-of-Sync App Connect Resources in OpenShift GitOps"
excerpt_separator: <!--more-->
image:
  path: /assets/app-connect-gitops-sync.jpg
  alt: A GitOps pipeline synchronizing configuration and state between Git repositories and container runtimes.
---

When managing IBM App Connect in containers (part of IBM Integration Software, formerly Cloud Pak for Integration) with OpenShift GitOps (Argo CD), newly deployed `IntegrationRuntime` and `Configuration` resources can immediately appear `OutOfSync`.

This occurs because the App Connect operator automatically mutates live Custom Resources (CRs) after creation by injecting default configurations and migrating inline data to secrets.

<!--more-->

> **Note on Naming:** Cloud Pak for Integration has been renamed to **IBM Integration Software**. While operator license annotations and fields (such as `use: CloudPakForIntegrationNonProductionFREE`) may still carry the legacy identifiers for backward compatibility, the platform itself is now IBM Integration Software.

---

## Issue 1: IntegrationRuntime Auto-Injected Configurations

By default, an `IntegrationRuntime` enables both the admin REST interface and the MCP (Model Context Protocol) server. If you do not specify configuration objects for these features, the operator automatically provisions them and appends two entries to `spec.configurations` on the live cluster:

- `<runtime-name>-mcp-ba-creds`
- `<runtime-name>-ir-adminssl`

### Example

You declare this resource in your Git repository:

```yaml
apiVersion: appconnect.ibm.com/v1beta1
kind: IntegrationRuntime
metadata:
  name: greetingapi1
spec:
  license:
    accept: true
    license: L-RJGW-BUAA2R
    use: CloudPakForIntegrationNonProductionFREE
  barURL:
    - https://gitea.example.com/barfiles/raw/branch/main/greetingapiv1.bar
  configurations:
    - barauth-empty
  logFormat: basic
  version: "13.0.8"
```

The operator mutates the live resource on the cluster:

```yaml
spec:
  configurations:
    - barauth-empty
    - greetingapi1-mcp-ba-creds
    - greetingapi1-ir-adminssl
```

Because Git only contains `barauth-empty`, Argo CD reports the resource as `OutOfSync`.

### Solutions

1. **Disable MCP if unused (Recommended for MCP):**
   If you are not using the MCP server, disable it explicitly in Git under `spec.mcp.runtime.disabled`:

   ```yaml
   spec:
     mcp:
       runtime:
         disabled: true
   ```
   This prevents the operator from injecting `<name>-mcp-ba-creds`.

2. **Handle the Admin SSL Configuration:**
   For `*-ir-adminssl`, you can either pre-create and declare your own admin SSL `Configuration` object in Git (recommended), or configure Argo CD to ignore the generated configuration in its diff calculation.

   To ignore the generated SSL configuration across all `IntegrationRuntime` resources, add a `resourceIgnoreDifferences` entry to your `ArgoCD` custom resource:

   ```yaml
   spec:
     resourceIgnoreDifferences:
       resourceIdentifiers:
         - group: appconnect.ibm.com
           kind: IntegrationRuntime
           customization:
             jqPathExpressions:
               - '.spec.configurations[] | select(endswith("-ir-adminssl"))'
   ```

   You can use similar configuration in a GitOps `Application` if you don't control the whole GitOps installation. Use the same block, but under `spec.ignoreDifferences`.

---

## Issue 2: Configuration Resources Mutating `spec.data`

When you create a `Configuration` resource that defines inline base64 data using `spec.data`, the operator extracts the payload into an automatically generated `Secret`, removes `spec.data` from the CR on the cluster, and sets `spec.secretName` and `spec.version`.

### Example

You declare this in Git:

```yaml
apiVersion: appconnect.ibm.com/v1beta1
kind: Configuration
metadata:
  name: barauth-empty
spec:
  type: barauth
  description: Empty BarAuth file for git access.
  data: eyJhdXRoVHlwZSI6IkJBU0lDX0FVVEgiLCJjcmVkZW50aWFscyI6eyJ1c2VybmFtZSI6IiIsInBhc3N3b3JkIjoiIn19Cg==
```

On the cluster, the operator updates it to:

```yaml
spec:
  type: barauth
  description: Empty BarAuth file for git access.
  secretName: barauth-empty-bpq5l
  version: 13.0.8.2-r1
```

Argo CD flags the missing `spec.data` as a live diff.

### Solutions

1. **Create the Secret directly (Recommended):**
   Instead of embedding `spec.data` directly in the `Configuration` CR, manage the Kubernetes `Secret` explicitly with GitOps (leveraging Sealed Secrets, External Secrets, or SOPS for sensitive values) and reference it with `spec.secretName`:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: barauth-empty-secret
   type: Opaque
   data:
     configuration: eyJhdXRoVHlwZSI6IkJBU0lDX0FVVEgiLCJjcmVkZW50aWFscyI6eyJ1c2VybmFtZSI6IiIsInBhc3N3b3JkIjoiIn19Cg==
   ---
   apiVersion: appconnect.ibm.com/v1beta1
   kind: Configuration
   metadata:
     name: barauth-empty
   spec:
     type: barauth
     description: Empty BarAuth file for git access.
     secretName: barauth-empty-secret
   ```

2. **Ignore `spec.data` differences in Argo CD:**
   If you want to keep inline `spec.data` in Git, configure the `ArgoCD` CR to ignore `.spec.data` differences on `Configuration` resources:

   ```yaml
   spec:
     resourceIgnoreDifferences:
       resourceIdentifiers:
         - group: appconnect.ibm.com
           kind: Configuration
           customization:
             jqPathExpressions:
               - .spec.data
   ```
