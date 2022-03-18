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

With the release of OpenShift 4.10 version the MetalLB operator working in BGP mode becomes generally available.
The purpose of this post is to provide basic information on its design and purpose, to cover the configuration and usage, and finally share my findings about it.

<!-- more -->

### Brief on MetalLB, BGP and BFD

Some concepts need to be conveyed in order to follow the next section where I will go through the configuration procedure.

First and foremost I need to address what is MetalLB ?

It is an OpenSource project that provides the ability to create ```LoadBalancer``` type of kubernetes Services on top of a baremetal OpenShift/Kubernetes cluster and in turn create the same user experience that one would get on a public cloud provider like in AWS, GCP or Azure.[^1]

A service of type ```LoadBalancer``` in kubernetes will automatically request the cloud provider to provision a load balancer that will direct traffic to the right cluster node and port tuples, and subsequently assign an externally accessible IP to it. Then information like the assigned IP address will be updated in the status section of the new service resource.[^2] In an on prem environment Kubernetes will create a ClusterIP service and assign a node port and wait for the administrator to create the load balancer side configuration.

In order to achieve a similar behavior to public cloud providers, MetalLB has to handle two main tasks: _address allocation_ which means managing the address pools that the new ```LoadBalancer``` services will use, and secondly the _external announcement_ of these services' IPs, so that external entities to the cluster can learn how to reach them.

In regard to address allocation MetalLB has to be told which IPs or IP ranges it can assign when a new service is created and this is accomplished through the ```AddressPool``` CRD.

As to external announcement of the IP, MetalLB offers two alternatives:

   1. **Layer 2 mode.** This option has been generally available for a while now and basically relies on ARP (IPv4) and NDP (IPv6) to publish which cluster node is the owner of the service IP. Even though the method helps and it is acceptable for many use cases, it has some drawbacks. Namely, the node owning the IP will become a bottleneck, and the time to failover in case the node is gone may be slow. In this article I do not intend to focus in this mode as the BGP alternative is deemed superior.

   2. **BGP mode.** This method is based of course in the __two-napkin__[^3] protocol (BGP), which allows the cluster administrator to establish peering sessions between BGP speaker pods placed in selected nodes from the cluster to an external BGP speaker, like the upstream router. Therefore the incoming requests directed to the service external IP that arrive to the router will be properly forwarded to the nodes. This method allows appropiate load balancing of the ingress traffic if the external router is doing multipath of the incoming requests.


The remaining part of this article I am going to focus on the latter, BGP mode.

When MetalLB works in BGP mode, it makes use of FRR[^4], an opensource project that started as an spin off of Quagga, and provides GNU/Linux based fully featured IP routing capabilities. It supports all sort of routing protocols, and among them the one MetalLB needs: BGP. 

BGP or Border Gateway Protocol is the Internet and industry de facto protocol for routing, which you probably heard of. In brief, BGPv4 was defined under RFC1654[^5] as an exterior gateway path-vector routing protocol, with the goal of conveying network reachibility between autonomous systems. The autonomous system can be a flexible term but in its conception refers to a set of routers under a single technical administration, ideally with a common internal gateway protocol in use, common metrics, etc. Each AS will be identified by an ASN or autonomous system number that will be typically a unique 16-bit number (or 32 bits with RFC 4893).

BGP minimum allowed time to keep a failed session (hold time) is 3 seconds, which may be unacceptable amount of time for a service to be unavailable. Therefore, in addition to the standard BGP speaking capabilities, MetalLB (and FRR) also supports to configure Bi-directional Forwarding Detection (or BFD) protocol. BFD adds further resiliency to the solution providing faster failure detection. It allows two routers to detect faults in the bidirectional path among them at subsencond intervals. BFD works at layer 3, and basically offers a lightweight liveness detection protocol (aka "hello" protocol) that is independant of the routing protocol itself, it will only limit itself to notify the routing protocol about the failure and not to take any corrective action.[^6]


### Environment

The network where the OpenShift cluster will run on needs an adjacent router to speak also BGP and BFD, moreover the router will do ECMP (Equal Cost Multipath) in order to distribute the incoming requests uniformly across multiple worker nodes. So if a service is advertised from multiple worker nodes, each should get an even amount of requests.

For this purpose I am going to use an isolated instance of FRR running containerized within an external system in order to emulate that router.

The following diagram describes the test scenario in more detail.

<img src="/images/LAB_METALLB_BGP.png" alt="OCP LAB METALLB BGP MODE" style="width:75%;"/>


For the purpose of testing this feature I have an already installed cluster of OpenShift 4.10.3 using baremetal UPI deployment method, and also using Openshift SDN as CNI plugin with default configuration.

{% highlight bash %}

$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.10.3    True        False         23h     Cluster version is 4.10.3

