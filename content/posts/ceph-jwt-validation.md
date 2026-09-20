---
title: Convincing the Ceph RADOS Gateway to validate modern JWTs
author: Noa Resare
date: 2026-09-20
---

I have been building a new Kubernetes cluster for my home lab in the last few days. Lots of fun, and lots of
interesting technical challenges. But tonight I found a workaround that was complex enough to warrant a blog post.

My new Kubernetes cluster is built using the upstream `kubeadm` tool. If you build a new cluster today and 
specify `encryptionAlgorithm: ECDSA-P256` in your cluster configuration, you will not only get ECDSA TLS certificates
but also workload identity tokens signed using the `ES256` algorithm.

Overall, the new tokens have worked really well, up until I tried using them to authenticate with the Ceph RADOS Gateway
that implements part of the AWS Security Token Service API. I use the gateway to be able to connect with workload 
identities to an S3-compatible Ceph object store in the local cluster, but apparently there was something wrong with 
the new tokens that my cluster generates. The gateway log simply said `Signature can not be validated 
with the JWKS present`. How helpful.

However, with open source it is possible to check what is going 
on [under the hood](https://github.com/ceph/ceph/blob/v20.2.4/src/rgw/rgw_rest_sts.cc#L751). It turns out that 
in Ceph 20.2.4, ECDSA algorithm will fail to validate if the public key is published using a bare JWK.
However, I noticed something interesting looking at the code: If the public key is instead wrapped 
in an X.509 certificate, the code seems to be able to read the key just fine. Too bad that the Kubernetes 
API server doesn't provide the key wrapped in this way. Then I realised that I am running a small specialised proxy 
to make the public token signing keys available on the open internet. I could modify the proxy to add a wrapped version of 
the key as it is being served, and we could then convince the gateway to validate the key that Kubernetes issued. Said 
and done, the proxy now has an [optional feature](https://github.com/nresare/jwksproxy/pull/23) to add an extra `x5c` 
field to the keys that the Kubernetes cluster publishes. The extra field contains the same key but wrapped in an X.509 
certificate, and now the Ceph RADOS Gateway can validate my `ES256` tokens.