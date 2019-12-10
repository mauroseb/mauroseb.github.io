---
layout: posts
title: "Troubleshoot OpenStack Networking Like a Netdev Kernel Developer - Part 1"
date: 2019-10-25
categories: [blog]
tags: [ neutron, openstack, networking, kernel ]
---

## Introduction

Firstly, I would like to mention that many times I had the chance to work next to some brilliant people at Red Hat, and during the process could learn some interesting techniques when it comes to troubleshoot network issues in complex environments like OpenStack, where there are dozens or even hundreds of virtual devices, overlay networks involved, featured smart NICs, and more. Hopefully this article can help to sort out similar problems.


## OpenStack Network Architecture

OpenStack is a dynamic product and by design fully maleable to fit one's needs. Virtually every component is plugin based and can be interchangable by something else. In regard to networking it is no differen. The main project, Neutron can handle a large range of _ML2_ plugins (core component) from different opensource projects or from different vendors, and inside every ML2 plugin most of the networking services are also designed as pluggable as long as they respect well most of the defined Neutron API. With this said, this article mainly refers to the stock network layout for Neutron which is ML2 with OpenvSwitch (OvS) mechanism driver (recently OVN has been introduced however still uncommon in the field), as it has been the one where the problems covered below arised.

The architecture of ML2/OvS has been largely documented and described (just to list some references: [1][2][3][4] ) so the assumption is that the reader is already familiar with it. However a diagram like the one following is needed to illustrate to some extent what a typical OpenStack compute and networker node layout looks like in order to proceed with the cases' analysis.

![Compute Network Layout](/images/neutron_architecture.png)


## General Approach (if any?)

As disclaimer there should be definitely a better standard approach to engage problems of this nature than the one  following, however it should still be useful, as at least I have myself applied it successfully in many occasions.

Also there is no general recipe, and the starting point where to look may vary based on the overall description of the issue. It can be more often than not something utterly generic like _low throughput between A and B_, or more specfic isolated observations like: packet drops in X device, retransmissions, high CPU in one or more cores, and so on so forth. In a nutshell, the problem quickly winds up into finding a bottleneck, a misconfiguration or a bug.

For the sake of simplicity I will try to keep the list short:

 1. **Reproduce it.** First and foremost if there is an identified set of steps that can reproduce the problem will simplify the witch hunt considerably. And determine which recurrence, if it happens in only one system, in many. Sometimes this is not possible as the problem only shows up sporadically and under unknown conditions. Some times it is just enough to use **iperf3** , **netperf** or similar tools to display the problem. 
 
 2. **Understand the virtual and physical layout**. To understand the problem we need to have a clear picture of the path the packets need to cross. Without that the search for a root cause will be partial and inaccurate. Taking note of **all** the network components, virtual or physical, is paramount.
 
 3. **Quickly test different scenarios.** The reproduction of the problem can depend on multiple factors like hardware architecture, NIC vendor/model, firmware version, OS and kernel version, NIC driver version, physical network devices configuration and firmware/software versions (switches, load balancers and routers). Many times permutation of any of these components is helpful to narrow down the problem. For instnance, one of the first actions I normally try is use the latest kernel available (downstream in case of ```RHEL```), then the latest kernel upstream, and sometimes the latest kernel in ```net-next``` branch of the kernel which will become the next upstream linux kernel release. Same goes for NIC firmware and drivers. Just by doing that one can discard hundreds of bugs and enhancements that have been already fixed and incorporated in the latests releases. This has been by far the way to identify bugs. Just by knowing it is not happening in the kernel version X, means that we only need to backport certain fix to a downstream kernel in case of ```RHEL```  or that the fix will be released soon in case of using the ```net-next``` kernel. Of course there is some extra work to identify which commit or set of commits are needed to solve the problem however the vendors should take care of that, we still can touch base on that later on.
 
 4. **Performance metrics.** Starting from a performance related description like _low troughput..._, normally would start with checking the output of performance and metrics tooling. There are hundreds of command line tools to chose here but for simplicity we can use the good old ```sar``` only because is the most widespread accross systems. It is normal to find environments with proper performance tooling like ```prometheus```, ```grafana```, ```ganglia```, or any ```rrdtool``` based plotter. One interesting tool to check is also ```pcp``` (performance co-pilot [5]). It does not really matter what tool to use as long as we can read simple metrics (for a comprehensive list of Linux command line tools check out the mind blowing work of Brendan Gregg [6][7]).


### Detecting Software Segmentation

During the past years I stumbled at least 4 times upon network driver bugs that prevent ```GSO``` to work properly (each time a different driver: ```ixgbe```, ```mlx5_core```, ```mlx4_core```, can't recall the last one yet).


### TCP Retransmissions due to veth device

### Performance drop in VxLAN

### SNAT Port Exhaustion

### Netlink Messages Overflow

### Tracing VNF packet drops


## References

[1] https://www.rdoproject.org/networking/networking-in-too-much-detail/

[2] https://docs.openstack.org/neutron/latest/

[3] https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html-single/networking_guide/index

[4] https://www.slideshare.net/nyechiel/neutron-networking-with-red-hat-enterprise-linux-openstack-platform

[5] https://pcp.io/docs/guide.html

[6] http://www.brendangregg.com/blog/2014-11-22/linux-perf-tools-2014.html

[7] http://www.brendangregg.com/

