---
layout: posts
title: "Troubleshoot OpenStack Networking Like a Netdev Kernel Developer - Part 1"
date: 2019-10-25
categories: [blog]
tags: [ neutron, openstack, networking, kernel ]
---

## Introduction

Firstly, I would like to mention that many times I had the chance to work next to some brilliant people at Red Hat, and during the process could learn some interesting techniques when it comes to troubleshoot network issues in complex environments like OpenStack, where there are dozens or even hundreds of virtual devices, overlay networks involved, featured smart NICs, and more. Hopefully this article can help to give not just a boring collection of front line tales but also give shape to an approach that would help to sort out similar problems.

It should be noted that even though most examples in the series involve OpenStack environments, the approaches and techniques discussed would hopefully help with networking issues in **any** linux based environment, with or without OpenvSwitch or OpenStack in the picture.

Lastly this is the first article from hopefully many, therefore will end up in the first example after covering the basics. More to follow.


## OpenStack Network Architecture

OpenStack is a dynamic product and by design fully maleable to fit one's needs. Virtually every component is plugin based and can be interchangable by something else. In regard to networking it is no differen. The main project, Neutron can handle a large range of _ML2_ plugins (core component) from different opensource projects or from different vendors, and inside every ML2 plugin most of the networking services are also designed as pluggable as long as they respect well most of the defined Neutron API. With this said, this article mainly refers to the stock network layout for Neutron which is ML2 with OpenvSwitch (OvS) mechanism driver (recently OVN has been introduced however still uncommon in the field), as it has been the one where the problems covered below arised.

The architecture of ML2/OvS has been largely documented and described (just to list some references: [1][2][3][4] ) so the assumption is that the reader is already familiar with it. However a diagram like the one following is needed to illustrate to some extent what a typical OpenStack compute and networker node layout looks like in order to proceed with the cases' analysis.

<img src="/images/neutron_architecture.png" alt="Compute Network Layout" style="width:1200px;"/>

## General Approach (if any?)

As disclaimer there should be definitely a better standard approach to engage problems of this nature than the one  following, however it should still be useful, as at least I have myself applied it successfully in many occasions.

Also there is no general recipe, and the starting point where to look may vary based on the overall description of the issue. It can be more often than not something utterly generic like _low throughput between A and B_, or more specfic isolated observations like: packet drops in X device, retransmissions, high CPU in one or more cores, and so on so forth. In a nutshell, the problem quickly winds up into finding a bottleneck, a misconfiguration or a bug.

For the sake of simplicity I will try to keep the list short:

### 1. Reproduce it

First and foremost if there is an identified set of steps that can reproduce the problem will simplify the witch hunt considerably for you and any other involved party. To determine which recurrence, if it happens in only one system, in many, time patterns, etc. Sometimes this is not possible as the problem only shows up sporadically and under unknown conditions. Some times it is just enough to use **iperf3** , **netperf** or similar tools to display the problem. Many times is difficult to locate resources to reproduce a "production"-like environment as it may use multiple expensive equipments. Hence the use of virtual reproducers is often the case, unless the involved pieces of hardware are part of the problem.
 
### 2. Understand the virtual and physical layout

To understand the problem we need to have a clear picture of the path the packets need to traverse. Without that, the search for a root cause will be partial and inaccurate. Taking note of **ALL** the network components, virtual or physical, is paramount. Following is a basic example of an instance running in an OpenStack compute node:

        +-------+
        |       |
        | PF 0  +---+++++++++     +++++++++++++  +++++++++++++   +++++++++++++   +++++++++++++
        | (eth0)|   +       +     +           +  +           +   +           +   +           +
        +-------+   +       +     +   (VLAN)  +  +  (VxLAN)  +   +  linux    +   +           +
        +-------+   + bond0 +---- +    OvS    +--+-   OvS    +---+  bridge   +---+  virtio   +
        |       |   +       +     +           +  +           +   +  (qbr)    +   +  guest    +
        | PF 1  +---+++++++++     +++++++++++++  +++++++++++++   +++++++++++++   +++++++++++++
        | (eth1)|                            (patch)    (qbo-qbv veths)      (tap)
        +-------+

        COMPUTE HOST



In the diagram above the OvS circuitry is a bit more complex because it is performing VLAN tagging/untagging of the "tenant" network on ```br-ex``` (OvS bridge) internal port, which in turn carries VxLAN traffic, that is then forwarded internally to the ```br-tun``` where the VTEP lives (with the IP address of the previously mentioned internal port), and terminates each ```VNI``` corresponding to each tenant, then the traffic is forwarded via OvS internal patches to the ```br-int``` bridge that in turn forwards the traffic to the instance's qvo veth device.

For the same purpose, there is an excellent tool from Jiri Benc: **plotnetcfg**[6]. To run it needs either ```root``` ileges or ```CAP_SYS_ADMIN``` and ```CAP_NET_ADMIN``` capabilities. The tool will create an output file in ```dot``` format, that can then be converted to ```PNG``` format with the **dot** command.

        # dnf install -y plotnetcfg
        # plotnetcfg > layout.out
        # dot -Tpng layout.out > layout.png

The former will create a picture like the following:

<img src="/images/plotnet_sample.png" alt="plotnet sample PNG" style="width:1200px;"/>

