---
layout: posts
title: "OVN troubleshooting day"
date: 2020-05-23
categories: [blog]
tags: [ openstack, ovn, networking ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---

## Intro

Hello again. While testing RHOSP 16.0 I ran into a connectivity issue to the outside world that turned out to be a documentation bug.
Follwing is the troubleshooting that took me to arrive to conclusions.

## Problem description

While testing the just released RHOSP 16.0.2 platform, I deployed as usual a testing tenant to verify all works as expected, when realized that only when floating IPs are associated to the instances they cannot reach further than the external gateway.
The logical layout looks somewhat like the following:

{% highlight bash %}
< Tenant Network > --- < Neutron vRouter > --- < External Physical Router > --- < Internet >
{% endhighlight %}

The first important fact that concerns to the problem is that when the instance does not have a floating IP associated (in other words, is using default tenant vrouter SNAT) to reach the outside all connectivity works well.
However as soon as a floating IP is associated we can reach up to the External Physical Router but not to the Internet.

The second remark is that default SDN in RHOSP16 is OpenVirtual Networking (OVN) which replaces the ever lasting Neutron/OVS ML2 plugin which has been there for several years.
The plugin can operate in two modes: _DVR_ or _NON DVR_. 

Third point is that according to the documentation the _NON DVR_ mode should have been the default.

Following there is a brief explanation of the traffic flows that are shared and that are different between the two:

  1. All east-west traffic is always distributed no matter which mode we choose. That means that if an instance needs to talk to another instnace within openstack there is no need to reach the _gateway nodes_ (which in OVN speak is the analog to a Networker node in pre OVN setups).
  
  2. All north-south traffic (from OpenStack instances to external networks) from instances that _DO NOT_ have a specific floating IP associated (normally referred to as SNAT traffic), need to use the external IP of the vrouter, which is also shared among all intances in that tenant network. Given only one gateway node can hold this IP at a given point in time, this traffic is centralized no matter which mode is chosen.
  
  3. Finally the north-south traffic from instances that _DO_ have a floating IP associated with them differs in each mode. DVR mode will mean that the traffic will use the compute nodes as gateway nodes, thus distruting this kind of traffic. Mode means that this traffic is expected to traverse the gateway nodes in the controllers just like SNAT traffic.

In OVN speak every node that runs an __ovn-controller__ is called a chassis.


## Steps taken

### 1. Understanding OVN configuration and status

OVN creates the configuration in a logical plane (stored in northbound DB) that then is translated by _ovn-northd_  into the logical flows (stored in the southbound DB), which in turn _ovn-controller_ then uses as input to set OvS DB in the physical nodes. But first lets see how this looks from neutron point of view. 
I basically have two tenant routers (yes, one is for OpenShift) connected to respective tenant netowrks and both use as gateway _T01_external_ network.

{% highlight bash %}
(admin)(overcloud) [stack@undercloud-osp16 ~]$ openstack router list
+--------------------------------------+----------------------------------+--------+-------+----------------------------------+
| ID                                   | Name                             | Status | State | Project                          |
+--------------------------------------+----------------------------------+--------+-------+----------------------------------+
| 420fc376-5a71-4d19-8cc3-39160764a08c | ocp4-test3-9gvz2-external-router | ACTIVE | UP    | b2e7297100144d6a8feb856c0a8c18f5 |
| febf537f-d9bc-4382-8648-3c05e3e6b3d3 | T01_router                       | ACTIVE | UP    | b2e7297100144d6a8feb856c0a8c18f5 |
+--------------------------------------+----------------------------------+--------+-------+----------------------------------+
{% endhighlight %}

One thing to note at this point is that the routers are no longer highly available using _VRRP_, instead OVN uses _BFD_ protocol. Also this HA configuration is internal for OVN and not visible from OpenStack point of view.
Then from the northbound DB looks like this:

{% highlight bash %}
()[root@ice-ctl-01 /]# ovn-nbctl show
switch 18497b28-a55e-4fdc-b7c1-9a2b4cb138a1 (neutron-cb46658b-ee94-4eab-968d-b1f595ca01df) (aka T01_external)
    port provnet-cb46658b-ee94-4eab-968d-b1f595ca01df
        type: localnet
        addresses: ["unknown"]
    port a104502b-66eb-4aee-aeca-6dd38f6b021d
        type: router
        router-port: lrp-a104502b-66eb-4aee-aeca-6dd38f6b021d
    port 0d9badbf-07bd-4a2a-a3ca-5a4c15726c9b
        type: localport
        addresses: ["fa:16:3e:47:b7:31"]
    port 778f4f3b-62a4-4517-ad41-fac604b11c01
        type: router
        router-port: lrp-778f4f3b-62a4-4517-ad41-fac604b11c01
switch 3fba8179-e3a7-4b9c-a660-af042deb6169 (neutron-d64cd2ac-0307-4c68-8e60-a2f08a18278b) (aka ocp4-test3-9gvz2-openshift)
    port f649ae6c-eeb6-4ffe-b709-07bd446dd686 (aka ocp4-test3-9gvz2-api-port)
        type: virtual
        addresses: ["fa:16:3e:2c:14:d6 10.0.0.5"]
    port ab12213e-3c42-4e07-bef2-de30c542359c (aka ocp4-test3-9gvz2-bootstrap-port)
        addresses: ["fa:16:3e:09:3e:f1 10.0.3.222"]
    port 3ac5ceeb-fd2f-4490-a1a6-0df678b9e835 (aka ocp4-test3-9gvz2-ingress-port)
        type: virtual
        addresses: ["fa:16:3e:5c:d1:54 10.0.0.7"]
    port 14d50817-a4ec-42b4-b252-9f6e4c8b875c (aka ocp4-test3-9gvz2-master-port-2)
        addresses: ["fa:16:3e:13:6a:6c 10.0.0.126"]
    port 67801343-18f8-481f-b92b-5d149c0d57ce (aka ocp4-test3-9gvz2-master-port-1)
        addresses: ["fa:16:3e:ac:c3:0a 10.0.2.50"]
    port 0e300b43-503d-442a-81ed-8babc3dbcbd2
        type: localport
        addresses: ["fa:16:3e:28:1d:81 10.0.0.10"]
    port 73229f55-eb31-4536-9b96-912a1ebf1ef1
        type: router
        router-port: lrp-73229f55-eb31-4536-9b96-912a1ebf1ef1
    port 8f3320dc-af87-4180-9ef1-fc4157b777d8 (aka ocp4-test3-9gvz2-master-port-0)
        addresses: ["fa:16:3e:84:e0:47 10.0.2.7"]
    port c997f6f2-c83f-4c5a-b8a5-60c9bda1c2b5
        addresses: ["fa:16:3e:e5:1d:3c 10.0.0.105"]
switch e73748df-9ba7-4224-8792-b38ce5dc97e3 (neutron-5bccd77e-5eae-4213-a0f2-fdbb7f3e2d81) (aka T01_internal)
    port ef4306ae-a37d-40d9-92a8-900c84a764e1
        type: localport
        addresses: ["fa:16:3e:7c:62:4b 192.168.155.10 2001::f816:3eff:fe7c:624b"]
    port e6eb2a14-3842-4dc6-b0cb-d1b53b59faa0
        type: router
        router-port: lrp-e6eb2a14-3842-4dc6-b0cb-d1b53b59faa0
    port 8f4efe53-774b-4285-8c99-8fcf9e67c476
        type: router
        router-port: lrp-8f4efe53-774b-4285-8c99-8fcf9e67c476
    port e169efcb-d425-416b-b738-91fdb551cb80
        addresses: ["fa:16:3e:33:03:7a 192.168.155.14 2001::f816:3eff:fe33:37a"]
router 14e104b1-923c-463d-84a2-d69248e21dab (neutron-febf537f-d9bc-4382-8648-3c05e3e6b3d3) (aka T01_router)
    port lrp-8f4efe53-774b-4285-8c99-8fcf9e67c476
        mac: "fa:16:3e:6c:ff:d4"
        networks: ["2001::1/64"]
    port lrp-e6eb2a14-3842-4dc6-b0cb-d1b53b59faa0
        mac: "fa:16:3e:b2:d4:43"
        networks: ["192.168.155.1/24"]
    port lrp-778f4f3b-62a4-4517-ad41-fac604b11c01
        mac: "fa:16:3e:61:e9:61"
        networks: ["192.168.122.124/24"]
        gateway chassis: [eca06807-53e8-4923-90a5-481146358704 1b9c5ca7-e479-4232-89e6-8956fc086e94 365b3cf7-14c8-43e1-a198-90209c1d77f9 3fc4fc28-1888-4287-bd78-62840b97d2dd 190c75d4-bb66-4367-9442-69694c5cbf73]
    nat 36153ea8-de25-462d-9447-48069aa6330f
        external ip: "192.168.122.124"
        logical ip: "192.168.155.0/24"
        type: "snat"
    nat cb60b8d0-f6f3-4c74-9ff7-9c1819d886f1
        external ip: "192.168.122.111"
        logical ip: "192.168.155.14"
        type: "dnat_and_snat"
router 081b5bb7-cda0-4d43-a3c6-09e8012f71a1 (neutron-420fc376-5a71-4d19-8cc3-39160764a08c) (aka ocp4-test3-9gvz2-external-router)
    port lrp-a104502b-66eb-4aee-aeca-6dd38f6b021d
        mac: "fa:16:3e:9b:b7:06"
        networks: ["192.168.122.119/24"]
        gateway chassis: [7f4067b7-b0e9-43e7-885b-10a69b1aa0d8 190c75d4-bb66-4367-9442-69694c5cbf73 3fc4fc28-1888-4287-bd78-62840b97d2dd 365b3cf7-14c8-43e1-a198-90209c1d77f9 1b9c5ca7-e479-4232-89e6-8956fc086e94]
    port lrp-73229f55-eb31-4536-9b96-912a1ebf1ef1
        mac: "fa:16:3e:44:22:9d"
        networks: ["10.0.0.1/16"]
    nat 255a82be-038d-464f-9e56-f7d17e255805
        external ip: "192.168.122.128"
        logical ip: "10.0.3.222"
        type: "dnat_and_snat"
    nat bc083570-cd91-4b49-a55b-f9f63aca68da
        external ip: "192.168.122.121"
        logical ip: "10.0.0.105"
        type: "dnat_and_snat"
    nat cba09a09-7c2b-44b9-bc07-5cfc7c852023
        external ip: "192.168.122.119"
        logical ip: "10.0.0.0/16"
        type: "snat"
    nat e8a6cb3b-03f0-40ad-b340-2ef747f63dac
        external ip: "192.168.122.112"
        logical ip: "10.0.0.5"
        type: "dnat_and_snat"

{% endhighlight %}

Coversely the southbound DB looks as follows:

{% highlight bash %}
()[root@ice-ctl-01 /]# ovn-sbctl show
Chassis "1b9c5ca7-e479-4232-89e6-8956fc086e94"
    hostname: "ice-com-03.lab.local"
    Encap geneve
        ip: "172.16.100.105"
        options: {csum="true"}
Chassis "3fc4fc28-1888-4287-bd78-62840b97d2dd"
    hostname: "ice-com-02.lab.local"
    Encap geneve
        ip: "172.16.100.104"
        options: {csum="true"}
    Port_Binding "ab12213e-3c42-4e07-bef2-de30c542359c"
    Port_Binding "e169efcb-d425-416b-b738-91fdb551cb80"
Chassis "365b3cf7-14c8-43e1-a198-90209c1d77f9"
    hostname: "ice-ctl-02.lab.local"
    Encap geneve
        ip: "172.16.100.101"
        options: {csum="true"}
Chassis "eca06807-53e8-4923-90a5-481146358704"
    hostname: "ice-ctl-01.lab.local"
    Encap geneve
        ip: "172.16.100.100"
        options: {csum="true"}
    Port_Binding "cr-lrp-778f4f3b-62a4-4517-ad41-fac604b11c01"
Chassis "190c75d4-bb66-4367-9442-69694c5cbf73"
    hostname: "ice-ctl-03.lab.local"
    Encap geneve
        ip: "172.16.100.102"
        options: {csum="true"}
Chassis "7f4067b7-b0e9-43e7-885b-10a69b1aa0d8"
    hostname: "ice-com-01.lab.local"
    Encap geneve
        ip: "172.16.100.103"
        options: {csum="true"}
    Port_Binding "c997f6f2-c83f-4c5a-b8a5-60c9bda1c2b5"
    Port_Binding "cr-lrp-a104502b-66eb-4aee-aeca-6dd38f6b021d"
{% endhighlight %}

**NOTE:** This commands have to be run in the container for ovn-db.

### 2. Understanding the flow logic

One can traverse the OvS OpenFlow tables and try to follow which rules the packet will cross. But this requires some experience in the matter. The starting point is to determine the instance compute node and TAP interface that corresponds to the instance. This can be easily captured by checking the instance libvirt XML file. In my case it is __tapab12213e-3c__. With this infromation, check the br-int table 0 where all starts:

{% highlight bash %}
[root@ice-com-02 ~]# ovs-ofctl dump-flows br-int table=0 --no-stats
 priority=180,vlan_tci=0x0000/0x1000 actions=conjunction(100,2/2)
 priority=180,conj_id=100,in_port="patch-br-int-to",vlan_tci=0x0000/0x1000 actions=load:0x5->NXM_NX_REG13[],load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],load:0x1->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],mod_dl_src:fa:16:3e:61:e9:61,resubmit(,8)
 priority=100,in_port="ovn-1b9c5c-0" actions=move:NXM_NX_TUN_ID[0..23]->OXM_OF_METADATA[0..23],move:NXM_NX_TUN_METADATA0[16..30]->NXM_NX_REG14[0..14],move:NXM_NX_TUN_METADATA0[0..15]->NXM_NX_REG15[0..15],resubmit(,33)
 priority=100,in_port="ovn-7f4067-0" actions=move:NXM_NX_TUN_ID[0..23]->OXM_OF_METADATA[0..23],move:NXM_NX_TUN_METADATA0[16..30]->NXM_NX_REG14[0..14],move:NXM_NX_TUN_METADATA0[0..15]->NXM_NX_REG15[0..15],resubmit(,33)
 priority=100,in_port="ovn-190c75-0" actions=move:NXM_NX_TUN_ID[0..23]->OXM_OF_METADATA[0..23],move:NXM_NX_TUN_METADATA0[16..30]->NXM_NX_REG14[0..14],move:NXM_NX_TUN_METADATA0[0..15]->NXM_NX_REG15[0..15],resubmit(,33)
 priority=100,in_port="ovn-eca068-0" actions=move:NXM_NX_TUN_ID[0..23]->OXM_OF_METADATA[0..23],move:NXM_NX_TUN_METADATA0[16..30]->NXM_NX_REG14[0..14],move:NXM_NX_TUN_METADATA0[0..15]->NXM_NX_REG15[0..15],resubmit(,33)
 priority=100,in_port="ovn-365b3c-0" actions=move:NXM_NX_TUN_ID[0..23]->OXM_OF_METADATA[0..23],move:NXM_NX_TUN_METADATA0[16..30]->NXM_NX_REG14[0..14],move:NXM_NX_TUN_METADATA0[0..15]->NXM_NX_REG15[0..15],resubmit(,33)
 priority=100,in_port="tape169efcb-d4" actions=load:0x8->NXM_NX_REG13[],load:0x7->NXM_NX_REG11[],load:0x6->NXM_NX_REG12[],load:0x2->OXM_OF_METADATA[],load:0x4->NXM_NX_REG14[],resubmit(,8)
 priority=100,in_port="tap4f1711d6-70" actions=load:0x9->NXM_NX_REG13[],load:0x7->NXM_NX_REG11[],load:0x6->NXM_NX_REG12[],load:0x2->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],resubmit(,8)
 __priority=100,in_port="tapab12213e-3c" actions=load:0xe->NXM_NX_REG13[],load:0xc->NXM_NX_REG11[],load:0xd->NXM_NX_REG12[],load:0xa->OXM_OF_METADATA[],load:0x6->NXM_NX_REG14[],resubmit(,8)__
 priority=100,in_port="tap531c4daa-10" actions=load:0xf->NXM_NX_REG13[],load:0xc->NXM_NX_REG11[],load:0xd->NXM_NX_REG12[],load:0xa->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],resubmit(,8)
 priority=100,in_port="tapc997f6f2-c8" actions=load:0x10->NXM_NX_REG13[],load:0xc->NXM_NX_REG11[],load:0xd->NXM_NX_REG12[],load:0xa->OXM_OF_METADATA[],load:0x9->NXM_NX_REG14[],resubmit(,8)
 priority=100,in_port="patch-br-int-to",dl_vlan=0 actions=strip_vlan,load:0x5->NXM_NX_REG13[],load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],load:0x1->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],resubmit(,8)
 priority=100,in_port="patch-br-int-to",vlan_tci=0x0000/0x1000 actions=load:0x5->NXM_NX_REG13[],load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],load:0x1->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],resubmit(,8)
{% endhighlight %}