$ oc get nodes
NAME                          STATUS   ROLES    AGE   VERSION
ice-ocp4-master-0.lab.local   Ready    master   15h   v1.23.3+e419edf
ice-ocp4-master-1.lab.local   Ready    master   15h   v1.23.3+e419edf
ice-ocp4-master-2.lab.local   Ready    master   14h   v1.23.3+e419edf
ice-ocp4-worker-0.lab.local   Ready    worker   28m   v1.23.3+e419edf
ice-ocp4-worker-1.lab.local   Ready    worker   28m   v1.23.3+e419edf

$ oc get network cluster -o yaml
apiVersion: config.openshift.io/v1
kind: Network
metadata:
  creationTimestamp: "2022-03-16T18:51:34Z"
  generation: 2
  name: cluster
  resourceVersion: "5174"
  uid: b4f13c36-bef8-44fb-9cec-780b6c565eda
spec:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  externalIP:
    policy: {}
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
status:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  clusterNetworkMTU: 1450
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16

$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.10.3    True        False         False      48m     
baremetal                                  4.10.3    True        False         False      15h     
cloud-controller-manager                   4.10.3    True        False         False      16h     
cloud-credential                           4.10.3    True        False         False      16h     
cluster-autoscaler                         4.10.3    True        False         False      15h     
config-operator                            4.10.3    True        False         False      15h     
console                                    4.10.3    True        False         False      40m     
csi-snapshot-controller                    4.10.3    True        False         False      15h     
dns                                        4.10.3    True        False         False      15h     
etcd                                       4.10.3    True        False         False      14h     
image-registry                             4.10.3    True        False         False      14h     
ingress                                    4.10.3    True        False         False      53m     
insights                                   4.10.3    True        False         False      15h     
kube-apiserver                             4.10.3    True        False         False      14h     
kube-controller-manager                    4.10.3    True        False         False      15h     
kube-scheduler                             4.10.3    True        False         False      15h     
kube-storage-version-migrator              4.10.3    True        False         False      15h     
machine-api                                4.10.3    True        False         False      15h     
machine-approver                           4.10.3    True        False         False      15h     
machine-config                             4.10.3    True        False         False      14h     
marketplace                                4.10.3    True        False         False      15h     
monitoring                                 4.10.3    True        False         False      20m     
network                                    4.10.3    True        False         False      15h     
node-tuning                                4.10.3    True        False         False      14h     
openshift-apiserver                        4.10.3    True        False         False      14h     
openshift-controller-manager               4.10.3    True        False         False      15h     
openshift-samples                          4.10.3    True        False         False      14h     
operator-lifecycle-manager                 4.10.3    True        False         False      15h     
operator-lifecycle-manager-catalog         4.10.3    True        False         False      15h     
operator-lifecycle-manager-packageserver   4.10.3    True        False         False      14h     
service-ca                                 4.10.3    True        False         False      15h     
storage                                    4.10.3    True        False         False      15h     

{% endhighlight %}



### Pre-Requisites

For MetalLB in BGP mode to work that needs to be done is as follows:

 * First and foremost a **baremetal cluster** in place. It can be deployed either with IPI or UPI.
 * The external router as described in the Environment section.
 * The external router has to live in the same network segment as the openshift nodes (single hop network topology).
 * The addresses that will be assigned to the AddressPool custom resource should be reserved and belong to the external network (as opposite to the machineNetwrok).
 * There should be open communication between the external router and the cluster nodes on port 179/TCP (BGP), 3784/UDP and 3785/UDP (BFD). 

### Procedure

The procedure consists of deploying the external router, then deploy and configure MetalLB operator and test the resulting scenario.

#### External Router

In order to emulate the external router I will make use of FRR itself, by deploying an FRR container on a external system in the same collision domain as the OpenShift worker nodes. The container will use a configuration directory holding three: frr.conf (main FRR config file), daemons (which daemons will FRR use) and vtysh.conf (empty file).

In the system in question, create the directory that will hold the FRR configuration (in this example $HOME/frr), and copy the following files into it.

{% highlight console %}
frr version 8.0.1_git
frr defaults traditional
hostname frr-upstream                              👈[1]
!
debug bgp updates
debug bgp neighbor
debug zebra nht
debug bgp nht
debug bfd peer
log file /tmp/frr.log debugging
log timestamp precision 3
!
interface virbr2                                   👈[2]
 ip address 192.168.133.1/24                       👈[3]
!
router bgp 64521                                   👈[4]
 bgp router-id 192.168.133.1                       👈[5]
 timers bgp 3 15                                   👈[6]
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 no bgp network import-check
 neighbor metallb peer-group
 neighbor metallb remote-as 64520                  👈[7]
 neighbor 192.168.133.71 peer-group metallb        👈[8]
 neighbor 192.168.133.71 bfd                       👈[9]
 neighbor 192.168.133.72 peer-group metallb
 neighbor 192.168.133.72 bfd