Actually there are many different formats to choose as output:

        -Tbmp        -Tcmapx_np   -Tfig        -Timap       -Tjpeg       -Tmp         -Tplain-ext  -Tps2        -Ttiff       -Tvmlz       -Txdot1.4    
        -Tcanon      -Tdot        -Tgtk        -Timap_np    -Tjpg        -Tpdf        -Tpng        -Tsvg        -Ttk         -Tx11        -Txdot_json  
        -Tcmap       -Tdot_json   -Tgv         -Tismap      -Tjson       -Tpic        -Tpov        -Tsvgz       -Tvdx        -Txdot       -Txlib       
        -Tcmapx      -Teps        -Tico        -Tjpe        -Tjson0      -Tplain      -Tps         -Ttif        -Tvml        -Txdot1.2  

 
### 3. Test initial conditions

The reproduction of the problem can depend on multiple factors like hardware architecture, NIC vendor/model, firmware version, OS version, kernel version, NIC driver version, physical network devices (switches, load balancers and routers). Many times permutation of any of these components is helpful to narrow down the problem to a particular component before delving deeper into the analysis. 

One of the first actions I normally try is to try to reproduce with the latest kernel available (downstream in case of ```RHEL```), then the latest upstream kernel, also sometimes the latest kernel in ```net-next``` tree of the linux kernel (which will become part of the next upstream linux kernel release) if there is any promising commit. 

For ```RHEL```:

  * **Test latest downstream kernel.** You can check easily in CDN for the latest if you are using a local repo.
  
  * **Test lastest upstream stable kernel.** The easiest here is to leverage ```elrepo``` repository which provides an RPM for ```Centos``` and ```RHEL``` distros built from the ```mainline``` stable branch of Linux Kernel Archive and thus named ```kernel-ml``` to avoid conflict with RHEL stock kernels. In the following example I am installing the RPM for major version 7 and setting grub to boot from it only once as we just want to test a reproducer and go back to the default kernel:
  
          # rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
          # yum install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
          # yum --enablerepo=elrepo-kernel install kernel-ml
          # gru2-reboot 0
     
    And just like that one can retry and discard tons of already fixed bugs or include new features that could improve the situation.

Similar approach could be adopted for NIC firmware, drivers or even testing with a different NIC hardware if resources permit. 

If the layout determined at step 2. is too complex. Chopping down the devices and test if the issue can still be reproduced will remove false positives from the way. In example, if the issue happens with a bond device, does it still happen if we use directly one leg, or without bond at all ?  If the answer is yes, then we continue with the next piece to chop.

Just by doing that one can discard hundreds of bugs and enhancements that have been already fixed and incorporated in the latests releases, hardware/firmaware/driver issues that affect a single vendor, and so forth. 

Actions like these have been by far the fastest way to identify existing bugs. Just by knowing it is not happening in the kernel version X, means that we only need to backport certain fix to a downstream kernel (in case of ```RHEL```)  or that the fix will be released soon in case of using the ```net-next``` kernel. 

Of course there is some extra work to identify which commit or set of commits are needed to solve the problem, like using ```git bisect```, exploring the repo logs, and some other manual tasks. However the software and hardware vendors should normally take care of that. As a quick example, to search through branches for a given commit:

          $ git branch --contains cc2af34db9a5b5222eefdc25fd1265e305df9f2e
          * (HEAD detached at kernel-3.10.0-1122.el7)
          

 
### 4. Performance metrics

Dealing with a performance issue with a broad description like _low troughput..._, normally would involve also checking the output of performance and metrics monitoring tools, looking for stats like RX/TX packet counts and sizes in each interface involved, error counts and in general what counters are moving network wise to understand if they are or not part of the problem. 

There are hundreds of command line tools to chose here but for the sake of simplicity I will focus on the readings of ```ethtool``` and ```sar``` (the later because is the most widespread accross systems). It is normal to find environments with proper performance tooling like stacks combining ```collectd```, ```prometheus```, ```grafana```, ```ganglia```, or any ```rrdtool``` based plotter. One interesting tool is also ```pcp``` (performance co-pilot [5]). It does not really matter which tool to use as long as one can get the metrics that is after (for a comprehensive list of Linux command line tools check out the mind blowing work of Brendan Gregg [7][8]).

### 5. Tracing

Once we have identified that a bottleneck resides in certain userland process or kernel thread, what happens next? Understanding what that task is doing is a fair next step which will reveal which part of the code is generating the noise. 

For that more tools come handy: **perf** is probably one I used the most in this cases. Also facilities like dynamic kernel tracing or ```ftrace```, either through scripting directly or through **trace-cmd** command line tool.

Lastly, I must mention ```eBPF``` (extended Barkley Packet Filter, originially named after BSD's BPF, however radically different) which is remarkably valuable and versatile kernel facility that was created for tracing purposes, but quickly became a swiss-army knife within the kernel that allows the user to create his own code and run it in a JIT compiler
in kernel land. Scary? Of course.


## Detecting Software Segmentation

During the past years I stumbled a few times upon network driver bugs that prevent ```GSO``` (generic segmentation offloading) to work properly (each time on a different driver: ```ixgbe```, ```mlx5_core```, ```mlx4_core```, can't recall the last one yet), in combination with ```VxLAN``` encapsulation on top of a ```VLAN```.


## References

[1] https://www.rdoproject.org/networking/networking-in-too-much-detail/

[2] https://docs.openstack.org/neutron/latest/

[3] https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html-single/networking_guide/index

[4] https://www.slideshare.net/nyechiel/neutron-networking-with-red-hat-enterprise-linux-openstack-platform

[5] https://pcp.io/docs/guide.html

[6] https://github.com/jbenc/plotnetcfg

[7] http://www.brendangregg.com/

[8] http://www.brendangregg.com/blog/2014-11-22/linux-perf-tools-2014.html

