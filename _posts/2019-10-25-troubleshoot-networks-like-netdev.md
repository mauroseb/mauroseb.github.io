---
layout: posts
title: "Troubleshoot OpenStack Networking Like a Netdev Kernel Developer - Part 1"
date: 2019-10-25
categories: [blog]
tags: [ neutron, openstack, networking, kernel ]
---

## Introduction

Firstly, I would like to mention that many times I had the honor to work next to some brilliant people at Red Hat, and during the process had the chance to learn some very interesting techniques when it comes to troubleshoot network issues in complex environments like OpenStack where there are tens or even hundreds of virtual devices, overlay networks involved and more. Even though I cannot touch the feet of these people I can still share some lessons learned.


## OpenStack Network Architecture

OpenStack is a dynamic product and by design fully maleable to fit one's needs. Virtually every component can be interchangable with something else. In respect to networking it is no different. The main project Neutron can handle a range of _ML2_ plugins from different upstream projects or from different vendors, and inside every ML2 plugin most of the networking services are designed as pluggable and can also be interchanged. With this said, this article mainly refers to the stock network layout for Neutron which is ML2 with OpenvSwitch (OvS) mechanism driver (recently OVN has been introduced however still uncommon in the field), as it has been the one where the problems covered below arised.

The architecture of ML2/OvS has been largely documented and described (just to list some references: [1][2][3][4] ) so the assumption is that the reader is already familiar with it. However a diagram like the one following is needed to illustrate to some extent what a typical OpenStack compute and networker node layout looks like in order to proceed with the cases' analysis.

![Compute Network Layout](/images/neutron_architecture.png)

## Cases Scenarios

### TCP Retransmissions due to vtap device

### Performance drop in VxLAN

### SNAT Port Exhaustion

### Netlink Messages Overflow

## References

[1] https://www.rdoproject.org/networking/networking-in-too-much-detail/

[2] https://docs.openstack.org/neutron/latest/

[3] https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html-single/networking_guide/index

[4] https://www.slideshare.net/nyechiel/neutron-networking-with-red-hat-enterprise-linux-openstack-platform
