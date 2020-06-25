---
layout: posts
title: "Troubleshoot OpenStack Networking Like a Netdev Kernel Developer - Part 1"
date: 2019-10-25
categories: [blog]
tags: [ neutron, openstack, networking, kernel ]
excerpt_separator: "<!-- more -->"
author: mauroseb
---

## Introduction

Firstly, I would like to mention that many times I had the chance to work next to some brilliant people at Red Hat, and during the process could learn some interesting techniques when it comes to troubleshoot network issues in complex environments like OpenStack, where there are dozens or even hundreds of virtual devices, overlay networks involved, featured smart NICs, and more. Hopefully this article can help to give not just a boring collection of front line tales but also give shape to an approach that would help to sort out similar problems.

It should be noted that even though most examples in the series involve OpenStack environments, the approaches and techniques discussed would hopefully help with networking issues in **any** linux based environment, with or without OpenvSwitch or OpenStack in the picture.

Lastly this is the first article from hopefully many, therefore will end up in the first example after covering the basics. More to follow.
<!-- more -->

## General Approach

A ton has been written on the matter and arguably many methods and techniques may compete to be proven effective under specific circumstances. But firstly it has to be acknowledged that even if one strives to abide to a method as much as possible, the complexity of modern software systems makes the process not precisely deterministic, and that at the time there is critical impact the expert may choose the tools that her/his experience is dicating in order to address the problem first in the fastest manner, and then chase the root cause. With that said, I personally find Google SRE Guide Troubleshooting section quite comprehensive[^1] and references the hypothetico-deductive model as proposed method. But there are many other sources for methodologies too. There are techniques exclusive to root cause analysis like the simple _5 whys_, _RPR_, and other covered by _Problem Management_ domain in _ITILv3_ literature.

The second point that has to be acknowledged is that no matter what technique it is, there is a common ground for all. Most agree that a thorough observation and data gathering has to take place first. Which is what I am going to focus on in the following section.

<img src="https://imgs.xkcd.com/comics/networking_problems.png" alt="plotnet sample PNG" style="width:400px;align:center;"/>


The follwoing method is assuming a basic triage on the subject problem has taken place and it is worth a deeper analysis. I do not intend to cover basic troubleshooting of network connectivity issues (for what you can also find good resources[^2]), where one would normally start checking IP configuration, routing and so forth. Also it is assumed that there was a working environment in an earlier stage, and now an unsual and/or erratic behavior is manifestating. 

As mentioned before, the level of knowledge of whom applies the method matters, picking a sensitive starting point, depth of the analysis in every step, so on so forth, depending on time/resource constraints and on the overall description of the issue can be valueable. Nonetheless it is also important not to be biased by past experiences and in general be skeptical that what is being witnesses now shares causality with previous events. This especially holds true for networking problems. 

Finally the following is definitely flawed and incomplete (mea culpa) but may still come handy for my own forgetful mind and luckily some other reader.


### 1. Understand the virtual and physical layout

To understand the problem we need to have a clear picture of the path the packets need to traverse. Without that, the search for a root cause will be partial and inaccurate. Taking note of **ALL** the network components, virtual or physical, is paramount. Following is a basic example of an instance running in an OpenStack compute node:


{% highlight shell %}
   +-------+
   |       |
   | PF 0  +---+++++++++     +++++++++++++  +++++++++++++   +++++++++++++   +++++++++++++
   | (eth0)|   +       +     +           +  +           +   +           +   +           +
   +-------+   +       +     +   (VLAN)  +  +  (VxLAN)  +   +  linux    +   +   virtio  +
   +-------+   + bond0 +---- +    OvS    +--+-   OvS    +---+  bridge   +---+   guest   +
   |       |   +       +     +           +  +           +   +  (qbr)    +   +           +
   | PF 1  +---+++++++++     +++++++++++++  +++++++++++++   +++++++++++++   +++++++++++++
   | (eth1)|                            (patch)    (qbo-qbv veths)      (tap)
   +-------+
   
   COMPUTE HOST
   
{% endhighlight %}



In the diagram above the OvS circuitry is a bit more complex because it is performing VLAN tagging/untagging of the "tenant" network on **br-ex** (OvS bridge) internal port, which in turn carries VxLAN traffic, that is then forwarded internally to the **br-tun** where the VxLAN tunnel endpoint lives (for short VTEP - it holds the IP address of the previously mentioned internal port), and terminates each **VNI** corresponding to each tenant, then the traffic is forwarded via OvS internal patches to the **br-int** bridge that in turn forwards the traffic to the instance's qvo veth device.

For the same purpose, there is an excellent tool from Jiri Benc: **plotnetcfg**[^3]. To run it needs either root privileges or **CAP_SYS_ADMIN** and **CAP_NET_ADMIN** capabilities. The tool will create an output file in dot format, that can then be converted for example to PNG format with the **dot** command.

{% highlight shell %}
   # dnf install -y plotnetcfg
   # plotnetcfg > layout.out
   # dot -Tpng layout.out > layout.png
{% endhighlight %}

The former will create a picture like the following:

<img src="/images/plotnet_sample.png" alt="plotnet sample PNG" style="width:1000px;"/>