The rules in this table depend on the ingress port, the name for the instance's port matches the TAP device name __in_port="tapab12213e-3c"__ (there is also a numerical value to it - 15 in this case).
The actions taken are to load some values in some registers (can be ignored for now) and resubmitted to table 8.

Checking table 8:

{% highlight bash %}
[root@ice-com-02 ~]# ovs-ofctl dump-flows br-int table=8 --no-stats
 cookie=0x45c9b514, table=8, priority=100,metadata=0x3,vlan_tci=0x1000/0x1000 actions=drop
 cookie=0xa0dfbc17, table=8, priority=100,metadata=0x1,vlan_tci=0x1000/0x1000 actions=drop
 cookie=0x741dbebb, table=8, priority=100,metadata=0x2,vlan_tci=0x1000/0x1000 actions=drop
 cookie=0x870d8275, table=8, priority=100,metadata=0xb,vlan_tci=0x1000/0x1000 actions=drop
 cookie=0x3d7bf566, table=8, priority=100,metadata=0xa,vlan_tci=0x1000/0x1000 actions=drop
 cookie=0x45c9b514, table=8, priority=100,metadata=0x3,dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
 cookie=0xda98272a, table=8, priority=100,metadata=0x1,dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
 cookie=0xf546d6b5, table=8, priority=100,metadata=0x2,dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
 cookie=0x870d8275, table=8, priority=100,metadata=0xb,dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
 cookie=0x8f4868ca, table=8, priority=100,metadata=0xa,dl_src=01:00:00:00:00:00/01:00:00:00:00:00 actions=drop
 cookie=0x2c2341, table=8, priority=50,reg14=0x1,metadata=0x1 actions=resubmit(,9)
 cookie=0xb987da46, table=8, priority=50,reg14=0x3,metadata=0x1 actions=resubmit(,9)
 cookie=0x1ee84a52, table=8, priority=50,reg14=0x2,metadata=0x1 actions=resubmit(,9)
 cookie=0xcd2cfb18, table=8, priority=50,reg14=0x1,metadata=0x2 actions=resubmit(,9)
 cookie=0xa929ae55, table=8, priority=50,reg14=0x2,metadata=0x2 actions=resubmit(,9)
 cookie=0x7311ab3a, table=8, priority=50,reg14=0x3,metadata=0x2 actions=resubmit(,9)
 cookie=0x1250dcb1, table=8, priority=50,reg14=0x4,metadata=0x1 actions=resubmit(,9)
 cookie=0x7eba3fa9, table=8, priority=50,reg14=0x1,metadata=0xa actions=resubmit(,9)
 cookie=0x86e5f0f0, table=8, priority=50,reg14=0x2,metadata=0xa actions=resubmit(,9)
 cookie=0xd7380714, table=8, priority=50,reg14=0x1,metadata=0x3,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,9)
 cookie=0xc8f90bc8, table=8, priority=50,reg14=0x3,metadata=0x3,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,9)
 cookie=0x65841c3f, table=8, priority=50,reg14=0x4,metadata=0x3,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,9)
 cookie=0xfc955284, table=8, priority=50,reg14=0x1,metadata=0xb,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,9)
 cookie=0x8ad5172e, table=8, priority=50,reg14=0x3,metadata=0xb,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,9)
 cookie=0x442a9af3, table=8, priority=50,reg14=0x3,metadata=0x3,dl_dst=fa:16:3e:b2:d4:43 actions=resubmit(,9)
 cookie=0xb9db9c7b, table=8, priority=50,reg14=0x4,metadata=0x3,dl_dst=fa:16:3e:6c:ff:d4 actions=resubmit(,9)
 cookie=0xb2d4772b, table=8, priority=50,reg14=0x1,metadata=0x3,dl_dst=fa:16:3e:94:4a:9b actions=resubmit(,9)
 cookie=0xcfe68a77, table=8, priority=50,reg14=0x3,metadata=0xb,dl_dst=fa:16:3e:44:22:9d actions=resubmit(,9)
 cookie=0xd81159b7, table=8, priority=50,reg14=0x1,metadata=0xb,dl_dst=fa:16:3e:c2:be:99 actions=resubmit(,9)
 cookie=0x942a2974, table=8, priority=50,reg14=0x1,metadata=0xb,dl_dst=fa:16:3e:55:f0:8a actions=resubmit(,9)
 cookie=0xaed209e2, table=8, priority=50,reg14=0x4,metadata=0x2,dl_src=fa:16:3e:33:03:7a actions=resubmit(,9)
 **cookie=0x51155ce9, table=8, priority=50,reg14=0x6,metadata=0xa,dl_src=fa:16:3e:09:3e:f1 actions=resubmit(,9)**
 cookie=0x3721ef07, table=8, priority=50,reg14=0x9,metadata=0xa,dl_src=fa:16:3e:e5:1d:3c actions=resubmit(,9)
{% endhighlight %}