!
 address-family ipv4 unicast
  neighbor 192.168.133.71 next-hop-self            👈[10]
  neighbor 192.168.133.71 activate                 👈[11]
  neighbor 192.168.133.72 next-hop-self
  neighbor 192.168.133.72 activate
 exit-address-family
!
line vty

{% endhighlight %}
```{epigraph}
  - frr.conf -
```

The important sections to adjust of the previous configuration file are the following:

 **[1] hostname <NAME>:** in my case is frr-bgp. <br/>
 **[2] interface <DEV>:** the interface name that is in the same subnet as the OCP worker nodes. <br/>
 **[3] ip address <IP/PREFIX>:** External host IP address and prefix, 192.168.133.1/24.<br/>
 **[4] router bgp <ASN>:** pick the ASN for the external router, 64521.<br/>
 **[5] bgp router-id <IP>:** pick the IP for the external router host, 192.168.133.1.<br/>
 **[6] timers bgp 3 15:** BGP hold time (15 secs) and keepalive timeout (3 secs). It can be adjusted to your needs.<br/>
 **[7] neighbor metallb remote-as <ASN>:** the remote (MetalLB) ASN, 64520.<br/>
 **[8] neighbor <IP> peer-group metallb:** each OCP node that runs a speaker pod should be identified as neighbour. I also mark these peers as are part of the peer-group metallb.<br/>
 **[9] neighbor <IP> bfd:** Enable BFD with the neighbour in question.<br/>
 **[10] neighbor <IP> next-hop-self:** tells FRR that the routes learned from this neighbor will have the BGP router address as the next hop.<br/>
 **[11] neighbor <IP> activate:** states that the IPs listed will have the IPv4 address family enabled, and will receive announcements from this router. <br/>

For more details on FRR BGP configuration check their documentation[^7].


{% highlight console %}
# This file tells the frr package which daemons to start.
#
# Sample configurations for these daemons can be found in
# /usr/share/doc/frr/examples/.
#
# ATTENTION:
#
# When activating a daemon for the first time, a config file, even if it is
# empty, has to be present *and* be owned by the user and group "frr", else
# the daemon will not be started by /etc/init.d/frr. The permissions should
# be u=rw,g=r,o=.
# When using "vtysh" such a config file is also needed. It should be owned by
# group "frrvty" and set to ug=rw,o= though. Check /etc/pam.d/frr, too.
#
# The watchfrr, zebra and staticd daemons are always started.
#
bgpd=yes
ospfd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=yes
fabricd=no
vrrpd=no
pathd=no

#
# If this option is set the /etc/init.d/frr script automatically loads
# the config via "vtysh -b" when the servers are started.
# Check /etc/pam.d/frr if you intend to use "vtysh"!
#
#
vtysh_enable=yes
zebra_options="  -A 127.0.0.1 -s 90000000"
bgpd_options="   -A 127.0.0.1"
ospfd_options="  -A 127.0.0.1"
ospf6d_options=" -A ::1"
ripd_options="   -A 127.0.0.1"
ripngd_options=" -A ::1"
isisd_options="  -A 127.0.0.1"
pimd_options="   -A 127.0.0.1"
ldpd_options="   -A 127.0.0.1"
nhrpd_options="  -A 127.0.0.1"
eigrpd_options=" -A 127.0.0.1"
babeld_options=" -A 127.0.0.1"
sharpd_options=" -A 127.0.0.1"
pbrd_options="   -A 127.0.0.1"
staticd_options="-A 127.0.0.1"
bfdd_options="   -A 127.0.0.1"
fabricd_options="-A 127.0.0.1"
vrrpd_options="  -A 127.0.0.1"
pathd_options="  -A 127.0.0.1"

# configuration profile
#
#frr_profile="traditional"
#frr_profile="datacenter"

#
# This is the maximum number of FD's that will be available.
# Upon startup this is read by the control files and ulimit
# is called.  Uncomment and use a reasonable value for your
# setup if you are expecting a large number of peers in
# say BGP.
MAX_FDS=1024
# The list of daemons to watch is automatically generated by the init script.
#watchfrr_options=""

# To make watchfrr create/join the specified netns, use the following option:
#watchfrr_options="--netns"
# This only has an effect in /etc/frr/<somename>/daemons, and you need to
# start FRR with "/usr/lib/frr/frrinit.sh start <somename>".

# for debugging purposes, you can specify a "wrap" command to start instead
# of starting the daemon directly, e.g. to use valgrind on ospfd:
#   ospfd_wrap="/usr/bin/valgrind"
# or you can use "all_wrap" for all daemons, e.g. to use perf record:
#   all_wrap="/usr/bin/perf record --call-graph -"
# the normal daemon command is added to this at the end.
{% endhighlight %}
```{epigraph}
  - daemons -
```
The daemons file only needs to ensure the right daemons are enabled bgpd and bfdd.

