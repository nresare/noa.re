---
title: Configuring Envoy Gateway to use post-quantum key exchange algorithms 
author: Noa Resare
date: 2026-07-26
---

## Intro

I have a small Kubernetes cluster at home, serving a few small websites. The HTTP traffic
is served by Envoy Proxy managed by an [Envoy Gateway](https://gateway.envoyproxy.io).
I have configured the gateway controller to merge all Gateway resources to a single workload so
that it can be connected to via a single set of NodePorts. These NodePorts are then being exposed 
by a simple network-layer proxy on the Linux box that is connected directly to my 
internet service provider.

I saw someone mention on fediverse that recent versions of major web browsers now support
post-quantum key exchange mechanisms, and that support on the server side is growing. This 
made me curious to figure out whether it was something I could introduce in my small environment
as well. In the end, it turned out to be a rather small change but the documentation was a bit 
lacking, so I thought I would share my findings here.

## Needed configuration

Envoy Gateway defines a custom resource kind `ClientTrafficPolicy` which references a Gateway resource
and configures how Envoy Proxy listeners handle downstream connections.

I'm running an Envoy Gateway workload installed with
version `1.8.3` of the [official Helm chart](https://gateway.envoyproxy.io/v1.8/install/install-helm/) 
which in turn uses the `envoyproxy/envoy` image with the version tag `distroless-v1.38.3`.

I introduced an object with the following content for the Gateway named `zq-lu` that I use for the website
https://zq.lu:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: ClientTrafficPolicy
metadata:
  name: zq-lu
  namespace: web
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: zq-lu
  tls:
    ecdhCurves:
    - X25519MLKEM768
    - X25519
    - P-256
    minVersion: '1.3'
```

Once the resource has been created and its Accepted condition becomes True, Envoy is configured to
support the groups in the listed order. When the client also supports it, the TLS handshake can 
negotiate the hybrid X25519MLKEM768 group. Clients that do not support it can fall back to regular
X25519, and then to P-256 for a truly vintage key-exchange experience.

X25519MLKEM768 combines the X25519 key-agreement algorithm with the post-quantum
ML-KEM-768 key-encapsulation mechanism.
This protects the key-establishment phase against “harvest now, decrypt later” attacks, 
where an attacker records encrypted traffic today in the hope of decrypting it in the future
when there might be quantum computers that make such attacks feasible.

The configuration also enforces at least version 1.3 of TLS, which excludes some very old clients.
Given that TLS 1.3 was standardised in 2018, one could argue that they had it coming.

There are several online tools that check the post-quantum aspects of your TLS configuration.
I used https://quready.com/check/ which reported the expected results.