This table basically drops some unwanted traffic and resumbit traffic to table 9 if it matches some characteristics. In my case that the __dl_dst__ (datalink destination) is the MAC of the tenant router in question (had to check manually).

Then check table 9... an so on so forth. You see the picture and it's not beautiful. That is why OvS tracing comes handy here. We can check what OpenFlow rules the packet will cross by providing some input similar to what a problematic packet may look like (source and dest MAC / IPs , for instance, and the ingress port):

{% highlight bash %}
[root@ice-com-02 ~]# ovs-appctl ofproto/trace br-int in_port=15,icmp,dl_src=fa:16:3e:09:3e:f1,nw_src=10.0.3.222,nw_dst=192.168.122.119
Flow: icmp,in_port=15,vlan_tci=0x0000,dl_src=fa:16:3e:09:3e:f1,dl_dst=00:00:00:00:00:00,nw_src=10.0.3.222,nw_dst=192.168.122.119,nw_tos=0,nw_ecn=0,nw_ttl=0,icmp_type=0,icmp_code=0

bridge("br-int")
----------------
 0. in_port=15, priority 100
    set_field:0xe->reg13
    set_field:0xc->reg11
    set_field:0xd->reg12
    set_field:0xa->metadata
    set_field:0x6->reg14
    resubmit(,8)
 8. reg14=0x6,metadata=0xa,dl_src=fa:16:3e:09:3e:f1, priority 50, cookie 0x51155ce9
    resubmit(,9)
 9. ip,reg14=0x6,metadata=0xa,dl_src=fa:16:3e:09:3e:f1,nw_src=10.0.3.222, priority 90, cookie 0xb3a17d75
    resubmit(,10)