{% highlight console %}
$ tree /home/frr/
/home/frr/
├── daemons
├── frr.conf
└── vtysh.conf

$ podman run -d --rm  -v /home/maur0x/frr:/etc/frr:Z --net=host --name frr-upstream --privileged quay.io/frrouting/frr
bdf2b6a9ebe087acc7df8b57f86dbf7e3f253f5df66ced360a18d2d40574b8f1
{% endhighlight %}

Finally allow BGP (179/TCP) and BFD (3784/UDP and 3785/UDP) communications to go through the firewall if you have one running between OpenShift nodes and the router. 
Since in my LAB environment the host running the FRR pod is a Fedora hypervisor and the OpenShift cluster is virtualized, I will allow this port via firwalld and in the ```libvirt``` zone.

{% highlight console %}
$ firewall-cmd --zone=libvirt --add-port=179/tcp --add-port=3784/udp --add-port=3785/udp --permanent
success
$ firewall-cmd --reload
success
{% endhighlight %}


#### Metal LB

The follwing steps will install and configure MetalLB.

##### 1. Check the operator is available and install it

{% highlight console %}

$ oc get packagemanifest | grep metal
metallb-operator                                    Red Hat Operators     10m

$ cat << _EOF_ | oc apply -f -
---
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
spec: {}
_EOF_
namespace/metallb-system created


$ cat << _EOF_ | oc apply -f -
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: metallb-operator
  namespace: metallb-system
spec:
  targetNamespaces:
  - metallb-system
_EOF_
operatorgroup.operators.coreos.com/metallb-operator created

$ cat << _EOF_ | oc apply -f -
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: metallb-operator-sub
  namespace: metallb-system
spec:
  name: metallb-operator
  channel: "4.10"
  source: redhat-operators
  sourceNamespace: openshift-marketplace
_EOF_
subscription.operators.coreos.com/metallb-operator-sub created

{% endhighlight %}


##### 2. Check the installation process and the install plan

{% highlight console %}
$ oc get csv -A
NAMESPACE                              NAME                                   DISPLAY            VERSION               REPLACES   PHASE
metallb-system                         metallb-operator.4.10.0-202203081809   MetalLB Operator   4.10.0-202203081809              Succeeded
openshift-operator-lifecycle-manager   packageserver                          Package Server     0.19.0                           Succeeded


$ oc get installplan -n metallb-system
NAME            CSV                                    APPROVAL    APPROVED
install-brb9w   metallb-operator.4.10.0-202203081809   Automatic   true
{% endhighlight %}

##### 3. Create a MetalLB resource and check the controller deployment

{% highlight console %}
$ cat << _EOF_ | oc apply -f -
---
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
_EOF_
metallb.metallb.io/metallb created
{% endhighlight %}

```{note}
**NOTE:** In this step we can select which nodes will be running the speakers via _spec.nodeSelector_. This can typically be worker nodes or infra nodes.
```

{% highlight console %}

$  oc get deployment -n metallb-system controller 
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
controller   1/1     1            1           91s

$ oc get deployment -n metallb-system controller -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2022-03-17T12:42:31Z"
  generation: 1
  labels:
    app: metallb
    component: controller
  name: controller
  namespace: metallb-system
  ownerReferences:
  - apiVersion: metallb.io/v1beta1
    blockOwnerDeletion: true
    controller: true
    kind: MetalLB
    name: metallb
    uid: 10fe2c8f-7122-43ee-a735-b3062caf97e6
  resourceVersion: "464444"
  uid: 3bcc9c38-0489-4005-9fcc-ad6341461e5f
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 3
...
{% endhighlight %}


##### 4. Create an AddressPool resource

As mentioned before, the ```AddressPool``` resource will tell MetalLB which External IP addresses are valid to be assigned to a ```LoadBalancer``` service.
These addresses can be specified as subnet range, or individual addresses like the example below. To avoid collissions these IP addresses should be available and reserved for this use. The protocol has to be set to ```bgp```. The other protocol option is the older method ```layer2``` which relies on ARP/NDP to advertise the addresses instead of BGP.

{% highlight console %}
$ cat << _EOF_ | oc apply -f -
---
apiVersion: metallb.io/v1beta1
kind: AddressPool
metadata:
  name: address-pool-bgp
  namespace: metallb-system
spec:
  addresses:
  - 192.168.133.150/32
  - 192.168.133.151/32
  - 192.168.133.152/32
  - 192.168.133.153/32
  - 192.168.133.154/32
  - 192.168.133.155/32
  autoAssign: true
  protocol: bgp
_EOF_
addresspool.metallb.io/address-pool-bgp created
{% endhighlight %}

##### 4. Create a BFD profile

