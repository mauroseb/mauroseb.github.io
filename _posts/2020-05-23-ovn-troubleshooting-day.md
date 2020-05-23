---
layout: posts
title: "OVN troubleshooting day"
date: 2020-04-02
categories: [blog]
tags: [ openstack, ovn, networking ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---

## Intro

Hello again. While testing RHOSP 16 I ran into a connectivity issue to the outside world that turned out to be a documentation bug.
Follwing is the troubleshooting that took me to arrive to conclusions.

## Problem description

While testing the just released RHOSP 16.0.2 platform, I deployed as usual a testing tenant to verify all works as expected, when realized that only when floating IPs are associated to the instances they cannot reach further than the external gateway.
The logical layout looks somewhat like the following:
~~~
< Tenant Network > --- < Neutron vRouter > --- < External Physical Router > --- < Internet >
~~~

The first important fact is that when the instance does not have a floating IP associated (in other words, is using default tenant vrouter SNAT) to reach the outside all connectivity works well.
However as soon as a floating IP is associated we can reach up to the External Physical Router but not to the Internet.

The second remark is that default SDN in RHOSP16 is OpenVirtual Networking (OVN) which replaces the ever lasting Neutron/OVS ML2 plugin which has been there for several years.
The plugin can operate in two modes: _DVR_ or _NON DVR_. 

A brief explanation of the traffic flows that are shared and that are different between the two:

  1. All east-west traffic is always distributed no matter which mode we choose. That means that if an instance needs to talk to another instnace within openstack there is no need to reach the _gateway nodes_ (which in OVN speak is the analog to a Networker node in pre OVN setups).
  
  2. All north-south traffic (from OpenStack instances to external networks) from instances that _DO NOT_ have a specific floating IP associated (normally referred to as SNAT traffic), need to use the external IP of the vrouter, which is also shared among all intances in that tenant network. Given only one gateway node can hold this IP at a given point in time, this traffic is centralized no matter which mode is chosen.
  
  3. Finally the north-south traffic from instances that _DO_ have a floating IP associated with them differs in each mode. DVR mode will mean that the traffic will use the compute nodes as gateway nodes, thus distruting this kind of traffic. Mode means that this traffic is expected to traverse the gateway nodes in the controllers just like SNAT traffic.



Third point is that according to the documentation the _NON DVR_ mode should have been the default.

## Steps taken

### 1. Understanding OVN configuration and status

OVN creates the configuration in a logical plane (stored in northbound DB) that then is translated by _ovn-northd_  into what needs to be applied in the physical nodes
(stored in the southbound DB). But first lets see how this looks from neutron point of view. 
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

We can traverse the OvS OpenFlow tables one by one and try to follow which rules the packet will cross. But this requres some experience in the matter.

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
 priority=100,in_port="tapab12213e-3c" actions=load:0xe->NXM_NX_REG13[],load:0xc->NXM_NX_REG11[],load:0xd->NXM_NX_REG12[],load:0xa->OXM_OF_METADATA[],load:0x6->NXM_NX_REG14[],resubmit(,8)
 priority=100,in_port="tap531c4daa-10" actions=load:0xf->NXM_NX_REG13[],load:0xc->NXM_NX_REG11[],load:0xd->NXM_NX_REG12[],load:0xa->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],resubmit(,8)
 priority=100,in_port="tapc997f6f2-c8" actions=load:0x10->NXM_NX_REG13[],load:0xc->NXM_NX_REG11[],load:0xd->NXM_NX_REG12[],load:0xa->OXM_OF_METADATA[],load:0x9->NXM_NX_REG14[],resubmit(,8)
 priority=100,in_port="patch-br-int-to",dl_vlan=0 actions=strip_vlan,load:0x5->NXM_NX_REG13[],load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],load:0x1->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],resubmit(,8)
 priority=100,in_port="patch-br-int-to",vlan_tci=0x0000/0x1000 actions=load:0x5->NXM_NX_REG13[],load:0x2->NXM_NX_REG11[],load:0x3->NXM_NX_REG12[],load:0x1->OXM_OF_METADATA[],load:0x1->NXM_NX_REG14[],resubmit(,8)
{% endhighlight %}


### 3. Identify which node is the actual gateway node for this tenant vrouter

### 4. Resolution