10. metadata=0xa, priority 0, cookie 0x556a736d
    resubmit(,11)
11. ip,metadata=0xa, priority 100, cookie 0x2b0fd59f
    load:0x1->NXM_NX_XXREG0[96]
    resubmit(,12)
12. metadata=0xa, priority 0, cookie 0xdeb132b7
    resubmit(,13)
13. ip,reg0=0x1/0x1,metadata=0xa, priority 100, cookie 0xc54c1ac2
    ct(table=14,zone=NXM_NX_REG13[0..15])
    drop
     -> A clone of the packet is forked to recirculate. The forked pipeline will be resumed at table 14.
     -> Sets the packet to an untracked state, and clears all the conntrack fields.

Final flow: icmp,reg0=0x1,reg11=0xc,reg12=0xd,reg13=0xe,reg14=0x6,metadata=0xa,in_port=15,vlan_tci=0x0000,dl_src=fa:16:3e:09:3e:f1,dl_dst=00:00:00:00:00:00,nw_src=10.0.3.222,nw_dst=192.168.122.119,nw_tos=0,nw_ecn=0,nw_ttl=0,icmp_type=0,icmp_code=0
Megaflow: recirc_id=0,eth,icmp,in_port=15,vlan_tci=0x0000/0x1000,dl_src=fa:16:3e:09:3e:f1,dl_dst=00:00:00:00:00:00,nw_src=10.0.3.222,nw_frag=no,icmp_type=0x0/0xff
Datapath actions: ct(zone=14),recirc(0x8ff5)

