---
layout: post
tags: go fips redhat containers kubernetes
title: "Automatic FIPS 140-3 in Go: What Actually Works"
---

Ideally, you want to build software once and run it anywhere. With FIPS and Go that
can be tricky because in upstream, FIPS is baked in at build time. Using the Red Hat
golang toolchain you can produce binaries that detect the FIPS mode of the underlying
host, and automatically run in the right mode.

![A Go gopher wearing a FIPS compliance badge, standing in front of a server rack with a green padlock icon glowing on the front panel. Clean flat illustration, tech blog style, dark background.](/assets/go-automatic-fips.jpg)

Go 1.26 introduced `crypto/fips140` — a new standard library package giving Go
programs first-class awareness of FIPS 140-3 mode. Red Hat has been shipping a
patched Go toolchain in their UBI images for longer than that, with OpenSSL-backed
crypto that integrates with the Linux kernel's FIPS flag.

But how do these two mechanisms interact? And which build combinations actually
give you automatic, kernel-aware FIPS — meaning FIPS active on a FIPS-enabled
host and transparent everywhere else, from the same binary?

We ran a systematic test across all meaningful build combinations on both a
FIPS-enabled RHEL 10 host and a standard non-FIPS Linux host. This post shares
what we found. The test program and Makefile are available at
[github.com/Jamstah/fips-go](https://github.com/Jamstah/fips-go).

## TL;DR

To build a Go binary that automatically uses FIPS on FIPS-enabled hosts and runs
normally everywhere else, pick one:

**Option A — OpenSSL backend (simplest, no runtime config):**
```dockerfile
FROM registry.access.redhat.com/ubi10/go-toolset:1.26 AS builder
RUN go build -o /myapp .          # no GOFIPS140, no no_openssl

FROM registry.access.redhat.com/ubi10/ubi-minimal
COPY --from=builder /myapp /usr/local/bin/myapp
ENTRYPOINT ["/usr/local/bin/myapp"]
```

**Option B — Go native FIPS module (`fips140.Enabled()` works correctly):**
```dockerfile
FROM registry.access.redhat.com/ubi10/go-toolset:1.26 AS builder
RUN go build -tags no_openssl -o /myapp .

FROM registry.access.redhat.com/ubi10/ubi-minimal
COPY --from=builder /myapp /usr/local/bin/myapp
ENV GODEBUG=fips140=auto
ENTRYPOINT ["/usr/local/bin/myapp"]
```
FIPS activates automatically when `/proc/sys/crypto/fips_enabled=1`, and
`crypto/fips140.Enabled()` correctly reflects the state.

---

## Background: Two independent FIPS mechanisms

Before looking at results, it helps to understand that there are two completely
separate ways Go can enforce FIPS on RHEL/UBI. They do not communicate with
each other.

### 1. The Go native FIPS module (`GOFIPS140`)

`GOFIPS140` is a build-time environment variable consumed by `cmd/go`. When set
to anything other than `off`, it compiles the FIPS 140-3 cryptographic module
into the binary and bakes a `GODEBUG=fips140=on` default into the binary at
link time. This default is not a runtime environment variable — it is embedded
in the binary and takes effect on every host, unconditionally.

The `crypto/fips140.Enabled()` function introduced in Go 1.26 reflects this
mechanism.

### 2. The Red Hat OpenSSL backend

The Red Hat `ubi10/go-toolset` image replaces Go's standard crypto with an
OpenSSL-backed implementation by default (`GOEXPERIMENT=opensslcrypto`). OpenSSL
reads `/proc/sys/crypto/fips_enabled` at process startup and enforces FIPS when
the kernel flag is set — completely independently of `GOFIPS140`.

**`crypto/fips140.Enabled()` is blind to this mechanism.** A binary built with
`GOFIPS140=off` on a FIPS-enabled RHEL host will report `fips140.Enabled() = false`
while OpenSSL is actively enforcing FIPS.

### Why you can't combine them

Combining `GOFIPS140=latest` (or any non-off value) with the OpenSSL backend
panics at startup:

```
panic: opensslcrypto: GOLANG_FIPS and GODEBUG=fips140 are mutually exclusive
```

The Red Hat toolchain deliberately prevents both paths from being active
simultaneously. The `no_openssl` build tag disables the OpenSSL backend and
switches to the pure-Go FIPS module.

---

## The test

We built a small Go program that:

1. Prints build metadata from the binary's embedded build info (Go version,
   `GOFIPS140`, build tags) so each run is self-describing
