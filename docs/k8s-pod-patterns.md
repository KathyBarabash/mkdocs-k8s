---
title: Overview
summary: 
authors:
    - Kathy Barabash
date: 2022-01-14
---

# Overview

While ususally each k8s pod contains a single container, there are several usefull patterns for deploying multiple containers in a single pod. In most cases, the patterns call for adding an auxiliarry container alongside the main application container in a pod. As all the containers in a pod share their k8s, network, and storage namespaces and can therefore connect to the main container on `localhost` and mount the same volumes the main container can mount, the auxilarry containers can transparently add/augment/adapt application functionality. This helps decoupling development of certain application wide aspects from the main application logics. 

The following patterns are widely known and endorsed by the community:

1. [__Sidecars__](k8s-pod-patterns-sidecar.md) extend and enhance
1. [__Ambassadors__](k8s-pod-patterns-ambassador.md) proxy and represent
1. [__Adapters__](k8s-pod-patterns-adapter.md) normalize and present
1. [__Initcontainers__](k8s-pod-patterns-init.md)

## References

1. Designing Distributed Systems [book](https://azure.microsoft.com/en-us/resources/designing-distributed-systems/) by Brendan Burns

