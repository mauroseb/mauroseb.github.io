---
layout: posts
title: "OpenShift Container Storage 4.6 - External mode via CLI"
date: 2021-03-25
categories: [blog]
tags: [ ocp, kubernetes, ocs, odf ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---
### Intro

One of my customers requested me a way to deploy OpenShift Container Storage 4.6 (soon to be renamed to OpenShift Data Foundation) in external mode via CLI.
What is external mode you may ask ? OCS can provision storage to OpenShift by deploying its own in-cluster Ceph installation (internal mode) using in the background the Rook operator, or it can connect to an already existing Ceph cluster (external mode) where OCS mainly acts as Ceph client.

Despite there is no documented way to achieve this yet, we have some upstrem reference to deploying internal mode via CLI [^1], so we just need to add a little twist to make this work for external mode also.

To be noted, even though it is not straight away supported installation method it is technically the same result as the UI documented installation [^2].

In the following sections you can find the procedure I came up with.

<!-- more -->


### Environment Description

As usual I am doing all my testing in a KVM based virtual environment, where I have two networks:

  1. Where my OCP 4.7 cluster is running (192.168.133.0/24).
  2. Where my Ceph 4.1.5 public network lives (192.168.122.0/24).

Both networks are routable among each other.

<!--
The following diagram shows how the environment looks like.

<img src="/images/LAB_OCP.png" alt="OCP LAB PNG" style="width:75%;"/>
-->

It is important to remark that the minimum supported version of Ceph 4 to deploy ODF external mode is 4.1.2.

This is how the Ceph cluster looks like at this point in time:

{% highlight bash %}
[root@ice-bastion-01 ~]# ceph versions
{
    "mon": {
        "ceph version 14.2.11-95.el8cp (1d6087ae858e7c8e72fe7390c3522c7e0d951240) nautilus (stable)": 3
    },
    "mgr": {
        "ceph version 14.2.11-95.el8cp (1d6087ae858e7c8e72fe7390c3522c7e0d951240) nautilus (stable)": 1
    },
    "osd": {
        "ceph version 14.2.11-95.el8cp (1d6087ae858e7c8e72fe7390c3522c7e0d951240) nautilus (stable)": 3
    },
    "mds": {
        "ceph version 14.2.11-95.el8cp (1d6087ae858e7c8e72fe7390c3522c7e0d951240) nautilus (stable)": 1
    },
    "rgw": {
        "ceph version 14.2.11-95.el8cp (1d6087ae858e7c8e72fe7390c3522c7e0d951240) nautilus (stable)": 1
    },
    "overall": {
        "ceph version 14.2.11-95.el8cp (1d6087ae858e7c8e72fe7390c3522c7e0d951240) nautilus (stable)": 9
    }
}

[root@ice-bastion-01 ~]# ceph -s
  cluster:
    id:     6454b2b8-2cb4-495f-942c-1f1767b222ff
    health: HEALTH_WARN
            5 pools have too few placement groups
 
  services:
    mon: 3 daemons, quorum ice-extceph-01,ice-extceph-02,ice-extceph-03 (age 4w)
    mgr: ice-extceph-01(active, since 4w)
    mds: cephfs:1 {0=ice-extceph-02=up:active}
    osd: 3 osds: 3 up (since 4w), 3 in (since 11M)
    rgw: 1 daemon active (ice-extceph-03.rgw0)
 
  task status:
    scrub status:
        mds.ice-extceph-02: idle
 
  data:
    pools:   13 pools, 384 pgs
    objects: 11.07k objects, 41 GiB
    usage:   44 GiB used, 103 GiB / 147 GiB avail
    pgs:     384 active+clean
 
  io:
    client:   10 KiB/s wr, 0 op/s rd, 0 op/s wr
{% endhighlight %}

I will use the already created **volumes** pool for RBD access which will give the fastest RWO (ReadWriteOnce) capabilities out of the Ceph different access methods. Also having an active MDS (Metadata service) is a requirement for the ODF Operator to be able to create RWX (ReadWriteMany) persistent volumes. Finally one could use the RGW (Rados Gateway) service as a Noobaa backend to provide Object Bucket Claims (OBCs) which means that object storage could then be consumed directly from the pods.

Lets take a look at the volumes pool and confirm that application is set to RBD:
{% highlight bash %}
[root@ice-bastion-01 ~]# ceph osd pool ls detail | grep volumes
pool 8 'volumes' replicated size 1 min_size 1 crush_rule 0 object_hash rjenkins pg_num 64 pgp_num 64 autoscale_mode warn last_change 138 flags hashpspool,selfmanaged_snaps stripe_width 0 application rbd
{% endhighlight %}

**NOTE:** The replica size is set to 1 to spare some I/O in this lab environment but in a production environment will be typically 3 (or 2 for all-SSD pools).

### Install

All the files for the following procedure can be found in [ocp4-day2ops](https://github.com/mauroseb/ocp4-day2ops) repo under ocs folder.

#### 1. Create openshift-storage namespace with cluster monitoring label and operator group that will host the installed operators

{% highlight bash %}
# cat 01-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    openshift.io/cluster-monitoring: "true"
  name: openshift-storage
spec: {}

# oc create -f 01-namespace.yaml
namespace/openshift-storage created

# cat 02-storage-operator-group.yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-storage-operatorgroup
  namespace: openshift-storage
spec:
  targetNamespaces:
  - openshift-storage

# oc create -f 02-storage-operator-group.yaml
operatorgroup.operators.coreos.com/openshift-storage-operatorgroup created

{% endhighlight %}


#### 2. Install the OCS operator in the brand new namespace

{% highlight bash %}
# cat 03-storage-operator-subscription.yaml 
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ocs-operator
  namespace: openshift-storage
spec:
  channel: "stable-4.6"
  installPlanApproval: Automatic
  name: ocs-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace

# oc create -f 03-storage-operator-subscription.yaml
subscription.operators.coreos.com/ocs-operator created

{% endhighlight %}

#### 3. Check the installation progress until it finishes

{% highlight bash %}
# oc get csv -A
NAMESPACE                              NAME                  DISPLAY                       VERSION   REPLACES   PHASE
openshift-operator-lifecycle-manager   packageserver         Package Server                0.17.0               Succeeded
openshift-storage                      ocs-operator.v4.6.4   OpenShift Container Storage   4.6.4                Installing
{% endhighlight %}

 - After 5 minutes or so it will move from __Installing__ to __Succeeded__
{% highlight bash %}
# oc get csv -A
NAMESPACE                              NAME                  DISPLAY                       VERSION   REPLACES   PHASE
openshift-operator-lifecycle-manager   packageserver         Package Server                0.17.0               Succeeded
openshift-storage                      ocs-operator.v4.6.4   OpenShift Container Storage   4.6.4                Succeeded
{% endhighlight %}


#### 4. Create a secret to store the external cluster details

 - When run through the UI, this step will ask you to download a python script called  __ceph-external-cluster-details-exporter.py__ and execute it in your Ceph deployment bastion host or a MON. Then you are instructed to upload the json output to the Operator installation form. What happens in the background is that the Operator creates a secret holding this file as a data key with the Ceph cluster details, and then creates a __StorageCluster__ custom resource stating it is external.

 - First create the JSON file in question
{% highlight bash %}
[root@ice-bastion-01 ~]# python3 ceph-external-cluster-details-exporter.py --rgw-endpoint 192.168.122.13:80 --monitoring-endpoint 192.168.122.11 --rbd-data-pool-name volumes
[{"name": "rook-ceph-mon-endpoints", "kind": "ConfigMap", "data": {"data": "ice-extceph-01=192.168.122.11:6789", "maxMonId": "0", "mapping": "{}"}}, {"name": "rook-ceph-mon", "kind": "Secret", "data": {"admin-secret": "admin-secret", "fsid": "6454b2b8-2cb4-495f-942c-1f1767b222ff", "mon-secret": "mon-secret"}}, {"name": "rook-ceph-operator-creds", "kind": "Secret", "data": {"userID": "client.healthchecker", "userKey": "AQCekEFgkL9uGRAA2yrG+uss2psoVtzPSoee0Q=="}}, {"name": "rook-csi-rbd-node", "kind": "Secret", "data": {"userID": "csi-rbd-node", "userKey": "AQCekEFg/81EHRAAIb/cTfjwyuE89O9QDbirSA=="}}, {"name": "ceph-rbd", "kind": "StorageClass", "data": {"pool": "volumes"}}, {"name": "monitoring-endpoint", "kind": "CephCluster", "data": {"MonitoringEndpoint": "192.168.122.11", "MonitoringPort": "9283"}}, {"name": "rook-csi-rbd-provisioner", "kind": "Secret", "data": {"userID": "csi-rbd-provisioner", "userKey": "AQCekEFgWupBIhAAk0DIALOmf3N8RU+EabtRMw=="}}, {"name": "rook-csi-cephfs-provisioner", "kind": "Secret", "data": {"adminID": "csi-cephfs-provisioner", "adminKey": "AQCekEFgAgIzMhAAUnE0IHx5xWinfVqjCsL3mg=="}}, {"name": "rook-csi-cephfs-node", "kind": "Secret", "data": {"adminID": "csi-cephfs-node", "adminKey": "AQCekEFgYiLZKRAABNX85LZP7aHetj9ymNwhRg=="}}, {"name": "cephfs", "kind": "StorageClass", "data": {"fsName": "cephfs", "pool": "cephfs_data"}}, {"name": "ceph-rgw", "kind": "StorageClass", "data": {"endpoint": "192.168.122.13:80", "poolPrefix": "default"}}]
{% endhighlight %}

 - And now create a secret with the following characteristics: a. data key has to be __external_cluster_details__, b. has to contain the BASE64 encoded JSON file

{% highlight bash %}
# cat 04-external-ocs-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: rook-ceph-external-cluster-details
  namespace: openshift-storage
data:
  external_cluster_details: W3sibmFtZSI6ICJyb29rLWNlcGgtbW9uLWVuZHBvaW50cyIsICJraW5kIjogIkNvbmZpZ01hcCIsICJkYXRhIjogeyJkYXRhIjogImljZS1leHRjZXBoLTAxPTE5Mi4xNjguMTIyLjExOjY3ODkiLCAibWF4TW9uSWQiOiAiMCIsICJtYXBwaW5nIjogInt9In19LCB7Im5hbWUiOiAicm9vay1jZXBoLW1vbiIsICJraW5kIjogIlNlY3JldCIsICJkYXRhIjogeyJhZG1pbi1zZWNyZXQiOiAiYWRtaW4tc2VjcmV0IiwgImZzaWQiOiAiNjQ1NGIyYjgtMmNiNC00OTVmLTk0MmMtMWYxNzY3YjIyMmZmIiwgIm1vbi1zZWNyZXQiOiAibW9uLXNlY3JldCJ9fSwgeyJuYW1lIjogInJvb2stY2VwaC1vcGVyYXRvci1jcmVkcyIsICJraW5kIjogIlNlY3JldCIsICJkYXRhIjogeyJ1c2VySUQiOiAiY2xpZW50LmhlYWx0aGNoZWNrZXIiLCAidXNlcktleSI6ICJBUUNla0VGZ2tMOXVHUkFBMnlyRyt1c3MycHNvVnR6UFNvZWUwUT09In19LCB7Im5hbWUiOiAicm9vay1jc2ktcmJkLW5vZGUiLCAia2luZCI6ICJTZWNyZXQiLCAiZGF0YSI6IHsidXNlcklEIjogImNzaS1yYmQtbm9kZSIsICJ1c2VyS2V5IjogIkFRQ2VrRUZnLzgxRUhSQUFJYi9jVGZqd3l1RTg5TzlRRGJpclNBPT0ifX0sIHsibmFtZSI6ICJjZXBoLXJiZCIsICJraW5kIjogIlN0b3JhZ2VDbGFzcyIsICJkYXRhIjogeyJwb29sIjogInZvbHVtZXMifX0sIHsibmFtZSI6ICJtb25pdG9yaW5nLWVuZHBvaW50IiwgImtpbmQiOiAiQ2VwaENsdXN0ZXIiLCAiZGF0YSI6IHsiTW9uaXRvcmluZ0VuZHBvaW50IjogIjE5Mi4xNjguMTIyLjExIiwgIk1vbml0b3JpbmdQb3J0IjogIjkyODMifX0sIHsibmFtZSI6ICJyb29rLWNzaS1yYmQtcHJvdmlzaW9uZXIiLCAia2luZCI6ICJTZWNyZXQiLCAiZGF0YSI6IHsidXNlcklEIjogImNzaS1yYmQtcHJvdmlzaW9uZXIiLCAidXNlcktleSI6ICJBUUNla0VGZ1d1cEJJaEFBazBESUFMT21mM044UlUrRWFidFJNdz09In19LCB7Im5hbWUiOiAicm9vay1jc2ktY2VwaGZzLXByb3Zpc2lvbmVyIiwgImtpbmQiOiAiU2VjcmV0IiwgImRhdGEiOiB7ImFkbWluSUQiOiAiY3NpLWNlcGhmcy1wcm92aXNpb25lciIsICJhZG1pbktleSI6ICJBUUNla0VGZ0FnSXpNaEFBVW5FMElIeDV4V2luZlZxakNzTDNtZz09In19LCB7Im5hbWUiOiAicm9vay1jc2ktY2VwaGZzLW5vZGUiLCAia2luZCI6ICJTZWNyZXQiLCAiZGF0YSI6IHsiYWRtaW5JRCI6ICJjc2ktY2VwaGZzLW5vZGUiLCAiYWRtaW5LZXkiOiAiQVFDZWtFRmdZaUxaS1JBQUJOWDg1TFpQN2FIZXRqOXltTndoUmc9PSJ9fSwgeyJuYW1lIjogImNlcGhmcyIsICJraW5kIjogIlN0b3JhZ2VDbGFzcyIsICJkYXRhIjogeyJmc05hbWUiOiAiY2VwaGZzIiwgInBvb2wiOiAiY2VwaGZzX2RhdGEifX0sIHsibmFtZSI6ICJjZXBoLXJndyIsICJraW5kIjogIlN0b3JhZ2VDbGFzcyIsICJkYXRhIjogeyJlbmRwb2ludCI6ICIxOTIuMTY4LjEyMi4xMzo4MCIsICJwb29sUHJlZml4IjogImRlZmF1bHQifX1dCg==
type: Opaque

# oc create -f 04-external-ocs-secret.yaml
secret/rook-ceph-external-cluster-details created
{% endhighlight %}

#### 5. Finally create the StorageCluster resource specifying that this is an external cluster 

 - That is achieved by specifying __spec.externalStorage.enable__ true and the channel in this case 4.6.0, which will pick the latest 4.6 version available.

{% highlight bash %}
# cat 05-storage-cluster.yaml
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-external-storagecluster
  namespace: openshift-storage
spec:
  encryption: {}
  externalStorage:
    enable: true
  labelSelector: {}
  managedResources:
    cephBlockPools: {}
    cephFilesystems: {}
    cephObjectStoreUsers: {}
    cephObjectStores: {}
  version: 4.6.0

# oc create -f 05-storage-cluster.yaml
storagecluster.ocs.openshift.io/ocs-external-storagecluster created
{% endhighlight %}


 - Now we can check the cluster resource
{% highlight bash %}
# oc get StorageCluster ocs-external-storagecluster -o yaml -n openshift-storage
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  annotations:
    uninstall.ocs.openshift.io/cleanup-policy: delete
    uninstall.ocs.openshift.io/mode: graceful
  creationTimestamp: "2021-04-16T16:39:16Z"
  finalizers:
  - storagecluster.ocs.openshift.io
  generation: 1
  managedFields:
  - apiVersion: ocs.openshift.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        .: {}
        f:encryption: {}
        f:externalStorage:
          .: {}
          f:enable: {}
        f:labelSelector: {}
        f:managedResources:
          .: {}
          f:cephBlockPools: {}
          f:cephFilesystems: {}
          f:cephObjectStoreUsers: {}
          f:cephObjectStores: {}
        f:version: {}
    manager: kubectl-create
    operation: Update
    time: "2021-04-16T16:39:16Z"
  - apiVersion: ocs.openshift.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:uninstall.ocs.openshift.io/cleanup-policy: {}
          f:uninstall.ocs.openshift.io/mode: {}
        f:finalizers: {}
      f:status:
        .: {}
        f:conditions: {}
        f:externalSecretHash: {}
        f:phase: {}
        f:relatedObjects: {}
    manager: ocs-operator
    operation: Update
    time: "2021-04-16T16:39:18Z"
  name: ocs-external-storagecluster
  namespace: openshift-storage
  resourceVersion: "1250535"
  selfLink: /apis/ocs.openshift.io/v1/namespaces/openshift-storage/storageclusters/ocs-external-storagecluster
  uid: af9abcef-70a0-4168-bcbc-b3dfd6ab80c2
spec:
  encryption: {}
  externalStorage:
    enable: true
  labelSelector: {}
  managedResources:
    cephBlockPools: {}
    cephFilesystems: {}
    cephObjectStoreUsers: {}
    cephObjectStores: {}
  version: 4.6.0
status:
  conditions:
  - lastHeartbeatTime: "2021-04-16T16:39:32Z"
    lastTransitionTime: "2021-04-16T16:39:19Z"
    message: Reconcile completed successfully
    reason: ReconcileCompleted
    status: "True"
    type: ReconcileComplete
  - lastHeartbeatTime: "2021-04-16T16:39:20Z"
    lastTransitionTime: "2021-04-16T16:39:20Z"
    message: CephCluster resource is not reporting status
    reason: CephClusterStatus
    status: "False"
    type: Available
  - lastHeartbeatTime: "2021-04-16T16:39:32Z"
    lastTransitionTime: "2021-04-16T16:39:20Z"
    message: Waiting on Nooba instance to finish initialization
    reason: NoobaaInitializing
    status: "True"
    type: Progressing
  - lastHeartbeatTime: "2021-04-16T16:39:19Z"
    lastTransitionTime: "2021-04-16T16:39:18Z"
    message: Reconcile completed successfully
    reason: ReconcileCompleted
    status: "False"
    type: Degraded
  - lastHeartbeatTime: "2021-04-16T16:39:20Z"
    lastTransitionTime: "2021-04-16T16:39:20Z"
    message: CephCluster resource is not reporting status
    reason: CephClusterStatus
    status: "False"
    type: Upgradeable
  - lastHeartbeatTime: "2021-04-16T16:39:30Z"
    lastTransitionTime: "2021-04-16T16:39:22Z"
    message: 'External CephCluster is trying to connect: Cluster is connecting'
    reason: ExternalClusterStateConnecting
    status: "True"
    type: ExternalClusterConnecting
  - lastHeartbeatTime: "2021-04-16T16:39:30Z"
    lastTransitionTime: "2021-04-16T16:39:22Z"
    message: 'External CephCluster is trying to connect: Cluster is connecting'
    reason: ExternalClusterStateConnecting
    status: "False"
    type: ExternalClusterConnected
  externalSecretHash: fed8ce5ab8a4e18e80e21774a7d012ea245e963696080cb04ae818cdbbc17485766ba1fead16496d1f3f12d413d0de0bce03fe6493768fc0bf28ba100ee8e7a2
  phase: Ready
  relatedObjects:
  - apiVersion: ceph.rook.io/v1
    kind: CephCluster
    name: ocs-external-storagecluster-cephcluster
    namespace: openshift-storage
    resourceVersion: "1250518"
    uid: 29c09d76-a4d7-481c-af5e-49d623dd34b1
  - apiVersion: noobaa.io/v1alpha1
    kind: NooBaa
    name: noobaa
    namespace: openshift-storage
    resourceVersion: "1250534"
    uid: 3aa4f603-3dc0-41b3-b518-086b8b77fb42
{% endhighlight %}

#### 6. Verifying the Installation

 - After a while we can also check the resources created in the namespace

{% highlight bash %}
# oc get all
NAME                                                READY   STATUS    RESTARTS   AGE
pod/csi-cephfsplugin-n75hj                          3/3     Running   0          14m
pod/csi-cephfsplugin-provisioner-66c59d467f-k2bvm   6/6     Running   0          14m
pod/csi-cephfsplugin-provisioner-66c59d467f-l8l22   6/6     Running   0          14m
pod/csi-cephfsplugin-sw9lv                          3/3     Running   0          14m
pod/csi-rbdplugin-7bvqv                             3/3     Running   0          14m
pod/csi-rbdplugin-provisioner-6b7dcf968-bxpxn       6/6     Running   0          14m
pod/csi-rbdplugin-provisioner-6b7dcf968-m9xk6       6/6     Running   0          14m
pod/csi-rbdplugin-zwtm7                             3/3     Running   0          14m
pod/noobaa-core-0                                   1/1     Running   1          14m
pod/noobaa-db-0                                     1/1     Running   0          14m
pod/noobaa-operator-7cc4795d56-whqst                1/1     Running   0          20m
pod/ocs-metrics-exporter-9b6f9786c-vbj9w            1/1     Running   0          20m
pod/ocs-operator-7b6b9dd749-7mqrp                   0/1     Running   0          20m
pod/rook-ceph-operator-6d454c4c58-7f7bw             1/1     Running   0          20m

NAME                                                                TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                    AGE
service/csi-cephfsplugin-metrics                                    ClusterIP      172.30.179.208   <none>        8080/TCP,8081/TCP                                          14m
service/csi-rbdplugin-metrics                                       ClusterIP      172.30.103.207   <none>        8080/TCP,8081/TCP                                          14m
service/noobaa-db                                                   ClusterIP      172.30.231.62    <none>        27017/TCP                                                  14m
service/noobaa-mgmt                                                 LoadBalancer   172.30.6.83      <pending>     80:30027/TCP,443:31525/TCP,8445:31211/TCP,8446:30772/TCP   14m
service/ocs-metrics-exporter                                        ClusterIP      172.30.153.173   <none>        8080/TCP,8081/TCP                                          14m
service/rook-ceph-mgr-external                                      ClusterIP      172.30.189.177   <none>        9283/TCP                                                   14m
service/rook-ceph-rgw-ocs-external-storagecluster-cephobjectstore   ClusterIP      172.30.83.139    <none>        80/TCP                                                     13m
service/s3                                                          LoadBalancer   172.30.7.93      <pending>     80:31156/TCP,443:30464/TCP,8444:31400/TCP                  14m

NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/csi-cephfsplugin   2         2         2       2            2           <none>          14m
daemonset.apps/csi-rbdplugin      2         2         2       2            2           <none>          14m

NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/csi-cephfsplugin-provisioner   2/2     2            2           14m
deployment.apps/csi-rbdplugin-provisioner      2/2     2            2           14m
deployment.apps/noobaa-operator                1/1     1            1           20m
deployment.apps/ocs-metrics-exporter           1/1     1            1           20m
deployment.apps/ocs-operator                   0/1     1            0           20m
deployment.apps/rook-ceph-operator             1/1     1            1           20m

NAME                                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/csi-cephfsplugin-provisioner-66c59d467f   2         2         2       14m
replicaset.apps/csi-rbdplugin-provisioner-6b7dcf968       2         2         2       14m
replicaset.apps/noobaa-operator-7cc4795d56                1         1         1       20m
replicaset.apps/ocs-metrics-exporter-9b6f9786c            1         1         1       20m
replicaset.apps/ocs-operator-7b6b9dd749                   1         1         0       20m
replicaset.apps/rook-ceph-operator-6d454c4c58             1         1         1       20m

NAME                           READY   AGE
statefulset.apps/noobaa-core   1/1     14m
statefulset.apps/noobaa-db     1/1     14m

NAME                                   HOST/PORT                                                 PATH   SERVICES      PORT         TERMINATION          WILDCARD
route.route.openshift.io/noobaa-mgmt   noobaa-mgmt-openshift-storage.apps.ocp4-bm.t1.lab.local          noobaa-mgmt   mgmt-https   reencrypt/Redirect   None
route.route.openshift.io/s3            s3-openshift-storage.apps.ocp4-bm.t1.lab.local                   s3            s3-https     reencrypt            None

{% endhighlight %}


**NOTE:** The ocs-operator container started and failed the readiness probe, and after restarting the pod it corrected. This seems to be caused by the rediness probe being triggered before the deployment is fully finished.

{% highlight bash %}
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-operator-6d454c4c58-7f7bw   1/1     Running   0          11d
{% endhighlight %}

 - The OCS storage classes are now avaialble

{% highlight bash %}
oc get sc
NAME                                   PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
ocs-external-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      Delete          Immediate           true                   13d
ocs-external-storagecluster-ceph-rgw   openshift-storage.ceph.rook.io/bucket   Delete          Immediate           false                  13d
ocs-external-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate           true                   13d
openshift-storage.noobaa.io            openshift-storage.noobaa.io/obc         Delete          Immediate           false                  13d
{% endhighlight %}

### Final Thoughts

When I create the JSON file a single MON/MGR IP is passed to the python script however the operator will take care from there to inspect the external Cluster and populate the 3 MONs and thus getting rid of the potential SPOC between OCP and Ceph.

Moreover in case we add, replace or remove a MON, the operator will take care of updating the internal information automatically.

### References

 [^1]: [Install OCS without the OpenShift Web UI](https://github.com/red-hat-storage/ocs-training/blob/master/training/modules/ocs4/pages/ocs4-install-no-ui.adoc)
 
 [^2]: [OpenShift Container Storage in External Mode](https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.6/html-single/deploying_openshift_container_storage_in_external_mode/index)