Actually there are many different formats to choose as output:

{% highlight shell %}
  -Tbmp        -Tcmapx_np   -Tfig        -Timap       -Tjpeg       -Tmp         -Tplain-ext  -Tps2        -Ttiff       -Tvmlz       -Txdot1.4    
  -Tcanon      -Tdot        -Tgtk        -Timap_np    -Tjpg        -Tpdf        -Tpng        -Tsvg        -Ttk         -Tx11        -Txdot_json  
  -Tcmap       -Tdot_json   -Tgv         -Tismap      -Tjson       -Tpic        -Tpov        -Tsvgz       -Tvdx        -Txdot       -Txlib       
  -Tcmapx      -Teps        -Tico        -Tjpe        -Tjson0      -Tplain      -Tps         -Ttif        -Tvml        -Txdot1.2  
{% endhighlight %}

#### Hardware architecture

If there is a performance problem, a point is to check that hardware architecture of the involved nodes: improper NIC specifications for the configuration, wrong CPU assignments or isolation, slow PCI bus, NUMA not taken into account, and other.


### 2. Reproduce it

One of the first questions I normally ask is if there is a clear set of steps that can reproduce the problem. If so, it will simplify the witch hunt considerably for you and any other involved party. This is a little short cut which is part of the initial observation. If there is one, great. 

Determine which recurrence, if it happens in only one system, in many, time patterns, etc. Sometimes this is not possible as the problem only shows up sporadically and under unknown, apparently non-deterministic conditions. 

Some times it is just enough to use **iperf3** , **netperf** or similar tools to display the problem. Also often times is difficult to locate resources to reproduce a _production-like_ environment as it may use expensive equipment. Hence the use of virtual reproducers is a common case, unless the involved pieces of hardware are also part of the problem.
 
 
### 3. Test initial conditions

The reproduction of the problem can depend on multiple factors like hardware architecture, NIC vendor/model, firmware version, OS version, kernel version, NIC driver version, physical network devices (switches, load balancers and routers). Many times permutation of any of these components is helpful to narrow down the problem to a particular component before delving deeper into the analysis. 

One of the first actions I normally try is to try to reproduce with the latest kernel available (downstream in case of _RHEL_), then the latest upstream kernel, also sometimes the latest kernel in __net-next__ tree of the linux kernel (which will become part of the next upstream linux kernel release) if there is any promising commit. 

For _RHEL_:

  * **Test latest downstream kernel.** You can check easily in CDN for the latest and update it if it is newer than the version in the system.
  
  * **Test lastest upstream stable kernel.** The easiest here is to leverage **elrepo** repository which provides an RPM for Centos and RHEL distros built from the **mainline** stable branch of Linux Kernel Archive and thus named **kernel-ml** to avoid conflict with RHEL stock kernels. In the following example I am installing the RPM for major version 7 and setting grub to boot from it only once as we just want to test a reproducer and go back to the default kernel:
  
  {% highlight shell %}
    # rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
    # yum install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
    # yum --enablerepo=elrepo-kernel install kernel-ml
    # grub2-reboot 0
  {% endhighlight %}

  * **Test linux-next kernel.** Similarly the branches that are already accepted for the next stable release can be tested as follows. 
  
  
  {% highlight shell %}
    $ git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
    $ cd linux
    $ git remote add linux-next https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
    $ git fetch linux-next
    $ git fetch --tags linux-next
  {% endhighlight %}

  Now a specific **linux-next** tag can be checked out and built[^4]. Alternatively the **net-next** branch can be used.

  {% highlight shell %}
    $ git remote add net git://git.kernel.org/pub/scm/linux/kernel/git/davem/net.git
    $ git fetch net
  {% endhighlight %}
 
  And just like that one can retry and discard tons of already fixed bugs or include new features that could improve the situation.

Similar approach could be adopted for NIC firmware, drivers or even testing with a different NIC hardware if resources permit. 

If the layout determined at step 2. is too complex. Chopping down the devices and test if the issue can still be reproduced will remove false positives from the way. In example, if the issue happens with a bond device, does it still happen if we use directly one leg, or without bond at all ?  If the answer is yes, then we continue with the next piece to chop.

Just by doing that one can discard hundreds of bugs and enhancements that have been already fixed and incorporated in the latests releases, hardware/firmaware/driver issues that affect a single vendor, and so forth. 

Actions like these have been by far the fastest way to identify existing bugs. Just by knowing it is not happening in a given kernel version, means that we only need to backport certain fix to a downstream kernel (in case of RHEL)  or that the fix will be released soon upstream in case of using the **linux-next** kernel.

There is some extra work to identify which commit or set of commits are needed to solve the problem. To start with, one could list the commits between the problematic and the fixed kernel:

{% highlight shell %}
  $ git log --oneline v5.6-rc2..v5.6-rc3 net/ include/net/
  3dc55dba6723 Merge git://git.kernel.org/pub/scm/linux/kernel/git/netdev/net
  3a20773beeee net: netlink: cap max groups which will be considered in netlink_bind()
  98bda63e20da net: disable BRIDGE_NETFILTER by default
  16a556eeb7ed openvswitch: Distribute switch variables for initialization
  46d30cb1045c net: ip6_gre: Distribute switch variables for initialization
  161d179261f9 net: core: Distribute switch variables for initialization
  41f57cfde186 Merge git://git.kernel.org/pub/scm/linux/kernel/git/bpf/bpf
  303d0403b8c2 udp: rehash on disconnect
  06f5201c6392 net/tls: Fix to avoid gettig invalid tls record
  33c4acbe2f4e bridge: br_stp: Use built-in RCU list checking
...
{% endhighlight %}

In case you missed it, git also provides __bisect__ subcommand which helps in pinpointing the culprit out of those commits by testing good and bad versions. Nevertheless the software and hardware vendors would normally take care of the search at this level. Once the commit or commits needed are known I can get in which branch was applied in the downstream tree:

{% highlight shell %}
  $ git branch --contains cc2af34db9a5b5222eefdc25fd1265e305df9f2e
  * (HEAD detached at kernel-3.10.0-1122.el7)    
{% endhighlight %}


### 4. Performance metrics

Dealing with a performance issue with a broad description, normally would involve also checking the output of performance and metrics monitoring tools, looking for stats like RX/TX packet counts and sizes in each interface involved, error counts and in general what counters are moving network wise to understand if they are or not part of the problem. 

There are hundreds of command line tools to chose here but for the sake of simplicity I will focus on the readings of __ethtool__ and __sar__ (the later because is the most widespread accross systems). It is normal to find environments with proper performance tooling like stacks combining __collectd__, __prometheus__, __grafana__, __ganglia__, or any __rrdtool__ derivative. One interesting tool to consider is __pcp__ (performance co-pilot [^5]). It does not really matter which tool to use as long as one can get the metrics that is after (for a comprehensive list of Linux command line tools check out the mind blowing work of Brendan Gregg [^6][^7]).


### 5. Tracing

Once we have identified that a bottleneck resides in certain userland process or kernel thread, what happens next? Understanding what that task is doing is a fair next step which will reveal which part of the code is generating the noise. 

For that more tools come handy: **perf** is probably one I used the most in this cases. Also facilities like dynamic kernel tracing or **ftrace**, either through scripting directly or through **trace-cmd** command line tool.

Lastly, I must mention **eBPF** (extended Barkley Packet Filter, originially named after BSD's BPF, however radically different) which is remarkably valuable and versatile kernel facility that was created for tracing purposes, but quickly became a swiss-army knife within the kernel that allows the user to create his own code and run it in a JIT compiler
in kernel land. Scary? Of course.

## OpenStack Network Architecture

OpenStack is a dynamic product and by design fully maleable to fit one's needs. Virtually every component is plugin based and can be interchangable by something else. In regard to networking it is no differen. The main project, Neutron can handle a large range of _ML2_ plugins (core component) from different opensource projects or from different vendors, and inside every ML2 plugin most of the networking services are also designed as pluggable as long as they respect well most of the defined Neutron API. With this said, this article mainly refers to the stock network layout for Neutron which is ML2 with OpenvSwitch (OvS) mechanism driver (recently OVN has been introduced but still uncommon in the field), as it has been the one where the problems covered below arised.

The architecture of ML2/OvS has been largely documented and described (just to list some references: [^8][^9][^10][^11] ) so the assumption is that the reader is already familiar with it. The following diagram illustrates to some extent what a typical OpenStack compute and networker node layout looks like in order to proceed with the cases' analysis.

<img src="/images/neutron_architecture.png" alt="Compute Network Layout" style="width:1000px;"/>

The best practices dictate to use at least 6 VLANs (+1 optional for management) that would be normally trunked to at least two physical node NICs so they can be bonded together, however the installer is totally flexible and allows the user to be creative, use multiple independant NICs, flat networks, etc. Typical diagram is shown in the picture below.

<img src="/images/6-vlan-arch.png" alt="Node connectivity" style="width:1000px;"/>


## Detecting Software Segmentation

During the past years I stumbled a few times upon network driver bugs that prevent _GSO_ (generic segmentation offloading) to work properly each time on a different driver: __ixgbe__, __mlx5_core__ , __mlx4_core__ in combination with VxLAN encapsulation on top of a VLAN frame.


## References

[^1]: https://landing.google.com/sre/sre-book/chapters/effective-troubleshooting/

[^2]: https://www.redhat.com/sysadmin/beginners-guide-network-troubleshooting-linux

[^3]: https://github.com/jbenc/plotnetcfg

[^4]: https://kernelnewbies.org/KernelBuild

[^5]: https://pcp.io/docs/guide.html

[^6]: http://www.brendangregg.com/

[^7]: http://www.brendangregg.com/blog/2014-11-22/linux-perf-tools-2014.html

[^8]: https://www.rdoproject.org/networking/networking-in-too-much-detail/

[^9]: https://docs.openstack.org/neutron/latest/

[^10]: https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/13/html-single/networking_guide/index

[^11]: https://www.slideshare.net/nyechiel/neutron-networking-with-red-hat-enterprise-linux-openstack-platform