The BFD profile holds the configuration and timeouts for BFD protocol.

{% highlight console %}
$ cat << _EOF_ | oc apply -f -
---
apiVersion: metallb.io/v1beta1
kind: BFDProfile
metadata:
  name: test-bfd-prof
  namespace: metallb-system
spec:
  detectMultiplier: 37
  echoMode: true
  minimumTtl: 10
  passiveMode: true
  receiveInterval: 35
  transmitInterval: 35
_EOF_
bfdprofile.metallb.io/test-bfd-prof created
{% endhighlight %}

##### 5. Create a BGPPeer resource

Finally create the BGPPeer resource which will pass to the seaker pods' frr container the right ASN numbers, local and remote, as well as the remote peer IP address.

{% highlight console %}
$ cat << _EOF_ | oc apply -f -
---
apiVersion: metallb.io/v1beta1
kind: BGPPeer
metadata:
  name: peer-test
  namespace: metallb-system
spec:
  bfdProfile: test-bfd-prof
  myASN: 64520
  peerASN: 64521
  peerAddress: 192.168.133.1
_EOF_
bgppeer.metallb.io/peer-test created
{% endhighlight %}

The speaker pods should be up and running in the worker nodes by now.

{% highlight console %}
oc get pods -n metallb-system
NAME                                                   READY   STATUS    RESTARTS   AGE
controller-5bcbccf6d4-lhp95                            2/2     Running   0          71s
metallb-operator-controller-manager-654df86cc5-szk96   1/1     Running   0          14m
speaker-bt8z2                                          6/6     Running   0          71s
speaker-czhc5                                          6/6     Running   0          71s

$oc get ds -n metallb-system
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                     AGE
speaker   2         2         2       2            2           node-role.kubernetes.io/worker=   110s
{% endhighlight %}


### Verification

Now that the environment is up and running lets verify it is behaving as expected.
Firstly, I will check that there are valid BGP sessions established in my speaker pods, for example ```speaker-6jsfc```.
The BGP state should be **Established**, and BFD status should be **Up**.

{% highlight console %}
BGP neighbor is 192.168.133.1, remote AS 64521, local AS 64520, external link
Hostname: ice-lab-01.lab.local
  BGP version 4, remote router ID 192.168.133.1, local router ID 192.168.133.71
  BGP state = Established, up for 04:20:09
  Last read 00:00:00, Last write 00:00:03
  Hold time is 15, keepalive interval is 5 seconds
  Configured hold time is 90, keepalive interval is 30 seconds
  Neighbor capabilities:
    4 Byte AS: advertised and received
    AddPath:
      IPv4 Unicast: RX advertised IPv4 Unicast and received
      IPv6 Unicast: RX advertised IPv6 Unicast
    Route refresh: advertised and received(old & new)
    Address Family IPv4 Unicast: advertised and received
    Address Family IPv6 Unicast: advertised
    Hostname Capability: advertised (name: ice-ocp4-worker-0.lab.local,domain name: n/a) received (name: ice-lab-01.lab.local,domain name: n/a)
    Graceful Restart Capability: advertised and received
      Remote Restart timer is 120 seconds
      Address families by peer:
        none
  Graceful restart information:
    End-of-RIB send: IPv4 Unicast
    End-of-RIB received: IPv4 Unicast
    Local GR Mode: Helper*
    Remote GR Mode: Helper
    R bit: False
    Timers:
      Configured Restart Time(sec): 120
      Received Restart Time(sec): 120
    IPv4 Unicast:
      F bit: False
      End-of-RIB sent: Yes
      End-of-RIB sent after update: Yes
      End-of-RIB received: Yes
      Timers:
        Configured Stale Path Time(sec): 360
    IPv6 Unicast:
      F bit: False
      End-of-RIB sent: No
      End-of-RIB sent after update: No
      End-of-RIB received: No
      Timers:
        Configured Stale Path Time(sec): 360
  Message statistics:
    Inq depth is 0
    Outq depth is 0
                         Sent       Rcvd
    Opens:                  1          1
    Notifications:          0          0
    Updates:                1          1
    Keepalives:          3122       5204
    Route Refresh:          2          0
    Capability:             0          0
    Total:               3126       5206
  Minimum time between advertisement runs is 0 seconds

 For address family: IPv4 Unicast
  Update group 3, subgroup 3
  Packet Queue length 0
  Community attribute sent to this neighbor(all)
  Inbound path policy configured
  Route map for incoming advertisements is *192.168.133.1-in
  0 accepted prefixes

 For address family: IPv6 Unicast
  Not part of any update group
  Community attribute sent to this neighbor(all)
  Inbound path policy configured
  Route map for incoming advertisements is *192.168.133.1-in
  0 accepted prefixes

  Connections established 1; dropped 0
  Last reset 04:20:10,  Waiting for peer OPEN