===============================================================================
recirc(0x8ff5) - resume conntrack with default ct_state=trk|new (use --ct-next to customize)
===============================================================================

Flow: recirc_id=0x8ff5,ct_state=new|trk,ct_zone=14,eth,icmp,reg0=0x1,reg11=0xc,reg12=0xd,reg13=0xe,reg14=0x6,metadata=0xa,in_port=15,vlan_tci=0x0000,dl_src=fa:16:3e:09:3e:f1,dl_dst=00:00:00:00:00:00,nw_src=10.0.3.222,nw_dst=192.168.122.119,nw_tos=0,nw_ecn=0,nw_ttl=0,icmp_type=0,icmp_code=0

bridge("br-int")
----------------
    thaw
        Resuming from table 14
14. ct_state=+new-est+trk,ip,reg14=0x6,metadata=0xa, priority 2002, cookie 0x5e6b2ce6
    load:0x1->NXM_NX_XXREG0[97]
    resubmit(,15)
15. metadata=0xa, priority 0, cookie 0x22ca4285
    resubmit(,16)
16. metadata=0xa, priority 0, cookie 0xc01f3cd9
    resubmit(,17)
17. metadata=0xa, priority 0, cookie 0x576ed5a2
    resubmit(,18)
18. ip,reg0=0x2/0x2,metadata=0xa, priority 100, cookie 0xcc7a350d
    ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0->NXM_NX_CT_LABEL[0]))
    load:0->NXM_NX_CT_LABEL[0]
     -> Sets the packet to an untracked state, and clears all the conntrack fields.
    resubmit(,19)
