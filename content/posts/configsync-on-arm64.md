---
title: Running config-sync on arm64
author: Noa Resare
date: 2024-11-24
---

First off, credit to Sean K.H. Liao that published a [blog post](https://seankhliao.com/blog/12023-10-08-kpt-config-sync-arm64/) that I based this effort on.

A while back I had the opportunity to work with [config-sync](https://cloud.google.com/kubernetes-engine/enterprise/config-sync/docs/overview),
a system to synchronize objects in Kubernetes clusters with manifests stored in a git repository.
Now that I'm experimenting with setting up a Kubernetes cluster with some Raspberry Pis at home, 
I wanted to use config-sync, and I was a bit sad to realise that the published images by the upstream maintainers are built
for the amd64 architecture only. However, since the source is available I should be able to build my own images.
This page contains some details on how I did that, with instructions for anyone else that might want to use my work.

## Just the installation instructions

You probably should not trust strangers on the internet to install a bunch of binaries into your infrastructure, 
but if you are foolish enough to do that, here are the instructions:

1. Pick up the `config-sync-manifest.yaml` from the latest release on https://github.com/nresare/kpt-config-sync/releases
2. Install onto your cluster with `kubectl apply -f config-sync-manifest.yaml`

## Changes made

The upstream kpt-config-sync repository contains some code and build infrastructure that utilises some third party
container images and binaries. The latest commits on https://github.com/nresare/kpt-config-sync/commits/v1.19.2-arm64/ contains 
changes to use different sources for the binaries and container images such that both arm64 and amd64 is supported.

While it makes perfect sense for the team that maintains config-sync to publish their own versions of software it
depends on to be able to meet stringent requirements for vulnerabilities patching, it is unfortunate that the versions 
being built are only targeting the amd64 architecture. The majority of the commits in the branch referenced above 
are changing those external dependencies to instead pull their content from the upstream sources as those are already
built for multiple architectures.

The only structural change to the build comes in [this commit](https://github.com/nresare/kpt-config-sync/commit/92200507634c69eb705ec8734c277117f239fa4c)
where I move the download and signature verification of the external tools `kustomize` and `helm` from the top level 
`make` invocation into the `docker build` step. This enables the installation script to pick the architecture for the 
external artefacts that matches the architecture currently being built.

### git-sync

The [git-sync](https://github.com/kubernetes/git-sync) project build infrastructure support building artefacts for 
multiple architectures, but outputs the resulting container images with architecture specific tags. To conveniently
support referencing git-sync in a way that automatically picks the right architecture in the config-sync manifests,
I manually built git-sync for arm64 and amd64, pushed to docker hub and then used `docker manifest` to create a 
manifest that supports both architectures using the following steps:

1. `git checkout git@github.com:kubernetes/git-sync.git && cd git-sync && git checkout v2.4.2`
2. `export REGISTRY=a_registry_path_that_you_have_push_rights_to`
3. `make push VERSION=v2.4.2 GOARCH=amd64`
4. `make push VERSION=v2.4.2 GOARCH=arm64`
5. `docker manifest create $REGISTRY/git-sync:v4.2.4 --amend $REGISTRY/git-sync:v4.2.4__linux_amd64 --amend $REGISTRY/git-sync:v4.2.4__linux_arm64`
6. `docker manifest push $REGISTRY/git-sync:v4.2.4`

## Next steps

Obviously, having upstream integrate these changes and support arm64 as well as amd64 would be the much perferred
variant. There is a [request for this](https://github.com/GoogleContainerTools/kpt-config-sync/issues/893) that has 
been open for a long time, so I'm not holding my breath. I'm hoping that we can land some upstream changes such that
at least the set of patches needed to build a version that supports multiple architectures is smaller.