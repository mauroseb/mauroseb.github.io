---
layout: posts
title: "Single Node OpenShift with Static IPs"
date: 2022-03-14
categories: [blog]
tags: [ openshift, assisted_installer, sno, ocp, networking ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---
### Intro

[!IMPORTANT]
The twp methods described in this article are not longer valid since the GA of [Agent-based installs](https://docs.openshift.com/container-platform/4.12/installing/installing_with_agent_based_installer/preparing-to-install-with-agent-based-installer.html) in OpenShift 4.12.

The purpose of this post is to showcase different Single Node OpenShift (SNO) deployment procedures that fulfill one requirement in particular:
SNO cluster should be deployed with static IP addressing instead of the expected and supported dynamic addressing (DHCP).

First and foremost, what is Single Node OpenShift. Well, indeed, it is pretty obvious, an all-in-one node deployment of OCP. Where master services and worker services converge into a single node. This is a supported configuration that helps tremendously in certain scenarios like Edge deployments in telco and other industries.

This kind of deployment does not offer any embeded availabilty measures like a full deployment does, where there are multiple etcd instances running as a cluster, or the kubeapi and ingress controllers have multiple replicas to survive an outage of a single node. Neither the workloads running on it are relying on that, and that is the whole point. We can lose the entire node/cluster and there will be probably another SNO deployment somewhere with the similar workloads that may take over user traffic, for instance.

I will present two methods to do the same, one based in **Assisted Installer** and the other in a manual deployment method (like a UPI method). It is fair to say there is a third method, which is to use Advanced Cluster Management (ACM), but in the backend still uses assisted installer.

So let's get started.

<!-- more -->

**WARNING:** At the moment using static IP addressing is not supported so reach out to your account team if you have a need for it.

### Prerequisites

Regardless of the method to deploy, following are the prerequisites for deploying Single Node OpenShift cluster as referenced in the [official documentation](https://docs.openshift.com/container-platform/4.9/installing/installing_sno/install-sno-preparing-to-install-sno.html):

 * No DHCP is needed for static IPs
 * Node resources should match at least: 8 vCPUs, 32 GB RAM, 120 GB disk
 * For disconnected environment access to a local registry is required
 * Administration host from where to prepare the ISO
 * DNS records in place for wildcard domain, API endpoints, node forward and reverse records.

{% highlight console %}
$ dig +short master-0.t2.lab.local
master-0.lab.local.
192.168.122.60
$ dig +short api.t2.lab.local
t2.lab.local.
192.168.122.60
$ dig +short api-int.t2.lab.local
t2.lab.local.
192.168.122.60
$ dig +short test.apps.t2.lab.local
192.168.122.60
$ dig +short -x 192.168.122.60
master-0.lab.local.
{% endhighlight %}


###  Manual Procedure

#### Overview

The first method will be based in a manual procedure to deploy just like a UPI deployment but on a single node.

<img src="/images/sno-static-ip-manual-prod-1.png" alt="SNO manual 1 PNG" style="width:75%;"/>

##### PROs

 * Highly customizable installation method
 * Can work in disconnectd mode
 * Can do static IP addressing


##### CONs

 * It has some added complexity in contrast with using Assisted Installer
 * For static IPs to work the network configuration needs to be persisted to the disk. I will need to use a hack to do that. And that is another reason why this methods is not supported.


#### Steps for Manual Procedure

##### 1. Download the stock RHCOS live ISO

{% highlight console %}
$  oc version -o yaml
clientVersion:
  buildDate: "2022-01-12T23:54:57Z"
  compiler: gc
  gitCommit: da3f635eaf1b5ee52d49217c80059c2c8e85c7a7
  gitTreeState: clean
  gitVersion: 4.10.0-202201122238.p0.gda3f635.assembly.stream-da3f635
  goVersion: go1.17.2
  major: ""
  minor: ""
  platform: linux/amd64
openshiftVersion: 4.9.19
releaseClientVersion: 4.10.0-0.nightly-2022-01-17-091922
serverVersion:
  buildDate: "2022-01-26T11:26:42Z"
  compiler: gc
  gitCommit: 3a0f2c90b43e6cffd07f57b5b78dd9f083e47ee2
  gitTreeState: clean
  gitVersion: v1.22.3+2cb6068
  goVersion: go1.16.6
  major: "1"
  minor: "22"
  platform: linux/amd64
$ export OCP_VERSION=stable-4.9
$ curl -k https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$OCP_VERSION/openshift-install-linux.tar.gz > openshift-install-linux.tar.gz
$ tar xvzf openshift-install-linux.tar.gz
README.md
openshift-install
$ chmod +x openshift-install
$ ISO_URL=$(./openshift-install coreos print-stream-json | grep location | grep x86_64 | grep iso | cut -d\" -f4)
$ curl $ISO_URL > rhcos-live.x86_64.iso
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  987M  100  987M    0     0   717k      0  0:23:27  0:23:27 --:--:-- 1068k
{% endhighlight %}

##### 2. Modify the ISO to include the kernel arguments for Static IP addressing config at boot time

* Here I am passing the IP 192.168.122.60 to the NIC named enp2s0. That name may change depending on the platform so you may need to boot first with the live ISO to know what name is assigned by the kernel to that NIC.

{% highlight console %}
$ alias coreos-installer='podman run --privileged --rm \
        -v /dev:/dev -v /run/udev:/run/udev -v $PWD:/data -w /data \
         quay.io/coreos/coreos-installer:release'
$ coreos-installer iso kargs modify -a ip=192.168.122.60::192.168.122.1:255.255.255.0:master-0.lab.local:enp2s0:none -a nameserver=192.168.3.80 rhcos-live.x86_64.iso
{% endhighlight %}



##### 3. Create an install-config.yaml and hack it to make it burn the static IP config to the disk

This is a very interesting and important point to understand. What we have done in step 2 will ensure that the live CD image will use static IP addressing, so far so good. However at some point this image will burn itself to the disk by invoking a script that will call [coreos-install](https://raw.githubusercontent.com/coreos/init/master/bin/coreos-install) with some arguments. When that happens it will completely ignore the networking configuration that was passed to the live image. Unless!... We pass the flag **--copy-network** to it. The question is how to do this. One simple answer is to modify the script that invokes coreos-install within the live image, however do to an existing [bug](https://bugzilla.redhat.com/show_bug.cgi?id=2051478) in how the stanza of BootstrapInPlace.InstallationDisk (where we indicate the root disk) is parsed within the install-config.yaml, we can also piggy-back the flag to the disk, without requiring to modify the live image. I know. It is not the cleaneast way to achieve the same but right now is considered less intrusive than modifiying the live image, which would also be unsupported. BTW this bug will not be fixed until we have a supported way to do disconnected environments in through assisted installer (referenced RFE).

So the install config will look like this:

{% highlight console %}
$  cat install-config.yaml
apiVersion: v1
baseDomain: lab.local
metadata:
  name: t2
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 1
networking:
  networkType: OVNKubernetes
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
  machineCIDR: 192.168.122.0/24
platform:
  none: {}
bootstrapInPlace:
  installationDisk: ‘/dev/vda —copy-network’
pullSecret: {{ pull-secret }}
sshKey: |
  {{ ssh-key }}
{% endhighlight %}


For disconnected environments the install-config.yaml should include imageContentSources stanza pointing to the local image registry. Check restricted network installation [documentation](https://docs.openshift.com/container-platform/4.9/installing/installing_bare_metal/installing-restricted-networks-bare-metal.html) for more details. Following there is a reference snippet. Using SNO in restircted networks is fully supported.

{% highlight console %}
imageContentSources:
- mirrors:
  - ice-quay-01.lab.local/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - ice-quay-01.lab.local/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
{% endhighlight %}

##### 4. Generate an ignition config from the install-config.yaml

So as in a normal UPI method, we will create the ignition config for the node but in this case we specify a special subcommand to indicate that we want to merge the worker and master ignition config into one.

{% highlight console %}
$ mkdir ocp
$ cp install-config-sno.yaml ocp/install-config.yaml
$ ./openshift-install --dir=ocp create single-node-ignition-config
INFO Consuming Install Config from target directory
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
INFO Single-Node-Ignition-Config created in: ocp and ocp/auth
{% endhighlight %}

##### 5. This newly created ignition config file now has to be embedded into the live image

{% highlight console %}
$ alias coreos-installer='podman run --privileged --rm \
        -v /dev:/dev -v /run/udev:/run/udev -v $PWD:/data \
        -w /data quay.io/coreos/coreos-installer:release'
$ cp ocp/bootstrap-in-place-for-live-iso.ign iso.ign
$ coreos-installer iso ignition embed -fi iso.ign rhcos-live.x86_64.iso
Trying to pull quay.io/coreos/coreos-installer:release...
Getting image source signatures
Copying blob a3ed95caeb02 done  
Copying blob 708557ab4058 done  
Copying blob 1111aecae5c4 done  
Copying blob 6bd970480768 done  
Writing manifest to image destination
Storing signatures
{% endhighlight %}

##### 6. Boot the node with the live image

The final step is to boot the node with the live image which now has all the needed changes to boot using static IP addressing.

While the node starts one can verify the progress like with any other UPI deployment.

{% highlight console %}
$ ./openshift-install --dir=ocp wait-for install-complete
INFO Waiting up to 40m0s for the cluster at https://api.t2.lab.local:6443 to initialize...
INFO Waiting up to 10m0s for the openshift-console route to be created...
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/maur0x/ocp4-ai/ocp/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.t2.lab.local
INFO Login to the console with user: "kubeadmin", and password: "kgzvM-ghmbi-kaGQK-SCZuy"
INFO Time elapsed: 5m18s     
...
{% endhighlight %}

And after some time you should have a the cluster fully available.

{% highlight console %}

$ export KUBECONFIG=$PWD/ocp/auth/kubeconfig

$ oc cluster-info
Kubernetes control plane is running at https://api.t2.lab.local:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ oc get nodes
NAME                 STATUS   ROLES           AGE   VERSION
master-0.lab.local   Ready    master,worker   24m   v1.22.3+b93fd35
$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.9.23    True        False         False      151m    
baremetal                                  4.9.23    True        False         False      164m    
cloud-controller-manager                   4.9.23    True        False         False      159m    
cloud-credential                           4.9.23    True        False         False      159m    
cluster-autoscaler                         4.9.23    True        False         False      161m    
config-operator                            4.9.23    True        False         False      166m    
console                                    4.9.23    True        False         False      149m    
csi-snapshot-controller                    4.9.23    True        False         False      166m    
dns                                        4.9.23    True        False         False      155m    
etcd                                       4.9.23    True        False         False      161m    
image-registry                             4.9.23    True        False         False      152m    
ingress                                    4.9.23    True        False         False      158m    
insights                                   4.9.23    True        False         False      155m    
kube-apiserver                             4.9.23    True        False         False      159m    
kube-controller-manager                    4.9.23    True        False         False      162m    
kube-scheduler                             4.9.23    True        False         False      159m    
kube-storage-version-migrator              4.9.23    True        False         False      166m    
machine-api                                4.9.23    True        False         False      149m    
machine-approver                           4.9.23    True        False         False      163m    
machine-config                             4.9.23    True        False         False      158m    
marketplace                                4.9.23    True        False         False      163m    
monitoring                                 4.9.23    True        False         False      149m    
network                                    4.9.23    True        False         False      170m    
node-tuning                                4.9.23    True        False         False      159m    
openshift-apiserver                        4.9.23    True        False         False      155m    
openshift-controller-manager               4.9.23    True        False         False      154m    
openshift-samples                          4.9.23    True        False         False      153m    
operator-lifecycle-manager                 4.9.23    True        False         False      149m    
operator-lifecycle-manager-catalog         4.9.23    True        False         False      149m    
operator-lifecycle-manager-packageserver   4.9.23    True        False         False      149m    
service-ca                                 4.9.23    True        False         False      166m    
storage                                    4.9.23    True        False         False      159m    
{% endhighlight %}


### Assisted Installer Procedure

#### Overview

Assisted installer is offered as a SaaS service through the [Assisted Installer Portal](https://console.redhat.com/openshift/assisted-installer/clusters/), and has been created to improve the deployment experience of OpenShift and increase its success rate by creating an even more opinionated deployment sticking to our best practices. In turn it reduces the prerequisites needed to deploy. It can deploy multi node or single node clusters like I will exercise here. And it also counts with a CLI tool called [aicli](https://github.com/karmab/aicli) and an [on-prem](https://cloud.redhat.com/blog/assisted-installer-on-premise-deep-dive) version. There is an [Request for Enhancement](https://issues.redhat.com/browse/RFE-1936) to support disconnected environments but until it is implemented we have to rely in the manual method to deploy in them.

<img src="/images/sno-static-ip-aicli-prod-1.png" alt="SNO aicli 1 PNG" style="width:75%;"/>

##### PROs

 * Eases the deployment considerably
 * Applies best practices
 * Can do static IP addressing by passing the node configuration via NMState format

##### CONs

 * Only supports connected clusters for now, until the aforementioned RFE is in place


#### Extra Prerequisites to use AICLI

The following steps are only needed for using aicli CLI tool.

 1. Install aicli tool using pip in the administration host

{% highlight console %}
$ pip install aicli
Defaulting to user installation because normal site-packages is not writeable
WARNING: Keyring is skipped due to an exception: Failed to open keyring: org.freedesktop.DBus.Error.NoReply: Did not receive a reply. Possible causes include: the remote application did not send a reply, the message bus security policy blocked the reply, the reply timeout expired, or the network connection was broken..
Collecting aicli
  Downloading aicli-99.0.202202230727.202103111306-py3-none-any.whl (17 kB)
Collecting assisted-service-client
  Downloading assisted_service_client-2.1.1.post1-py3-none-any.whl (379 kB)
     |████████████████████████████████| 379 kB 2.0 MB/s
Requirement already satisfied: prettytable in /usr/lib/python3.9/site-packages (from aicli) (0.7.2)
Requirement already satisfied: PyYAML in /usr/lib64/python3.9/site-packages (from aicli) (5.4.1)
Requirement already satisfied: urllib3>=1.23 in /usr/lib/python3.9/site-packages (from assisted-service-client->aicli) (1.25.10)
Collecting certifi>=2017.4.17
  Downloading certifi-2021.10.8-py2.py3-none-any.whl (149 kB)
     |████████████████████████████████| 149 kB 4.9 MB/s
Requirement already satisfied: python-dateutil>=2.1 in /usr/lib/python3.9/site-packages (from assisted-service-client->aicli) (2.8.1)
Requirement already satisfied: six>=1.10 in /usr/lib/python3.9/site-packages (from assisted-service-client->aicli) (1.15.0)
Installing collected packages: certifi, assisted-service-client, aicli
Successfully installed aicli-99.0.202202230727.202103111306 assisted-service-client-2.1.1.post1 certifi-2021.10.8
{% endhighlight %}




 2. Get offline token from <https://cloud.redhat.com/openshift/token>

 3. Test aicli tool passing the obtained token

{% highlight console %}
$ export MYTOKEN=eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJhZDU…
$ aicli –offlinetoken $MYTOKEN list clusters
Storing new token in /home/maur0x/.aicli/token.txt
+---------+--------------------------------------+-----------+------------+
| Cluster |                  Id                  |   Status  | Dns Domain |
+---------+--------------------------------------+-----------+------------+
|    t2   | 6364ff09-f449-4f89-b661-dcfa93013584 | installed | lab.local  |
+---------+--------------------------------------+-----------+------------+
{% endhighlight %}


#### Steps for Assisted Installer Method

##### 1. Create the new cluster via aicli

The first step is to create a cluster instance to represent the new cluster within assisted installer.
The pull secret must be named openshift_pull.json and located in the current working directory or passed as a parameter to the create command.

{% highlight console %}
$ aicli create cluster t2 -P base_dns_domain=lab.local -P sno=true
Creating cluster t2
Using openshift_pull.json as pull_secret file
Using openshift_pull.json as pull_secret file
{% endhighlight %}

Once created you can check the cluster status, which should be in **pending-for-input** status....

{% highlight console %}

$ aicli list cluster
+---------+--------------------------------------+-------------------+------------+
| Cluster |                  Id                  |       Status      | Dns Domain |
+---------+--------------------------------------+-------------------+------------+
|    t2   | e8281344-5e2d-4e86-b4fb-6b8263f591b5 | pending-for-input | lab.local  |
+---------+--------------------------------------+-------------------+------------+

$ aicli info cluster t2
ams_subscription_id: xxxxxxxxxxxxxxxxxxxxxxxx
base_dns_domain: lab.local
cluster_network_cidr: 10.128.0.0/14
cluster_networks: [{'cluster_id': 'e8281344-5e2d-4e86-b4fb-6b8263f591b5', 'cidr': '10.128.0.0/14', 'host_prefix': 23}]
connectivity_majority_groups: {"IPv4":[],"IPv6":[]}
controller_logs_collected_at: 0001-01-01 00:00:00+00:00
controller_logs_started_at: 0001-01-01 00:00:00+00:00
cpu_architecture: x86_64
created_at: 2022-02-25 12:07:27.067865+00:00
disk_encryption: {'enable_on': 'none', 'mode': 'tpmv2', 'tang_servers': None}
email_domain: redhat.com
feature_usage: {"SDN network type":{"id":"SDN_NETWORK_TYPE","name":"SDN network type"},"SNO":{"id":"SNO","name":"SNO"}}
high_availability_mode: None
hyperthreading: all
id: e8281344-5e2d-4e86-b4fb-6b8263f591b5
ignition_endpoint: {'url': None, 'ca_certificate': None}
install_completed_at: 0001-01-01 00:00:00+00:00
install_started_at: 0001-01-01 00:00:00+00:00
machine_networks: []
monitored_operators: [{'cluster_id': 'e8281344-5e2d-4e86-b4fb-6b8263f591b5', 'name': 'console', 'namespace': None, 'subscription_name': None, 'operator_type': 'builtin', 'properties': None, 'timeout_seconds': 3600, 'status': None, 'status_info': None, 'status_updated_at': datetime.datetime(1, 1, 1, 0, 0, tzinfo=tzutc())}, {'cluster_id': 'e8281344-5e2d-4e86-b4fb-6b8263f591b5', 'name': 'cvo', 'namespace': None, 'subscription_name': None, 'operator_type': 'builtin', 'properties': None, 'timeout_seconds': 3600, 'status': None, 'status_info': None, 'status_updated_at': datetime.datetime(1, 1, 1, 0, 0, tzinfo=tzutc())}]
name: t2
network_type: OpenShiftSDN
ocp_release_image: quay.io/openshift-release-dev/ocp-release:4.9.19-x86_64
openshift_version: 4.9.19
org_id: 1979710
platform: {'type': 'baremetal', 'ovirt': {'fqdn': None, 'username': None, 'password': None, 'insecure': True, 'ca_bundle': None, 'cluster_id': None, 'storage_domain_id': None, 'network_name': 'ovirtmgmt', 'vnic_profile_id': None}}
progress: {'total_percentage': None, 'preparing_for_installation_stage_percentage': None, 'installing_stage_percentage': None, 'finalizing_stage_percentage': None}
schedulable_masters: False
service_network_cidr: 172.30.0.0/16
service_networks: [{'cluster_id': 'e8281344-5e2d-4e86-b4fb-6b8263f591b5', 'cidr': '172.30.0.0/16'}]
status: pending-for-input
status_info: User input required
status_updated_at: 2022-02-25 12:07:27.585000+00:00
updated_at: 2022-02-25 12:07:34.006467+00:00
user_managed_networking: True
user_name: <user>
{% endhighlight %}

As part of the cluster creation, and infraenv object is also created, which will hold the cluster configuration. We note the ID as we will need it in a further step.

{% highlight console %}
$ aicli list infraenv
+--------------+--------------------------------------+---------+-------------------+----------+
|   Infraenv   |                  Id                  | Cluster | Openshift Version | Iso Type |
+--------------+--------------------------------------+---------+-------------------+----------+
| t2_infra-env | 93a32b6d-c420-424b-a0f8-6cbd6d513071 |    t2   |        4.9        | full-iso |
+--------------+--------------------------------------+---------+-------------------+----------+
{% endhighlight %}


##### 2. Create the NMstate configuration that has the static IP addressing for the node

Following [example](https://github.com/openshift/assisted-service/blob/master/docs/user-guide/restful-api-guide.md) we can create a YAML file with the network configuration. In my case I am calling this file master-0.yaml. Then this file has to be embedded into a JSON request that will be eventually passed to the AI API to patch our cluster config (infraenv). Just to clarify, this NMSate formatted file has nothing to do with the Kubernetes NMState Operator and its configuration.


{% highlight console %}
$ cat master-0.yaml
dns-resolver:
 config:
    server:
    - 192.168.3.80
interfaces:
- name: enp1s0
  mac-address: 52:54:00:66:70:00
  ipv4:
    dhcp: dhcp
    enabled: true
  ipv6:
    enabled: false
  state: up
  type: ethernet
- name: enp2s0
  mac-address: 52:54:00:76:70:00
  ipv4:
    enabled: true
    address:
    - ip: 192.168.122.60
        prefix-length: 24
       dhcp: false
  ipv6:
    enabled: false
  state: up
  type: ethernet
routes:
  config:
  - destination: 0.0.0.0/0
    next-hop-address: 192.168.122.1
    next-hop-interface: enp2s0
    table-id: 254


$ export SSHKEY="`cat ~/.ssh/id_rsa.pub`"

$ export NMSTATE_MASTER0=$(jq -n --arg ARG1 "`cat master-0.yaml`" '$ARG1')

$ cat << _EOF_ >> nmstate.json
{
  "ssh_public_key": "$SSHKEY",
  "image_type": "full-iso",
  "static_network_config": [
    {
      "network_yaml": "$NMSTATE_MASTER0",
      "mac_interface_map": [{"mac_address": "52:54:00:76:70:00", "logical_nic_name": "enp2s0"}, {"mac_address": "54:52:00:66:70:00", "logical_nic_name": "enp1s0"}]
    }
  ]
}
_EOF_

$ cat nmstate.json
{
  "ssh_public_key": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC/k3AOCC9eLHg3KxhGkaUeWrF5FuY/f8jb8r+UjLCv5BBQLaELaYLQYgRxKN7u0/T7/OvMYvOKjNEAYPXYiENVFDvbMZxKftkml0bdlTfHvYRgvQXeTPHQ5qbDzhJXeabDOrljWtW3qw48w0SGiqlKMqU9ayyAfUNjxg2zCS42pVcGoVQq4uTGbrxJBmUpJZEP8sWqXrJRpx+8/7er0xDx2n8fhMym7trrniyztOZnQy0uV7UKh5qseYtam+dkMrA9drDNeL5YO+e3bAppIRVdBRnp0QkdIVI+dzb0HpTPNJ9f5cwfUPAopqD2XawayfKSmOrFhEALXMjT70dhKqq2sVUP6ZazZWnYNgbJdcBOp+s0Am5YNLU/Ae1Qbk+ZMFm6cg237GjwljX9g/3FKVbekNMZFuAVA6a3jAcMjBtnTqbzp2HHfVeWLJWWJSShtpv4cPQVfKoRgS7RauBHmWB0z6t3Wx/+P+3YyFQ7bs38K4x74j7QoRleQXxKlKGtcnWnn682XM9x1ovFfQytEVK0KIi0L+NX1hU41z3juNtk974xVtFnIlPKAIYP9CGXXCDz3VB2dCkLxSE63vkUg8ZaxGcGAq44eOCsHHMQhCd02krxvivBGXLiy4s2p6nuZEDtmp4tsoL2f352M4dmHSqNL5EqKQ4ljKH1xOtMosHVMw==",
  "image_type": "full-iso",
  "static_network_config": [
    {
      "network_yaml": "dns-resolver:\n config:\n    server:\n    - 192.168.3.80\ninterfaces:\n- name: enp1s0\n  mac-address: 52:54:00:66:70:00\n  ipv4:\n    dhcp: false\n    enabled: false\n  ipv6:\n    enabled: false\n  state: up\n  type: ethernet\n- name: enp2s0\n  mac-address: 52:54:00:76:70:00\n  ipv4:\n    enabled: true\n    address:\n    - ip: 192.168.122.60\n      prefix-length: 24\n    dhcp: false\n  ipv6:\n    enabled: false\n  state: up\n  type: ethernet\nroutes:\n  config:\n  - destination: 0.0.0.0/0\n    next-hop-address: 192.168.122.1\n    next-hop-interface: enp2s0\n    table-id: 254",
      "mac_interface_map": [{"mac_address": "52:54:00:76:70:00", "logical_nic_name": "enp2s0"}, {"mac_address": "52:54:00:66:70:00", "logical_nic_name": "enp1s0"}]
    }
  ]
}
{% endhighlight %}

##### 3. Apply the network config to the cluster by calling the Assisted Installer API

This API call is not available via the Assisted Installer portal yet. The API call should point to the infraenv ID captured before, and pass the nmstate.json file generated in the previous step as payload.


{% highlight console %}
$ export MYTOKEN=$(/bin/cat /home/maur0x/.aicli/token.txt)

$ curl -s -X PATCH -H "Content-Type: application/json" -H "Authorization: Bearer $MYTOKEN" -d @nmstate.json "https://api.openshift.com/api/assisted-install/v2/infra-envs/2077c047-527b-40ec-82e3-15963825791a"
{"cluster_id":"a8fe1b10-6663-415d-87f3-c90e2390594c","cpu_architecture":"x86_64","created_at":"2022-03-02T15:36:56.191894Z","download_url":"https://api.openshift.com/api/assisted-images/images/2077c047-527b-40ec-82e3-15963825791a?arch=x86_64&image_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NDYzMzM2MjEsInN1YiI6IjIwNzdjMDQ3LTUyN2ItNDBlYy04MmUzLTE1OTYzODI1NzkxYSJ9.W0C65W2CbtsvDzxtjvcsU_SKbCtd86wkHBo7-ChBmaY&type=full-iso&version=4.9","email_domain":"example.com","expires_at":"2022-03-03T18:53:41.000Z","href":"/api/assisted-install/v2/infra-envs/2077c047-527b-40ec-82e3-15963825791a","id":"2077c047-527b-40ec-82e3-15963825791a","kind":"InfraEnv","name":"t2_infra-env","openshift_version":"4.9","org_id":"1979710","proxy":{},"pull_secret_set":true,"ssh_authorized_key":"ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC/k3AOCC9eLHg3KxhGkaUeWrF5FuY/f8jb8r+UjLCv5BBQLaELaYLQYgRxKN7u0/T7/OvMYvOKjNEAYPXYiENVFDvbMZxKftkml0bdlTfHvYRgvQXeTPHQ5qbDzhJXeabDOrljWtW3qw48w0SGiqlKMqU9ayyAfUNjxg2zCS42pVcGoVQq4uTGbrxJBmUpJZEP8sWqXrJRpx+8/7er0xDx2n8fhMym7trrniyztOZnQy0uV7UKh5qseYtam+dkMrA9drDNeL5YO+e3bAppIRVdBRnp0QkdIVI+dzb0HpTPNJ9f5cwfUPAopqD2XawayfKSmOrFhEALXMjT70dhKqq2sVUP6ZazZWnYNgbJdcBOp+s0Am5YNLU/Ae1Qbk+ZMFm6cg237GjwljX9g/3FKVbekNMZFuAVA6a3jAcMjBtnTqbzp2HHfVeWLJWWJSShtpv4cPQVfKoRgS7RauBHmWB0z6t3Wx/+P+3YyFQ7bs38K4x74j7QoRleQXxKlKGtcnWnn682XM9x1ovFfQytEVK0KIi0L+NX1hU41z3juNtk974xVtFnIlPKAIYP9CGXXCDz3VB2dCkLxSE63vkUg8ZaxGcGAq44eOCsHHMQhCd02krxvivBGXLiy4s2p6nuZEDtmp4tsoL2f352M4dmHSqNL5EqKQ4ljKH1xOtMosHVMw== maur0x@ice-lab-01.lab.local","static_network_config":"dns-resolver:\n config:\n    server:\n    - 192.168.3.80\ninterfaces:\n- name: enp1s0\n  mac-address: 52:54:00:66:70:00\n  ipv4:\n    dhcp: false\n    enabled: false\n  ipv6:\n    enabled: false\n  state: up\n  type: ethernet\n- name: enp2s0\n  mac-address: 52:54:00:76:70:00\n  ipv4:\n    enabled: true\n    address:\n    - ip: 192.168.122.60\n      prefix-length: 24\n    dhcp: false\n  ipv6:\n    enabled: false\n  state: up\n  type: ethernet\nroutes:\n  config:\n  - destination: 0.0.0.0/0\n    next-hop-address: 192.168.122.1\n    next-hop-interface: enp2s0\n    table-id: 254HHHHH52:54:00:66:70:00=enp1s0\n52:54:00:76:70:00=enp2s0","type":"full-iso","updated_at":"2022-03-03T14:53:42.8136Z","user_name":"{{ username }}"}
{% endhighlight %}


##### 4. Download the ISO from Assisted Installer service specifying the cluster config name (infraenv).

{% highlight console %}
$ aicli download iso t2
Downloading Iso for infraenv t2 in .
Storing new token in /home/maur0x/.aicli/token.txt
Generating new iso url---------------------------+---------+--------------+-------------+--------+-----+
{% endhighlight %}

You can use wget/curl as alternative here.

##### 5. Boot the node with the new downloaded ISO (t2.iso) and set the machine_network_cidr

After a short while the node will sohw up in the hosts listing in **discovering** state.

{% highlight console %}
$ aicli list hosts
+--------------------------------------+--------------------------------------+---------+--------------+-------------+--------+-----+
|                 Host                 |                  Id                  | Cluster |   Infraenv   |    Status   |  Role  |  Ip |
+--------------------------------------+--------------------------------------+---------+--------------+-------------+--------+-----+
| a2ccff37-10a1-49e9-9631-ab0f5636161b | a2ccff37-10a1-49e9-9631-ab0f5636161b |    t2   | t2_infra-env | discovering | master | N/A |
+--------------------------------------+--------------------------------------+---------+--------------+-------------+--------+-----+
{% endhighlight %}


After the discovery, the node will likely move to **insufficient** state, which could typically mean the host does not meet the resource capacity for SNO. However in this case the reason is that the machineNetwork is still unknown to the installer, therefore the assisted installer does not know if the node will have access to the right network. To correct this, as an additional step, set the machine_network_cidr parameter and that will move the node to **known** state.

{% highlight console %}
$ aicli list hosts
+--------------------+--------------------------------------+---------+--------------+--------------+--------+-----+
|        Host        |                  Id                  | Cluster |   Infraenv   |    Status    |  Role  |  Ip |
+--------------------+--------------------------------------+---------+--------------+--------------+--------+-----+
| master-0.lab.local | a2ccff37-10a1-49e9-9631-ab0f5636161b |    t2   | t2_infra-env | insufficient | master | N/A |
+--------------------+--------------------------------------+---------+--------------+--------------+--------+-----+

$ aicli update cluster t2 -P machine_network_cidr=192.168.122.0/24
Updating Cluster t2

$ aicli list hosts
+--------------------+--------------------------------------+---------+--------------+--------+--------+-----+
|        Host        |                  Id                  | Cluster |   Infraenv   | Status |  Role  |  Ip |
+--------------------+--------------------------------------+---------+--------------+--------+--------+-----+
| master-0.lab.local | a2ccff37-10a1-49e9-9631-ab0f5636161b |    t2   | t2_infra-env | known  | master | N/A |
+--------------------+--------------------------------------+---------+--------------+--------+--------+-----+
{% endhighlight %}

##### 6. Launch the cluster installation

Now that we have an node ready to deploy our cluster into, let's proceed.

{% highlight console %}
$ aicli start cluster t2
Starting cluster t2
Storing new token in /home/maur0x/.aicli/token.txt

$ aicli info cluster t2 | grep -e status: -e status_info: -e  progress:
progress: {'total_percentage': None, 'preparing_for_installation_stage_percentage': None, 'installing_stage_percentage': None, 'finalizing_stage_percentage': None}
status: preparing-for-installation
status_info: Preparing cluster for installation
{% endhighlight %}

After some minutes the status will move to **installing**

{% highlight console %}
$ aicli info cluster t2 | grep -e status: -e status_info: -e  progress:
progress: {'total_percentage': 10, 'preparing_for_installation_stage_percentage': 100, 'installing_stage_percentage': None, 'finalizing_stage_percentage': None}
status: installing
status_info: Installation in progress
{% endhighlight %}


**WARNING:** If the boot media was not properly removed the installer will ask for user action to do so.

{% highlight console %}
$ aicli info cluster t2 | grep -e status: -e status_info: -e  progress:
progress: {'total_percentage': 68, 'preparing_for_installation_stage_percentage': 100, 'installing_stage_percentage': 83, 'finalizing_stage_percentage': None}
status: installing-pending-user-action
status_info: Cluster has hosts pending user action
{% endhighlight %}


After correcting that the cluster will move to **installed** state.

{% highlight console %}
$ aicli info cluster t2 | grep -e status: -e status_info: -e  progress:
progress: {'total_percentage': 100, 'preparing_for_installation_stage_percentage': 100, 'installing_stage_percentage': 100, 'finalizing_stage_percentage': 100}
status: installed
status_info: Cluster is installed
{% endhighlight %}



And we have a fully working SNO cluster with Static IP addressing.
Download the kubeadmin password and the kubeconfig to check on it.

Download kubeadmin password and kubeconfig files and check the cluster
$ aicli download kubeconfig t2
Downloading Kubeconfig for Cluster t2 in ./kubeconfig.t2

$ aicli download kubeadmin-password t2
Downloading KubeAdminPassword for Cluster t2 in ./kubeadmin-password.t2

$ oc get nodes
NAME                 STATUS   ROLES           AGE   VERSION
master-0.lab.local   Ready    master,worker   25m   v1.22.3+2cb6068

$ oc cluster-info
Kubernetes control plane is running at https://api.t2.lab.local:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

{% highlight console %}
$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.9.19    True        False         False      3m46s   
baremetal                                  4.9.19    True        False         False      18m     
cloud-controller-manager                   4.9.19    True        False         False      17m     
cloud-credential                           4.9.19    True        False         False      68m     
cluster-autoscaler                         4.9.19    True        False         False      18m     
config-operator                            4.9.19    True        False         False      27m     
console                                    4.9.19    True        False         False      3m21s   
csi-snapshot-controller                    4.9.19    True        False         False      26m     
dns                                        4.9.19    True        False         False      20m     
etcd                                       4.9.19    True        False         False      21m     
image-registry                             4.9.19    True        False         False      6m29s   
ingress                                    4.9.19    True        False         False      17m     
insights                                   4.9.19    True        False         False      17m     
kube-apiserver                             4.9.19    True        False         False      17m     
kube-controller-manager                    4.9.19    True        False         False      19m     
kube-scheduler                             4.9.19    True        False         False      19m     
kube-storage-version-migrator              4.9.19    True        False         False      26m     
machine-api                                4.9.19    True        False         False      14m     
machine-approver                           4.9.19    True        False         False      23m     
machine-config                             4.9.19    True        False         False      20m     
marketplace                                4.9.19    True        False         False      23m     
monitoring                                 4.9.19    True        False         False      33s     
network                                    4.9.19    True        False         False      27m     
node-tuning                                4.9.19    True        False         False      22m     
openshift-apiserver                        4.9.19    True        False         False      15m     
openshift-controller-manager               4.9.19    True        False         False      3m14s   
openshift-samples                          4.9.19    True        False         False      17m     
operator-lifecycle-manager                 4.9.19    True        False         False      15m     
operator-lifecycle-manager-catalog         4.9.19    True        False         False      16m     
operator-lifecycle-manager-packageserver   4.9.19    True        False         False      13m     
service-ca                                 4.9.19    True        False         False      26m     
storage                                    4.9.19    True        False         False      18m     
{% endhighlight %}


As a final verification we can see the static IP addressing configuration exists within the SNO node.

{% highlight console %}
$ ssh core@192.168.122.60
Warning: Permanently added '192.168.122.60' (ED25519) to the list of known hosts.
Red Hat Enterprise Linux CoreOS 49.84.202201262103-0
  Part of OpenShift 4.9, RHCOS is a Kubernetes native operating system
  managed by the Machine Config Operator (`clusteroperator/machine-config`).

WARNING: Direct SSH access to machines is not recommended; instead,
make configuration changes via `machineconfig` objects:
  https://docs.openshift.com/container-platform/4.9/architecture/architecture-rhcos.html

---
[core@master-0 ~]$ sudo -i
[root@master-0 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:66:70:00 brd ff:ff:ff:ff:ff:ff
3: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:76:70:00 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.60/24 brd 192.168.122.255 scope global noprefixroute enp2s0
       valid_lft forever preferred_lft forever
[root@master-0 ~]# cat /etc/NetworkManager/system-connections/enp
enp1s0.nmconnection  enp2s0.nmconnection  
[root@master-0 ~]# cat /etc/NetworkManager/system-connections/enp2s0.nmconnection
[connection]
id=enp2s0
uuid=deb94e38-3c26-4f30-aefd-b73cfce6405a
type=ethernet
interface-name=enp2s0
permissions=
autoconnect=true
autoconnect-priority=1

[ethernet]
cloned-mac-address=52:54:00:76:70:00
mac-address-blacklist=

[ipv4]
address1=192.168.122.60/24
dhcp-client-id=mac
dns=192.168.3.80;
dns-priority=40
dns-search=
method=manual
route1=0.0.0.0/0,192.168.122.1
route1_options=table=254

[ipv6]
addr-gen-mode=eui64
dhcp-duid=ll
dhcp-iaid=mac
dns-search=
method=disabled

[proxy]
{% endhighlight %}
  
  
#### Conclusions
  
After going through this two mechanisms to deploy Single Node OpenShift with Static IPs one can notice that none is perfect nor straight forward, as this issue is still unsupported and under active development. However it is doable with some work and will eventually be simplified once the open RFEs are implemented around it.
  
That is all for now!