19. metadata=0xa, priority 0, cookie 0x7159fca6
    resubmit(,20)
20. metadata=0xa, priority 0, cookie 0x3c57b2c6
    resubmit(,21)
21. metadata=0xa, priority 0, cookie 0x749944d2
    resubmit(,22)
22. metadata=0xa, priority 0, cookie 0xcfea7be8
    resubmit(,23)
23. metadata=0xa, priority 0, cookie 0xdbbe17b0
    resubmit(,24)
24. metadata=0xa, priority 0, cookie 0xfc61b723
    resubmit(,25)
25. No match.
    drop

Final flow: recirc_id=0x8ff5,eth,icmp,reg0=0x3,reg11=0xc,reg12=0xd,reg13=0xe,reg14=0x6,metadata=0xa,in_port=15,vlan_tci=0x0000,dl_src=fa:16:3e:09:3e:f1,dl_dst=00:00:00:00:00:00,nw_src=10.0.3.222,nw_dst=192.168.122.119,nw_tos=0,nw_ecn=0,nw_ttl=0,icmp_type=0,icmp_code=0
Megaflow: recirc_id=0x8ff5,ct_state=+new-est-rel-rpl-inv+trk,ct_label=0/0x1,eth,ip,in_port=15,dl_src=fa:16:3e:09:3e:f1,dl_dst=00:00:00:00:00:00,nw_dst=192.168.0.0/17,nw_frag=no
Datapath actions: ct(commit,zone=14,label=0/0x1)
{% endhighlight %}

