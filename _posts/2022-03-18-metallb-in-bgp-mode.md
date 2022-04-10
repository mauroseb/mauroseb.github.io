---
layout: posts
title: "How to Setup MetalLB in BGP mode"
date: 2022-03-18
categories: [blog]
tags: [ ocp, metallb, bgp, kubernetes, networking ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---
### Intro

 With the release of OpenShift 4.10, the BGP or Border Gateway Protocol mode for the MetalLB operator became generally available.

BGP mode offers a novel way to statelessly load balance client traffic towards the applications running on baremetal and bare metal-like OpenShift clusters, by using standard routing facilities, and at the same time providing high availability for the services.

The purpose of this article is to introduce MetalLB design and goals and cover in more detail this new mode. Throughout this article, I will discuss its configuration and usage, and in addition, provide mechanisms to verify that it is working properly.

This article is available at Red Hat's Hybrid Cloud Blog:
- https://cloud.redhat.com/blog/metallb-in-bgp-mode

<!-- more -->