2. Reports `crypto/fips140.Enabled()`
3. Attempts an HTTPS connection to `3des.badssl.com`, a host that only
   negotiates the non-FIPS 3DES cipher suite. The client is configured to
   offer only that suite (`TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA`, TLS 1.2 max,
   `InsecureSkipVerify` since cert validation is not what we're testing). If
   anything has removed 3DES from the stack, the handshake fails — no runtime
   flags needed.

We tested eleven combinations: eight distinct image builds, plus three of those
re-run with `GODEBUG=fips140=auto` injected at runtime.

---

## Results

### The full matrix

| Target | `GODEBUG` at runtime | `fips140.Enabled()` | 3DES on FIPS host | 3DES on non-FIPS host | FIPS mechanism |
|---|---|---|---|---|---|
| `golang:1.26`, `GOFIPS140=off` | — | false | succeeded | succeeded | none |
| `golang:1.26`, `GOFIPS140` unset | — | false | succeeded | succeeded | none (unset == off on upstream) |
| `golang:1.26`, `GOFIPS140=latest` | — | **true** | **blocked** | **blocked** | Go native, unconditional |
| `ubi10/go-toolset`, `GOFIPS140=off` | — | false | **blocked** | succeeded | OpenSSL reads kernel flag |
| `ubi10/go-toolset`, `GOFIPS140` unset | — | false | **blocked** | succeeded | same as above |
| `ubi10/go-toolset`, `GOFIPS140=off`, `no_openssl` | — | false | succeeded | succeeded | none |
| `ubi10/go-toolset`, `GOFIPS140` unset, `no_openssl` | — | false | succeeded | succeeded | none |
| `ubi10/go-toolset`, `GOFIPS140=latest`, `no_openssl` | — | **true** | **blocked** | **blocked** | Go native, unconditional |
| `ubi10/go-toolset`, `GOFIPS140=off`, `no_openssl` | `fips140=auto` | false/**true** | **blocked** | succeeded | Go native, kernel-aware |
| `ubi10/go-toolset`, `GOFIPS140` unset, `no_openssl` | `fips140=auto` | false/**true** | **blocked** | succeeded | Go native, kernel-aware |
| `ubi10/go-toolset`, `GOFIPS140=latest`, `no_openssl` | `fips140=auto` | false/**true** | **blocked** | succeeded | `auto` overrides link-time `on` |

### Key observations

**`crypto/fips140.Enabled()` only reflects the Go native FIPS module.** When
the OpenSSL backend is enforcing FIPS, `fips140.Enabled()` returns false. You
cannot use this API to check whether FIPS is active in an OpenSSL-backed binary.

**`GOFIPS140=latest` is unconditional.** The FIPS module is active on every
host, FIPS-enabled or not. There is no "activate only if the kernel has FIPS
enabled" behaviour in the upstream toolchain.

**`GODEBUG=fips140=auto` overrides the link-time default in both directions.**
On a `GOFIPS140=latest` binary running on a non-FIPS host, setting
`GODEBUG=fips140=auto` at runtime disables FIPS — 3DES succeeded and
`fips140.Enabled()` returned false.

---

## The two approaches that give you automatic FIPS

If you want a binary that is FIPS-compliant on a FIPS host and runs normally
elsewhere — without changing the binary between environments — there are two
approaches that work today.

### Approach 1: Build with the Red Hat toolset, keep the OpenSSL backend

This is the simplest approach and the default when using `ubi10/go-toolset`.

```dockerfile
FROM registry.access.redhat.com/ubi10/go-toolset:1.26 AS builder
# Do NOT set GOFIPS140, do NOT add no_openssl
RUN go build -o /myapp .

FROM registry.access.redhat.com/ubi10/ubi-minimal
COPY --from=builder /myapp /usr/local/bin/myapp
ENTRYPOINT ["/usr/local/bin/myapp"]
```

At runtime, OpenSSL reads `/proc/sys/crypto/fips_enabled` and automatically
enforces FIPS when the kernel flag is set. On a non-FIPS host, it runs normally.
No build flags, no environment variables, no operator action required.

**The catch:** `crypto/fips140.Enabled()` returns false even when FIPS is
active, because enforcement is happening in OpenSSL, not the Go native module.
If your application code calls `fips140.Enabled()` to gate behaviour, it will
not work correctly with this approach.

Also note: for this to work at runtime, the container must link against OpenSSL
— meaning you need a runtime image that includes OpenSSL libraries (such as
`ubi-minimal` or `ubi`), not a scratch image.

### Approach 2: Build with the Red Hat toolset, `no_openssl`, and set `GODEBUG=fips140=auto` at runtime

This approach uses the Go native FIPS module and activates it via the
`GODEBUG=fips140=auto` setting introduced by the Red Hat patches to the Go
toolchain.

```dockerfile
FROM registry.access.redhat.com/ubi10/go-toolset:1.26 AS builder
# Disable OpenSSL backend to use the Go native FIPS module
RUN go build -tags no_openssl -o /myapp .

FROM registry.access.redhat.com/ubi10/ubi-minimal
COPY --from=builder /myapp /usr/local/bin/myapp
ENTRYPOINT ["/usr/local/bin/myapp"]
```

Then set `GODEBUG=fips140=auto` in your runtime environment:

```yaml
# Kubernetes deployment
env:
  - name: GODEBUG
    value: fips140=auto
```

```bash
# Or directly
GODEBUG=fips140=auto ./myapp
```

At startup, the Go runtime reads `/proc/sys/crypto/fips_enabled`. If it is `1`,
FIPS mode activates and `fips140.Enabled()` returns true. On a non-FIPS host,
`auto` resolves to off and the program runs normally.

**The advantage over Approach 1:** `crypto/fips140.Enabled()` correctly reflects
the FIPS state, so your application code can query it reliably.

**The caveat:** `GODEBUG=fips140=auto` overrides the link-time default in both
directions. If you set it in a base image or cluster-wide environment and also
deploy binaries built with `GOFIPS140=latest`, those binaries will have FIPS
disabled on non-FIPS hosts. Only use `auto` if you want kernel-aware activation.
If you need FIPS unconditionally, use `GOFIPS140=latest` and do not set a
runtime `GODEBUG` override.

---

## A note on `GOFIPS140` unset with the Red Hat toolchain

The [golang-fips patch](https://github.com/golang-fips/go/blob/go1.26.5/patches/004-host-fips-auto.patch)
is designed to make `GOFIPS140` unset default to `v1.0.0` (a certified snapshot)
with `GODEBUG=fips140=auto` baked in at link time — giving automatic
kernel-aware activation with no runtime configuration needed. When this ships,
Approach 2 above would not require any runtime env var at all.

However, `ubi10/go-toolset:1.26` does not yet implement this default:
`go env GOFIPS140` returns `off` when unset. Until the patch ships in the image,
`GODEBUG=fips140=auto` must be set at runtime as shown above.

---

## What about the upstream `golang:1.26` image?

The upstream `golang:1.26` image does not include the Red Hat OpenSSL patches.
`GOFIPS140` unset is treated as `off`. The only way to get FIPS is
`GOFIPS140=latest`, which activates it unconditionally on all hosts.

If you need a kernel-aware, automatic FIPS behaviour with the upstream image,
you would need to detect `/proc/sys/crypto/fips_enabled` yourself at startup and
set `GODEBUG=fips140=on` programmatically — but that is not standard Go
behaviour and outside the scope of this post.

For FIPS requirements, the Red Hat UBI toolset images are the right choice.

---

## Summary

| Goal | Approach |
|---|---|
| FIPS always active, all hosts | `GOFIPS140=latest`, `no_openssl`, no runtime override |
| FIPS only on FIPS hosts (kernel-aware), `fips140.Enabled()` accurate | `no_openssl`, `GODEBUG=fips140=auto` at runtime |
| FIPS only on FIPS hosts (kernel-aware), simpler setup | Default UBI toolset build (OpenSSL backend, no `no_openssl`) |
| No FIPS | `GOFIPS140=off` or upstream image |

For production workloads that need to run on both FIPS and non-FIPS hosts from a
single image, either the OpenSSL default or the `GODEBUG=fips140=auto` approach
will work today. The OpenSSL approach requires no runtime configuration but does
not expose FIPS state through `crypto/fips140.Enabled()`. The `auto` approach
gives you accurate `fips140.Enabled()` reporting but requires setting a runtime
environment variable until the `v1.0.0` default patch ships in the toolchain.

The test code, Dockerfiles, and Makefile for all eleven combinations tested here
are available at [github.com/Jamstah/fips-go](https://github.com/Jamstah/fips-go).