Therefore after crossing multiple tables the packet is finally dropped at table 25. The purpose of most tables is documented for the curious mind. Given that the packet is not going out of the compute node is not even reaching the gateway nodes. Remember floating IPs should still be created there in _non DVR_ mode. Next step is to determine which should be the gateway node for this network to inspect the other end.

### 3. Resolution

 - Find the port from the router in the external network.

{% highlight bash %}
(admin)(overcloud) [stack@undercloud-osp16 ~]$ openstack port list --device-id 420fc376-5a71-4d19-8cc3-39160764a08c --device-owner 'network:router_gateway'
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                             | Status |
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------+--------+
| a104502b-66eb-4aee-aeca-6dd38f6b021d |      | fa:16:3e:9b:b7:06 | ip_address='192.168.122.119', subnet_id='706ab6cc-c98d-44a4-9fa6-f581c8311cc7' | ACTIVE |
+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------+--------+
{% endhighlight %}

 - Once the logical router port id is known get OVN list of gateway nodes for this port  (prepending lrp- to the port ID above):
 
{% highlight bash %}
()[root@ice-ctl-01 /]# ovn-nbctl lrp-get-gateway-chassis lrp-a104502b-66eb-4aee-aeca-6dd38f6b021d
lrp-a104502b-66eb-4aee-aeca-6dd38f6b021d_7f4067b7-b0e9-43e7-885b-10a69b1aa0d8     5
lrp-a104502b-66eb-4aee-aeca-6dd38f6b021d_190c75d4-bb66-4367-9442-69694c5cbf73     4
lrp-a104502b-66eb-4aee-aeca-6dd38f6b021d_3fc4fc28-1888-4287-bd78-62840b97d2dd     3
lrp-a104502b-66eb-4aee-aeca-6dd38f6b021d_365b3cf7-14c8-43e1-a198-90209c1d77f9     2
lrp-a104502b-66eb-4aee-aeca-6dd38f6b021d_1b9c5ca7-e479-4232-89e6-8956fc086e94     1
{% endhighlight %}