Local host: 192.168.133.71, Local port: 37226
Foreign host: 192.168.133.1, Foreign port: 179
Nexthop: 192.168.133.71
Nexthop global: ::
Nexthop local: ::
BGP connection: shared network
BGP Connect Retry Timer in Seconds: 120
Read thread: on  Write thread: on  FD used: 22

  BFD: Type: single hop
    Detect Multiplier: 3, Min Rx interval: 300, Min Tx interval: 300
    Status: Up, Last update: 0:04:19:58

{% endhighlight %}

I can also check the pod's running configurtaion

{% highlight console %}
$  oc -n metallb-system exec -it speaker-bt8z2 -c frr -- vtysh -c "show running"
Building configuration...

Current configuration:
!
frr version 7.5
frr defaults traditional
hostname ice-ocp4-worker-0.lab.local
log file /etc/frr/frr.log informational
log timestamp precision 3
service integrated-vtysh-config
!
router bgp 64520
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 no bgp network import-check
 neighbor 192.168.133.1 remote-as 64521
 neighbor 192.168.133.1 bfd profile test-bfd-prof
 neighbor 192.168.133.1 timers 30 90
 !
 address-family ipv4 unicast
  neighbor 192.168.133.1 activate
  neighbor 192.168.133.1 route-map 192.168.133.1-in in
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor 192.168.133.1 activate
  neighbor 192.168.133.1 route-map 192.168.133.1-in in
 exit-address-family
!
route-map 192.168.133.1-in deny 20
!
route-map 192.168.133.1-out permit 1
!
ip nht resolve-via-default
!
ipv6 nht resolve-via-default
!
line vty
!
bfd
 profile test-bfd-prof
  detect-multiplier 37
  transmit-interval 35
  receive-interval 35
  passive-mode
  echo-mode
  minimum-ttl 10
 !
!
end

{% endhighlight %}

Similarily if I check the standalone FRR pod the  output should be similar against each peer.

{% highlight console %}
sudo podman exec -it  frr-upstream vtysh -c "show ip bgp neighbor"
BGP neighbor is 192.168.133.71, remote AS 64520, local AS 64521, external link
Hostname: ice-ocp4-worker-0.lab.local
 Member of peer-group metallb for session parameters
  BGP version 4, remote router ID 192.168.133.71, local router ID 192.168.133.1
  BGP state = Established, up for 04:35:09
  Last read 00:00:03, Last write 00:00:03
  Hold time is 15, keepalive interval is 3 seconds
  Configured hold time is 15, keepalive interval is 3 seconds
  Neighbor capabilities:
    4 Byte AS: advertised and received
    Extended Message: advertised
    AddPath:
      IPv4 Unicast: RX advertised and received
    Long-lived Graceful Restart: advertised
    Route refresh: advertised and received(old & new)
    Enhanced Route Refresh: advertised
    Address Family IPv4 Unicast: advertised and received
    Address Family IPv6 Unicast: received
    Hostname Capability: advertised (name: ice-lab-01.lab.local,domain name: n/a) received (name: ice-ocp4-worker-0.lab.local,domain name: n/a)
    Graceful Restart Capability: advertised and received
      Remote Restart timer is 120 seconds
      Address families by peer:
        none
  Graceful restart information:
    End-of-RIB send: IPv4 Unicast
    End-of-RIB received: IPv4 Unicast
    Local GR Mode: Helper*
    Remote GR Mode: Helper
    R bit: True
    Timers:
      Configured Restart Time(sec): 120
      Received Restart Time(sec): 120
    IPv4 Unicast:
      F bit: False
      End-of-RIB sent: Yes
      End-of-RIB sent after update: Yes
      End-of-RIB received: Yes
      Timers:
        Configured Stale Path Time(sec): 360
  Message statistics:
    Inq depth is 0
    Outq depth is 0
                         Sent       Rcvd
    Opens:                  2          2
    Notifications:          0          2
    Updates:                2          2
    Keepalives:          5550       3330
    Route Refresh:          0          2
    Capability:             0          0
    Total:               5554       3338
  Minimum time between advertisement runs is 0 seconds

 For address family: IPv4 Unicast
  metallb peer-group member
  Update group 2, subgroup 2
  Packet Queue length 0
  NEXT_HOP is always this router
  Community attribute sent to this neighbor(all)
  0 accepted prefixes

  Connections established 2; dropped 1
  Last reset 04:35:35,  No AFI/SAFI activated for peer
Local host: 192.168.133.1, Local port: 179
Foreign host: 192.168.133.71, Foreign port: 37226
Nexthop: 192.168.133.1
Nexthop global: ::
Nexthop local: ::
BGP connection: shared network
BGP Connect Retry Timer in Seconds: 120
Read thread: on  Write thread: on  FD used: 26

  BFD: Type: single hop
  Detect Multiplier: 3, Min Rx interval: 300, Min Tx interval: 300
  Status: Up, Last update: 0:04:34:58

