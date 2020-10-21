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

Firstly, I would like to mention that it gave me some hard time to find good comprehensive literature in regard to linux networking internals. In general for this specific topic one ends up crawling https://lkml.org/. Most common kernel related books like _"Understanding the Linux Kernel"_ or _"Linux Device Drivers"_ (which dedicates one chapter) do not cover this topic extensively. So I will start by noting the following book as the best source I found so far:

  - ["Linux Kernel Networking: Implementation and Theory" - Rami Rosen](https://www.amazon.nl/Linux-Kernel-Networking-Implementation-Theory/dp/143026196X/ref=sr_1_1?__mk_nl_NL=%C3%85M%C3%85%C5%BD%C3%95%C3%91&dchild=1&keywords=Linux+Kernel+Networking&qid=1596125842&sr=8-1)
  
Moreover, many times I had the chance to work next to some real experts in the matter at Red Hat who are and have been for ages active contributors to the kernel networking stack, and during the process could learn some interesting techniques when it comes to troubleshoot network issues in complex environments like OpenStack, where there are dozens or even hundreds of virtual devices, overlay networks involved, featured smart NICs, and more. Hopefully this article can help to give not just a boring collection of front line tales but also give shape to an approach that would help to sort out similar problems.

It should be noted that even though most examples in the series involve OpenStack environments, the approaches and techniques discussed would hopefully help with networking issues in **any** linux based environment, with or without OpenvSwitch or OpenStack in the picture.

Lastly this is the first article from hopefully many, therefore will end up in the first example after covering the basics. More to follow.
<!-- more -->

## General Approach

A ton has been written on the matter and arguably many methods and techniques may compete to be proven effective under specific circumstances. But firstly it has to be acknowledged that even if one strives to abide to a method as much as possible, the complexity of modern software systems makes the process not precisely deterministic, and that at the time there is critical impact the expert may choose the tools that her/his experience is dicating in order to address the problem first in the fastest manner, and then chase the root cause. With that said, I personally find Google SRE Guide Troubleshooting section quite comprehensive[^1] and references the hypothetico-deductive model as proposed method, which is basically, observing, creating an hypothesis, creating predictions on that hypothesis and put it to test. While from a logical standpoint this is obviously flawed with positive confirmation/reinforcement of the hypothesis, in the empirical world is the way to go. But hey, there are many other sources for methodologies too. There are techniques exclusive to root cause analysis like the simple _5 whys_, _RPR_, and other covered by _Problem Management_ domain in _ITILv3_ literature.

The second point that has to be acknowledged is that no matter what technique it is, there is a common ground for all. Most agree that a thorough observation and data gathering has to take place first. Which is what I am going to focus on in the following section.

<img src="https://imgs.xkcd.com/comics/networking_problems.png" alt="plotnet sample PNG" style="width:50%;align:center;"/>


The follwoing method is assuming a basic triage on the subject problem has taken place and it is worth a deeper analysis. I do not intend to cover basic troubleshooting of network connectivity issues (for what you can also find good resources[^2]), where one would normally start checking IP configuration, routing and so forth. Also it is assumed that there was a working environment in an earlier stage, and now an unsual and/or erratic behavior is manifestating. 

As mentioned before, the level of knowledge of whom applies the method matters, picking a sensitive starting point, depth of the analysis in every step, so on so forth, depending on time/resource constraints and on the overall description of the issue can be valueable. Nonetheless it is also important not to be biased by past experiences and in general be skeptical that what is being witnesses now shares causality with previous events. This especially holds true for networking problems. 

Finally the following is definitely also flawed and incomplete (mea culpa) but may still come handy for my own forgetful mind and luckily some other reader.

## Observation Phase

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
   | (eth1)|                            (patch)  (qvo-qvb veth pair)    (tap)
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

<img src="/images/plotnet_sample.png" alt="plotnet sample PNG" style="width:75%;"/>

Actually there are many other different formats to choose as output.

#### Hardware Architecture

If there is a performance problem, an important point is to check where we are standing in regards to the hardware architecture of the involved nodes. It is common to spot improper NIC specifications or configuration, wrong CPU assignments or isolation if in use (specially in NFV use cases), slow PCI busses, NUMA not taken into account, and similar mistakes. 

So to start with, lets see if the PCI port is proper for the NIC or NICs.

 - Get NIC PCI port information

{% highlight shell %}
    # lshw -c network -businfo
    Bus info          Device      Class          Description
    ========================================================
    pci@0000:01:00.0  em1         network        82599ES 10-Gigabit SFI/SFP+ Network
    pci@0000:01:00.1  em2         network        82599ES 10-Gigabit SFI/SFP+ Network
    ... 
{% endhighlight %}

 - Check the NIC driver and Firmware
{% highlight shell %}
# ethtool -i p2p1 
driver: qede
version: 8.10.10.21
firmware-version: mfw 8.25.7.0 storm 8.20.0.0
expansion-rom-version: 
bus-info: 0000:5f:00.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: no
supports-register-dump: yes
supports-priv-flags: yes
{% endhighlight %}

 - Check the NIC NUMA node
{% highlight shell %}
# lspci -s 03:00.1 -vv | grep NUMA
NUMA node: 1
# cat /sys/bus/pci/devices/0000\:03\:00.1/numa_node
1   
{% endhighlight %}

 - Now I can check the PCI Bus speed. For more than one 10Gbps NIC in the same bus (like dual-por NICs) at least PCI __Gen3__ (Width x8/x16) is recommended.
{% highlight shell %}
# lspci -s 03:00.1 -vv | grep LnkSta
LnkSta: Speed 8GT/s, Width x8, TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
{% endhighlight %}

Now that we know the NUMA node we can also check that if that is creating any impact in performance. In general a VM and its associated threads will run in a specific NUMA node and the memory pages and CPU in use should preferably be in the same NUMA node for highest performance. This is not always possible due to technical limitations. I.e. live-migration may have problems with CPU pinning/NUMA setups, which should be already fixed in the latest releases of Nova, however in some cases like NFV we would like to favor performance in detriment of the ability to livemigrate. CPU and NUMA pinning is a hard requirement in these scenarios. So the VNF (namely an instance) qemu-kvm threads will run in a pinned CPU, the memory pages should also match the NUMA node of the CPU and the NIC should also reside in it. Now there are multiple ways to consume this NIC from the guest, some workloads use DPDK or SR-IOV where one will dedicate specific sets of CPUs to the hypervisor and to the guests having as well as memory (normally hugepages) having in mind which NUMA node are they coming from, but I am not covering those here, and now a days there are even NUMA aware vSwitches, which will run an instnace and allocate a VIF to it in function of where the physical NIC that is in use by OpenvSwitch lives, thus avoiding cross-numa memory requests.

 - Check the NUMA architecture
{% highlight shell %}
#  numactl -H
available: 1 nodes (0)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23
node 0 size: 386761 MB
node 0 free: 91864 MB
node distances:
node   0 
  0:  10 
{% endhighlight %}

I only have here a single node, but the output will be similar with mulitple nodes. Important is to spot where the CPUs live, how much memory is in each node, the cross node distance (last table of the output) as in large architectures the distances varies from node to node and that is also a factor for the design.

 - Also the statistics can give an idea on how efficient is the usage at the moment
{% highlight shell %}
# numastat
                           node0
numa_hit              2176524897
numa_miss                      0
numa_foreign                   0
interleave_hit             23722
local_node            2176524897
other_node                     0
{% endhighlight %}

Here the numa_miss shows how many local misses occurred, and other_node count should show how many access from remote nodes have happened (each node has its own counter).

If one wanted to test how the performance would improve matching the NUMA node, one could use the commands __taskset__ and __migratepages__ on the instance PID to achieve that.

 - Check the current CPU affinity mask af qemu-kvm PID and restrict it to run on a specific set of CPUs (second CPU in this example) that correspond to a given NUMA node
 
{% highlight shell %}
# taskset -p 520733
pid 520733's current affinity mask: ffffff
# taskset -p  000002 520733
pid 520733's current affinity mask: ffffff
pid 520733's new affinity mask: 2
{% endhighlight %}

 - Similarly __migratepages__ command will move the memory pages from a set of source NUMA nodes to a given destination node
{% highlight shell %}
# numastat -c qemu-kvm
Per-node process memory usage (in MBs)
PID              Node 0 Node 1 Total
---------------  ------ ------ -----
2947 (qemu-kvm)    1802      0  1802
333578 (qemu-kvm   4904   3382  8286
---------------  ------ ------ -----
Total              6706   3382 10088

# migratepages 2947 0 1
# migratepages 333578 0 1
# numastat -c qemu-kvm
Per-node process memory usage (in MBs)
PID              Node 0 Node 1 Total
---------------  ------ ------ -----
2947 (qemu-kvm)       0   1802  1802
333578 (qemu-kvm      0   3522  3522
---------------  ------ ------ -----
Total                 0   5324  5324
{% endhighlight %}

Then again this is just for testing purposes and have a better grasp of what could improve the behavior observed.

What regards to the NIC configuration, it is useful to observe the statistics of the NIC at the moment of the issue, what counters increase from one reproduction to the next and build from there (we will cover this in step 4). Are the right offloadings that are needed supported by the NIC and enabled ? Are they working as expected ? For instance if using VxLAN/GENEVE tunneling or VLAN we want specific offloads to be enabled. Are we offloading flows ? Are there buffers like the RX ring properly sized ?


### 2. Reproduce it

One of the first questions I normally ask is if there is a clear set of steps that can reproduce the problem. If so, it will simplify the formulation of an hypothesis considerably for you and any other involved party. This is a little short cut which is part of the initial observation phase. If there is one, great. 

Determine which recurrence, if it happens in only one system, in many, time patterns, etc. Sometimes this is not possible as the problem only shows up sporadically and under unknown, apparently non-deterministic conditions. 

Some times it is just enough to use **iperf3** , **netperf** or similar tools to display the problem. Also often times is difficult to locate resources to reproduce a _production-like_ environment as it may use expensive equipment. Hence the use of virtual reproducers is a common case, unless the involved pieces of hardware are also part of the problem.

For example lets say we have two running instances inside two different compute nodes and we want to measure the throughput between them as we suspect there is an anomaly. They belong to the same tenant and virtual network. In the simplest test I would run a single TCP_STREAM test as follows to measure the baseline throughput between the compute nodes.

On the _server_ instance:
{% highlight shell %}
[root@hostb~]# iperf3 -s
{% endhighlight %}
  
On the _client_ instance:
{% highlight shell %}
[root@hosta~]# iperf3 -c 172.16.0.2
Connecting to host 172.16.0.2, port 5201
[  4] local 172.16.0.1 port 43488 connected to 172.16.0.2 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec   904 MBytes  7.58 Gbits/sec    0    820 KBytes       
[  4]   1.00-2.00   sec  1.08 GBytes  9.25 Gbits/sec    0    834 KBytes       
[  4]   2.00-3.00   sec  1.08 GBytes  9.24 Gbits/sec    0    840 KBytes       
[  4]   3.00-4.00   sec  1.08 GBytes  9.24 Gbits/sec    0    848 KBytes       
[  4]   4.00-5.00   sec  1.08 GBytes  9.24 Gbits/sec    0    863 KBytes       
[  4]   5.00-6.00   sec  1.08 GBytes  9.25 Gbits/sec    0    868 KBytes       
[  4]   6.00-7.00   sec  1.08 GBytes  9.24 Gbits/sec    0    871 KBytes       
[  4]   7.00-8.00   sec  1.08 GBytes  9.24 Gbits/sec    0    875 KBytes       
[  4]   8.00-9.00   sec  1.08 GBytes  9.24 Gbits/sec    0    877 KBytes       
[  4]   9.00-10.00  sec  1.08 GBytes  9.25 Gbits/sec    0    881 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  10.6 GBytes  9.08 Gbits/sec    0             sender
[  4]   0.00-10.00  sec  10.6 GBytes  9.07 Gbits/sec                  receiver

iperf Done.
{% endhighlight %}
 
Here we can observe throughput, numer of retries, the TCP Congestion Windown (Cwnd) size along the test and averages. This test can be further extended by using _iperf3_ options like: multiple parallel streams (-P<#>), port (-p), UDP traffic (-u), bandwidth/bitrate (-b), buffer size (-w), time length (-t), intervals (-i), reverse direction (-R), binding address (-B), and so on so forth.


### 3. Test initial conditions

The reproduction of the problem can depend on multiple factors like hardware architecture, NIC vendor/model, firmware version, OS version, kernel version, NIC driver version, physical network devices (switches, load balancers and routers), virtual devices that have to be crossed. Many times permutation or removal of any of these components is helpful to narrow down the problem to a particular component before delving deeper into the analysis. 

One of the first actions I normally try is to try to reproduce with the latest kernel available (downstream in case of _RHEL_), then the latest upstream kernel, also sometimes the latest kernel in __net-next__ tree of the linux kernel (which will become part of the next upstream linux kernel release) if there is any promising commit. 

For _RHEL_:

  * **Test latest downstream kernel.** You can check easily in CDN for the latest and update it if it is newer than the version in the system.
  
  * **Test lastest upstream stable kernel.** The easiest here is to leverage **elrepo** repository which provides an RPM for Centos and RHEL distros built from the **mainline** stable branch of Linux Kernel Archive and thus named **kernel-ml** to avoid conflict with RHEL stock kernels. In the following example I am installing the RPM for major version 7 and setting grub to boot from it (grub menu entry number 0) only once as we just want to test a reproducer and go back to the default kernel:
  
  {% highlight shell %}
  # rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
  # yum install https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
  # yum --enablerepo=elrepo-kernel install kernel-ml
  # grub2-reboot 0
  {% endhighlight %}

  * **Test linux-next kernel.** Similarly, when I expect some commit to solve my problem but it is not yet released upstream, I can test with the branches that are already accepted for the next stable release as follows. 
  
  
  {% highlight shell %}
  $ git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
  $ cd linux
  $ git remote add linux-next https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
  $ git fetch linux-next
  $ git fetch --tags linux-next
  {% endhighlight %}

  Now a specific **linux-next** tag can be checked out and built[^4]. Alternatively the **net-next** branch can also be used.

  {% highlight shell %}
  $ git remote add net git://git.kernel.org/pub/scm/linux/kernel/git/davem/net.git
  $ git fetch net
  {% endhighlight %}
 
  And just like that one can retry and discard tons of already fixed bugs or include new features that could improve the situation.

Similar approach could be adopted for NIC firmware, drivers or even testing with a different NIC hardware if resources permit. 

If the layout determined at step 2. is too complex. Cut down the devices involved and test if the issue can still be reproduced is also a good technique that will remove false suspects from the way. In example, if the issue happens with a bond device, does it still happen if we use directly one leg, or without bond at all ?  If the answer is yes, then we continue with the next piece to chop.

Actions like these have been the fastest way to identify existing bugs. Just by knowing it is not happening in a given kernel version, means that we only need to backport certain fix to a downstream kernel (in case of RHEL) or that the fix will be released soon upstream in case of using the **linux-next** kernel.

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


### 4. Performance Metrics

When dealing with a performance issue with a broad description of the problem, normally it is hard to know where to start digging. It always helps checking the output of performance and metrics monitoring tools, looking for stats like RX/TX packet counts and sizes in each interface involved, error counts in the NICs, and in general what counters overall are moving network wise or not.

In the simplest stats check, I would like to see the difference between the output of **ethtool -S <NIC>** before and after facing the issue or running the reproducer.
For instance, the following would tell that after the reproducer there was an increase of **no_buff_discards**, meaning that the NIC ran out of space in its RX ring buffer and had to discard the ingressing frames.

{% highlight shell %}
# ethtool -S p2p1  | egrep 'error|miss|drop|bad|crc|nop|discard' | egrep -v ': 0$'
     no_buff_discards: 10736

# ethtool -S p2p1  | egrep 'error|miss|drop|bad|crc|nop|discard' | egrep -v ': 0$'
     no_buff_discards: 25264
{% endhighlight %}

There are hundreds of command line tools to chose here but for the sake of simplicity I will focus on the readings of __ethtool__ and __sar__ (the later because is the most widespread accross systems). It is normal to find environments with proper performance tooling, like stacks combining __collectd__, __prometheus__, __grafana__, __ganglia__, or any __rrdtool__ derivative. One interesting tool to consider is __pcp__ (performance co-pilot [^5]). It does not really matter which tool to use as long as one can get the metrics that is after (for a comprehensive list of Linux command line tools check out the mind blowing work of Brendan Gregg [^6][^7]).


### 5. Tracing

There will be some situations where we _really_ want to know what is going on under the hood. For example, once we have identified that a bottleneck resides in certain userland or kernel thread, what happens next? Understanding what that task is doing is a fair next step which may reveal which part of the code is generating the problem. Another common example is that spurious drops are being observed but there is no clarity where or why are they being dropped.

From the set of tools that will help in this endeavour **perf** is probably the most powerful one and the one I will resort to the most also. I will show other examples like dynamic kernel tracing, using **ftrace**, either through scripting directly or through **trace-cmd** command line tool, or **dropwatch**.

Lets see some examples.
Say we see **ksoftirqd/X** kernel thread consuming 100% CPU while the issue is reproducing (that means that there are either too many softirqs being processed by the same CPU or each softirq is taking too much to be serviced, and in turn there could be packet drops), and I want to find out what is happening within that thread. Note that we will need the kernel-debuginfo package in order for perf to display useful information, otherwise the hex addresses are not translated to function names. Also note that we usually have to enable the channel that contains the debug packages for that (i.e. for RHEL7 is **rhel-7-server-debug-rpms**)

{% highlight shell %}
$ yum install -y perf kernel-debuginfo kernel-debuginfo-common
$ perf record --call-graph dwarf -p <PID> sleep 1
{% endhighlight %}

The previous command will create a perf.data file, with debugging data in DWARF format, in the working directory showing what user or kernel functions have been invoked from that thread during the recorded period (1 second).

This file can be visualized as follows.
{% highlight shell %}
$ perf report -i perf.data
{% endhighlight %}

The perf report can be navigated interactively in the command line (or it can also be non-interactive using --stdio flag), and it basically shows how much CPU is being consumed in each function in a break-down tree.

Following is an example output I captured from an OpenvSwitch process that was consuming 100% CPU in an OpenStack networker node. 
{% highlight shell %}
-   96.77%     0.00%  ovs-vswitchd    [kernel.kallsyms]   [k] system_call_fastpath           
   - system_call_fastpath                
      - 96.49% sys_sendmsg             
           __sys_sendmsg               
           ___sys_sendmsg               
           sock_sendmsg                  
         - netlink_sendmsg                   
            - 95.02% security_netlink_send    
                 selinux_netlink_send          
                 printk                                  
                 vprintk_default                        
               + vprintk_emit                                                                        
            + 1.42% netlink_unicast 
...
{% endhighlight %}

So in the previous example it can be observed that the process being checked is **ovs-vswitchd** and the funciton **security_netlink_send()** is consuming 95% of the CPU that the process is using. In this case it turned out to be a bug in openvswitch miscalculating the size of a netlink message.

The tool is extremely powerful. One can also see what a given CPU or group of CPUs is doing at a given point in time with **-C** flag.
{% highlight shell %}
$ perf top -C 0-2,3 -g
{% endhighlight %}

A similar result can be achieved using the **ftrace** facility directly by configuring **/sys/kernel/debug**. I have created a small script called [ftrace-2.sh](https://gist.githubusercontent.com/mauroseb/29d2e8664899aa2e1206e3a55b334aba/raw/5b462e5028bc17ca31e29559cd35b7157dc9bbb9/ftrace-2.sh
) to help me capture this data. This can come handy specially in **NFV** troubleshooting where a CPU has a single task of processing packets (running poll-mode driver thread) and any interruption or preemption from ot thread could bring packet drops. Hence if I see a packet drop, I want to know exactly what kernel or userspace threads are taking that CPU.

Another versatile tool to trace packets being dropped is **dropwatch**. This tool will basically report every time the **skb_free()** function is called and from which function. An **SKB** is the data structure that represents a packet in the kernel space, buffer which needs to be released after it is no longer useful when the packet dropped. There are many valid reasons why the **SKB** memory should be freed, however when there is an spurious drop it will also show up in the list.

{% highlight shell %}
# yum install -y dropwatch dropwatch-debuginfo
# dropwatch -l kas
Initalizing kallsyms db

dropwatch> start
Enabling monitoring...
Kernel monitoring activated.
Issue Ctrl-C to stop monitoring
1 drops at skb_queue_purge+18 (0xffffffffadc3c808)
21 drops at nf_hook_slow+f3 (0xffffffffadc93e33)
1 drops at __sme_end+11505a88 (0xffffffffc0705a88)
7 drops at nf_hook_slow+f3 (0xffffffffadc93e33)
21 drops at nf_hook_slow+f3 (0xffffffffadc93e33)
8 drops at nf_hook_slow+f3 (0xffffffffadc93e33)
1 drops at __sme_end+11505a88 (0xffffffffc0705a88)
21 drops at nf_hook_slow+f3 (0xffffffffadc93e33)
8 drops at nf_hook_slow+f3 (0xffffffffadc93e33)
21 drops at nf_hook_slow+f3 (0xffffffffadc93e33)
1 drops at nf_hook_slow+f3 (0xffffffffadc93e33)
7 drops at nf_hook_slow+f3 (0xffffffffadc93e33)
1 drops at __sme_end+11505a88 (0xffffffffc0705a88)
...
{% endhighlight %}

There is nothing fancy going on in the output above, though.

In regard to dynamic debugging, I will bring up one example. One customer of mine once was reporting that while live migrating a bulk amount of instances (+50) from one compute to another (operation that creates multiple parallel connections among the computes and demands high troughput), after 30 minutes or so their bond interface was getting into CHURNED state. I am not getting into details what that is, but you can consider that the syncrhonization between the switch and the compute node was severed. This issue turned to be a bug in **qed/qede** drivers which was at some point in time dropping LACPDUs (control frames for 802.3ad LACP bonds). To get to the bottom of it I just enabled dynamic debbugging in the bonding module by adding in the kernel command line (/etc/default/grub) **bonding.dyndbg="+p"**, this would allow to get debug from this module in **dmesg** output including things like when the LACPDU was being received. At some point was clear that the LACP PDU was not longer observed (because the driver was dropping it).

{% highlight shell %}
[88233.914226] bond1: Received LACPDU on port 1 slave p2p1
[88234.024144] bond1: Received LACPDU on port 2 slave p3p1
[88234.662755] Sent LACPDU on port 1
[88234.912929] bond1: Received LACPDU on port 1 slave p2p1
[88234.963108] Sent LACPDU on port 2
[88235.022547] bond1: Received LACPDU on port 2 slave p3p1
[88235.860830] Sent LACPDU on port 1
[88235.860838] Sent LACPDU on port 2
[88235.911258] bond1: Received LACPDU on port 1 slave p2p1
[88236.021118] bond1: Received LACPDU on port 2 slave p3p1
[88236.759651] Sent LACPDU on port 1
[88236.759658] Sent LACPDU on port 2
[88236.909813] bond1: Received LACPDU on port 1 slave p2p1
[88237.019850] bond1: Received LACPDU on port 2 slave p3p1                              <=== From this point on no more LACPDU observed
[88237.657462] Sent LACPDU on port 1
[88237.957743] Sent LACPDU on port 2
[88238.858547] Sent LACPDU on port 1
[88238.858555] Sent LACPDU on port 2
[88239.759359] Sent LACPDU on port 1
...
{% endhighlight %}


Lastly, I must mention **eBPF** (extended Barkley Packet Filter, originially named after BSD's BPF, however radically different) which is remarkably valuable and versatile kernel facility that was created for tracing purposes, but quickly became a swiss-army knife within the kernel that allows the user to create his own code and run it in a JIT compiler
in kernel land. Scary? Of course.


## OpenStack Network Architecture

OpenStack is a dynamic product and by design fully maleable to fit one's needs. Virtually every component is plugin based and can be interchangable by something else. In regard to networking it is no differen. The main project, Neutron can handle a large range of _ML2_ plugins (core component) from different opensource projects or from different vendors, and inside every ML2 plugin most of the networking services are also designed as pluggable as long as they respect well most of the defined Neutron API. With this said, this article mainly refers to the stock network layout for Neutron which is ML2 with OpenvSwitch (OvS) mechanism driver (recently OVN has been introduced but still uncommon in the field), as it has been the one where the problems covered below arised.

The architecture of ML2/OvS has been largely documented and described (just to list some references: [^8][^9][^10][^11] ) so the assumption is that the reader is already familiar with it. The following diagram illustrates to some extent what a typical OpenStack compute and networker node layout looks like in order to proceed with the cases' analysis.

<img src="/images/neutron_architecture.png" alt="Compute Network Layout" style="width:75%;"/>

The best practices dictate to use at least 6 VLANs (+1 optional for management) that would be normally trunked to at least two physical node NICs so they can be bonded together, however the installer is totally flexible and allows the user to be creative, use multiple independant NICs, flat networks, etc. Typical diagram is shown in the picture below.

<img src="/images/6-vlan-arch.png" alt="Node connectivity" style="width:75%;"/>



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
