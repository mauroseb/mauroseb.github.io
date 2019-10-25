---
layout: posts
title: "Troubleshoot OpenStack Networking Like a Netdev Kernel Developer"
date: 2019-10-25
categories: [blog]
tags: [ neutron, openstack, networking, kernel ]
---

## Troubleshoot OpenStack Networking Like a Netdev Kernel Developer

### Introduction

Firstly, I would like to mention that many times I had the honor to work next to some brilliant people at Red Hat, and during the process had the chance to learn some very interesting techniques when it comes to troubleshoot network issues in complex environments like OpenStack where there are tens or even hundreds of virtual devices, overlay networks involved and more. Even though I cannot touch the feet of these people I can still share some lessons learned.


### OpenStack Network Architecture

OpenStack is a dynamic product and by definition fully maleable to fit one's needs. Virtually every component can be interchangable with something else. In respect to networking it is no different. The main project Neutron can handle a range of ML2 plugins from different upstream projects or from different vendors, and inside every ML2 plugin most of the networking services are designed as pluggable and can also be interchanged. With this said, this article mainly refers to the stock network layout for Neutron which is ML2/OvS (recently OVN has been introduced however still uncommon), as it has been the one where the problems covered below arised.

The architecture of ML2/OvS has been largely documented and described [1] but the diagram below illustrates to some extent what is fonud in a typical OpenStack compute and networker node.

#### Compute Network Layout

<img src=/images/compute-layout.png alt="Compute Network Layout" >


### Cases Scenarios

#### Performance Issues in VTEP device

#### 

#### SNAT Port Exhaustion

#### 

[1] https://docs.openstack.org/neutron/latest/