BGP neighbor is 192.168.133.72, remote AS 64520, local AS 64521, external link
Hostname: ice-ocp4-worker-1.lab.local
 Member of peer-group metallb for session parameters
  BGP version 4, remote router ID 192.168.133.72, local router ID 192.168.133.1
  BGP state = Established, up for 04:35:09
  Last read 00:00:03, Last write 00:00:03
  Hold time is 15, keepalive interval is 3 seconds
  Configured hold time is 15, keepalive interval is 3 seconds
  Neighbor capabilities:
    4 Byte AS: advertised and received
    Extended Message: advertised
    AddPath:
      IPv4 Unicast: RX advertised and received
    Long-lived Graceful Restart: advertised
    Route refresh: advertised and received(old & new)
    Enhanced Route Refresh: advertised
    Address Family IPv4 Unicast: advertised and received
    Address Family IPv6 Unicast: received
    Hostname Capability: advertised (name: ice-lab-01.lab.local,domain name: n/a) received (name: ice-ocp4-worker-1.lab.local,domain name: n/a)
    Graceful Restart Capability: advertised and received
      Remote Restart timer is 120 seconds
      Address families by peer:
        none
  Graceful restart information:
    End-of-RIB send: IPv4 Unicast
    End-of-RIB received: IPv4 Unicast
    Local GR Mode: Helper*
    Remote GR Mode: Helper
    R bit: True
    Timers:
      Configured Restart Time(sec): 120
      Received Restart Time(sec): 120
    IPv4 Unicast:
      F bit: False
      End-of-RIB sent: Yes
      End-of-RIB sent after update: Yes
      End-of-RIB received: Yes
      Timers:
        Configured Stale Path Time(sec): 360
  Message statistics:
    Inq depth is 0
    Outq depth is 0
                         Sent       Rcvd
    Opens:                  2          2
    Notifications:          0          2
    Updates:                2          2
    Keepalives:          5550       3330
    Route Refresh:          0          2
    Capability:             0          0
    Total:               5554       3338
  Minimum time between advertisement runs is 0 seconds

 For address family: IPv4 Unicast
  metallb peer-group member
  Update group 2, subgroup 2
  Packet Queue length 0
  NEXT_HOP is always this router
  Community attribute sent to this neighbor(all)
  0 accepted prefixes

  Connections established 2; dropped 1
  Last reset 04:35:35,  No AFI/SAFI activated for peer
Local host: 192.168.133.1, Local port: 179
Foreign host: 192.168.133.72, Foreign port: 50294
Nexthop: 192.168.133.1
Nexthop global: ::
Nexthop local: ::
BGP connection: shared network
BGP Connect Retry Timer in Seconds: 120
Read thread: on  Write thread: on  FD used: 27

  BFD: Type: single hop
  Detect Multiplier: 3, Min Rx interval: 300, Min Tx interval: 300
  Status: Up, Last update: 0:04:34:58
{% endhighlight %}


Now I will create a test service using the hello-node deployment to verify MetalLB is working as expected.

{% highlight console %}

$ oc new-project test-metallb
Now using project "test-metallb" on server "https://api.t1.lab.local:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/e2e-test-images/agnhost:2.33 -- /agnhost serve-hostname

$ oc create deployment hello-node --image=k8s.gcr.io/e2e-test-images/agnhost:2.33 -- /agnhost serve-hostname
deployment.apps/hello-node created

$ oc get all
NAME                              READY   STATUS              RESTARTS   AGE
pod/hello-node-78bd88f59b-5kswh   0/1     ContainerCreating   0          23s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-node   0/1     1            0           24s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-node-78bd88f59b   1         1         0       23s

$ oc expose deployment.apps/hello-node --port 80 --target-port 9376 --name=test-frr --type=LoadBalancer
service/test-frr exposed
{% endhighlight %}

The newly created ```LoadBalancer``` service is healthy, it is getting an External IP from the defined pool and has the right endpoint.

{% highlight console %}
oc get svc
NAME       TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)        AGE
test-frr   LoadBalancer   172.30.129.156   192.168.133.150   80:31749/TCP   38s


$ oc describe svc test-frr
Name:                     test-frr
Namespace:                test-metallb
Labels:                   app=hello-node
Annotations:              <none>
Selector:                 app=hello-node
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       172.30.129.156
IPs:                      172.30.129.156
LoadBalancer Ingress:     192.168.133.150
Port:                     <unset>  80/TCP
TargetPort:               9376/TCP
NodePort:                 <unset>  31749/TCP
Endpoints:                10.131.0.125:9376
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason        Age   From                Message
  ----    ------        ----  ----                -------
  Normal  IPAllocated   65s   metallb-controller  Assigned IP ["192.168.133.150"]
  Normal  nodeAssigned  65s   metallb-speaker     announcing from node "ice-ocp4-worker-1.lab.local"
  Normal  nodeAssigned  64s   metallb-speaker     announcing from node "ice-ocp4-worker-0.lab.local"
{% endhighlight %}