This result is hinting an issue already, as with 3 controllers I would only expect 3 chassis to be listed there.

To clarify, each row corresponds to a chassis that can manage the logical port, the highest priority (number at the end) is the one in control of the logical port at the moment. To translate to a node name we need to use the last UUID of the long string with highest priority. In this example _7f4067b7-b0e9-43e7-885b-10a69b1aa0d8_.

{% highlight bash %}
()[root@ice-ctl-01 /]# ovn-sbctl get Chassis 7f4067b7-b0e9-43e7-885b-10a69b1aa0d8 hostname
"ice-com-01.lab.local"
{% endhighlight %}

And the owner is a compute node! This is unexpected since this deployment is not DVR enabled (__NeutronEnabledDVR__ default is documented as false in RHOSP16.0), and the port should be managed by a controller node. Since a compute is listed there it makes it look like DVR is indeed enabled but the compute nodes have no external network connectivity resulting in a communication error for floating IP traffic.

I can further probe that by checking Neutron configuration:

{% highlight bash %}
# grep -r distributed_floating /var/lib/config-data/puppet-generated/
/var/lib/config-data/puppet-generated/neutron/etc/neutron/plugins/ml2/ml2_conf.ini:enable_distributed_floating_ip=True
{% endhighlight %}

After further analysis it turned out to be a documentation bug which is now filed and corrected [^1].
So after redeploying the environment explicitly setting in TripleO templates __NeutronEnableDVR: false__ the behavior is correct and works as expected.

### References

[^1] https://bugzilla.redhat.com/show_bug.cgi?id=1839139