From the external FRR router pod the route to the external IP of the new service is learnt properly via BGP and the service is reachable on its external IP from other nodes in that network.

{% highlight console %}
$ sudo podman exec -it frr-upstream  bash
bash-5.1# ip r
default via 192.168.3.254 dev br0 proto static metric 425 
192.168.0.0/24 dev virbr1 proto kernel scope link src 192.168.0.254 linkdown 
192.168.3.254 dev br0 proto static scope link metric 20425 
192.168.4.0/24 dev vlan1001 proto kernel scope link src 192.168.4.2 metric 400 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 
192.168.133.0/24 dev virbr2 proto kernel scope link src 192.168.133.1 
192.168.133.150 nhid 494 proto bgp metric 20 
	nexthop via 192.168.133.71 dev virbr2 weight 1 
	nexthop via 192.168.133.72 dev virbr2 weight 1 

bash-5.1# exit

$ curl -l 192.168.133.150
hello-node-78bd88f59b-5kswh
{% endhighlight %}

Now the requests are routed uniformly across the worker nodes. Since this example is using OpenShift SDN and MetalLB respects the externalTrafficPolicy in place (which is the default ```cluster``` in this case), the traffic will be distributed evenly and in every node and then ```kube-proxy``` will take care of distributing the traffic to the running pods. This can be verified by inspecting the iptables rules created by ```kube-proxy``` on every node associated with the ```test-metallb``` project and the ```test-frr``` service.[^8]

{% highlight console %}
    0     0 KUBE-SVC-22I6P2EHQ5PNEHEH  tcp  --  *      *       0.0.0.0/0            172.30.129.156       /* test-metallb/test-frr cluster IP */ tcp dpt:80
 2280  137K KUBE-FW-22I6P2EHQ5PNEHEH  tcp  --  *      *       0.0.0.0/0            192.168.133.150      /* test-metallb/test-frr loadbalancer IP */ tcp dpt:80
    0     0 KUBE-SVC-22I6P2EHQ5PNEHEH  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-metallb/test-frr */ tcp dpt:31749
    0     0 KUBE-MARK-MASQ  tcp  --  !tun0  *       0.0.0.0/0            172.30.129.156       /* test-metallb/test-frr cluster IP */ tcp dpt:80
    0     0 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-metallb/test-frr */ tcp dpt:31749
 2281  137K KUBE-SEP-YIOY3MR4ZHJWXTUM  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-metallb/test-frr */
 2281  137K KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-metallb/test-frr loadbalancer IP */
 2281  137K KUBE-SVC-22I6P2EHQ5PNEHEH  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-metallb/test-frr loadbalancer IP */
    0     0 KUBE-MARK-DROP  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-metallb/test-frr loadbalancer IP */
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.131.0.125         0.0.0.0/0            /* test-metallb/test-frr */
 2281  137K DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-metallb/test-frr */ tcp to:10.131.0.125:9376
{% endhighlight %}

It is also important to remark that in order to distribute the traffic the ECMP implementation of the FRR external router, will in fact use a flow-based hashing so that all traffic associated with a particular flow uses the same next hop and the same path across the network. This means that if ingress traffic is coming from different sources it will be properly distributed, however if a given flow makes higher number of requests that traffic will end up in the same worker node.


### Conclusions

MetalLB is getting more and more mature as a project and using it in BGP mode offers a novel way to statelessly load balance traffic using standard routing facilities, instead of a regular load balancer network device. 

As we observed throughout this article, MetalLB also aids to achieve a substantially similar experience to what one would get in a public cloud provider, but on an on-prem platform provider, like a baremetal cluster. Moreover the even distribution of the traffic can also help to achieve better resiliency and better performance.

MetalLB documentation also points out to some limitations of the project that we should have in mind when architecting or implementing the solution[^9]. The main one being how BGP handles a peer going down, which we try to enhance via BFD. But the active connections associated with the node will be re-distributed to other nodes potentially breaking stateful connections.



### Resources

 [^1]: <https://metallb.universe.tf/>
 [^2]: <https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer>
 [^3]: <https://computerhistory.org/blog/the-two-napkin-protocol/>
 [^4]: <https://frrouting.org/>
 [^5]: <https://datatracker.ietf.org/doc/html/rfc1654>
 [^6]: <https://datatracker.ietf.org/doc/html/rfc5880>
 [^7]: <http://docs.frrouting.org/en/latest/bgp.html>
 [^8]: <https://metallb.org/usage/#bgp>
 [^9]: <https://metallb.org/concepts/bgp/>

