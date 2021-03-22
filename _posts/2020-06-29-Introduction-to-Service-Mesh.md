---
layout: posts
title: "Introduction to OpenShift Service Mesh"
date: 2020-06-29
categories: [blog]
tags: [ ocp, kubernetes, istio, ossm, servicemesh ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---
### What's that ? 

After moving from a monolithic application to a microservice oriented architecture, what previously was an RPC call within the application itself now has become a request that has to travel on the network where the microservices are running on, which creates security concerns and also increases the network usage considerably. In addition to that, multi-tenant environments also expose this networks to multiple consumers.

Moreover microservices architectures are often design to let the microservice itself take care of things like locating the services it needs to interact with, routing between different services, routing traffic between different api version (even asymetrically), decoupling load balancing from infrastructure scaling, and many other. All these aspects are a common factor in most microservices, which makes their code bloated an unnecesarily repeated.
<!-- more -->

A service mesh aims to provide a solution to the problems described by providing an enriched control plane. Provides for instance encrypted communication for inter pod traffic, micro segmentation to only permit communication within certain projects or pods, routing, service to service authentication, load-balancing. rate-limiting, monitoring and more.

The service mesh will create a service container for every application pod is added to the cluster and therefore extra resources may be required. Furthermore encryption also takes its toll on traffic. It means that in some cases a service mesh is not the best fit. For example, if one could consider the cluster network as properly isolated and trusted then the benifits of encryption may be not justfied by the extra cost.

Red Hat OpenShift Container Platform 4 integrates relies on Istio upstream project as its service mesh solution.

### Where is the documentation about that?

Good places to start learning about service mesh:

 - <https://docs.openshift.com/container-platform/4.4/service_mesh/service_mesh_arch/understanding-ossm.html>
 - <https://istio.io/latest/docs/concepts/what-is-istio/>
 - <https://istio.io/latest/docs/concepts/traffic-management/>
 - <https://learn.openshift.com/servicemesh> (interactive)
 - Here!

Now lets get to work.

### Prerequisites

 1. Assume you have a working OCP4 cluster and you are cluster admin.

{% highlight console %}
$ oc whoami
system:admin

$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.3.3     True        False         13h     Cluster version is 4.3.3

$ oc get nodes
NAME                                         STATUS   ROLES    AGE     VERSION
ip-10-0-128-75.eu-west-1.compute.internal    Ready    master   3h51m   v1.16.2
ip-10-0-143-141.eu-west-1.compute.internal   Ready    worker   3h39m   v1.16.2
ip-10-0-146-42.eu-west-1.compute.internal    Ready    worker   3h39m   v1.16.2

$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.3.3     True        False         False      12h
cloud-credential                           4.3.3     True        False         False      12h
cluster-autoscaler                         4.3.3     True        False         False      12h
console                                    4.3.3     True        False         False      12h
dns                                        4.3.3     True        False         False      12h
image-registry                             4.3.3     True        False         False      12h
ingress                                    4.3.3     True        False         False      3m45s
insights                                   4.3.3     True        False         False      12h
kube-apiserver                             4.3.3     True        False         False      12h
kube-controller-manager                    4.3.3     True        False         False      12h
kube-scheduler                             4.3.3     True        False         False      12h
machine-api                                4.3.3     True        False         False      12h
machine-config                             4.3.3     True        False         False      12h
marketplace                                4.3.3     True        False         False      3m33s
monitoring                                 4.3.3     True        False         False      3m1s
network                                    4.3.3     True        False         False      12h
node-tuning                                4.3.3     True        False         False      3m52s
openshift-apiserver                        4.3.3     True        False         False      12h
openshift-controller-manager               4.3.3     True        False         False      12h
openshift-samples                          4.3.3     True        False         False      12h
operator-lifecycle-manager                 4.3.3     True        False         False      12h
operator-lifecycle-manager-catalog         4.3.3     True        False         False      12h
operator-lifecycle-manager-packageserver   4.3.3     True        False         False      3m14s
service-ca                                 4.3.3     True        False         False      12h
service-catalog-apiserver                  4.3.3     True        False         False      12h
service-catalog-controller-manager         4.3.3     True        False         False      12h
storage                                    4.3.3     True        False         False      12h
{% endhighlight %}


### Install

The standard way to install Service Mesh is through its operator. But first one also has to install the operators that correspond to its dependencies: 
  - __Elasticsearch__
  - __Jaeger:__ Profiling component of the service mesh implementing the OpenTracing standard.
  - __Kiali:__ The WebUI to visualize the service mesh.

#### Elasticsearch

1. In the web UI, OperatorHub tab, search for 'elastic', and install the operator as show below.


2. Check that the installation is successful:

{% highlight console %}
$ oc get csv
NAME                                         DISPLAY                  VERSION               REPLACES   PHASE
elasticsearch-operator.4.1.41-202004130646   Elasticsearch Operator   4.1.41-202004130646              Succeeded

$ oc get pod -n openshift-operators | grep elastic
elasticsearch-operator-57cbcc6565-n9ts5   1/1     Running   0          4m59s
{% endhighlight %}

#### Jaeger

1. In the web UI, OperatorHub tab, search for 'jaeger', and install the operator as show below.


2. Check that the installation is successful:

{% highlight console %}
$ oc get csv
NAME                                         DISPLAY                    VERSION               REPLACES   PHASE
elasticsearch-operator.4.1.41-202004130646   Elasticsearch Operator     4.1.41-202004130646              Succeeded
jaeger-operator.v1.17.3                      Red Hat OpenShift Jaeger   1.17.3                           Succeeded

$ oc get pod -n openshift-operators | grep jae
jaeger-operator-ff8cccbd8-j67jc           1/1     Running   0          2m42s
{% endhighlight %}


#### Kiali

1. In the web UI, OperatorHub tab, search for 'jaeger', and install the operator as show below.


2. Check that the installation is successful:

{% highlight console %}
$ oc get csv
NAME                                         DISPLAY                    VERSION               REPLACES                  PHASE
elasticsearch-operator.4.1.41-202004130646   Elasticsearch Operator     4.1.41-202004130646                             Succeeded
jaeger-operator.v1.17.3                      Red Hat OpenShift Jaeger   1.17.3                                          Succeeded
kiali-operator.v1.12.13                      Kiali Operator             1.12.13               kiali-operator.v1.12.12   

$ oc get pod -n openshift-operators | grep kia
kiali-operator-69bffd8cd4-nsmcj           2/2     Running   0          59s
{% endhighlight %}


#### Install Service Mesh Operator

At this moment is time to proceed with the Service Mesh Operator itself.

{% highlight console %}
$ oc adm new-project istio-operator --display-name="Service Mesh Operator"
Created project istio-operator

$ oc project istio-operator
Now using project "istio-operator" on server "https://api.cluster-f2a8.f2a8.sandbox663.opentlc.com:6443".

$ oc apply -n istio-operator -f https://raw.githubusercontent.com/Maistra/istio-operator/maistra-1.0.0/deploy/servicemesh-operator.yaml
customresourcedefinition.apiextensions.k8s.io/servicemeshcontrolplanes.maistra.io created
customresourcedefinition.apiextensions.k8s.io/servicemeshmemberrolls.maistra.io created
clusterrole.rbac.authorization.k8s.io/maistra-admin created
clusterrolebinding.rbac.authorization.k8s.io/maistra-admin created
clusterrole.rbac.authorization.k8s.io/istio-operator created
serviceaccount/istio-operator created
clusterrolebinding.rbac.authorization.k8s.io/istio-operator-account-istio-operator-cluster-role-binding created
service/admission-controller created
deployment.apps/istio-operator created

$ oc get po -o wide -n istio-operator
NAME                              READY   STATUS    RESTARTS   AGE    IP            NODE                                         NOMINATED NODE   READINESS GATES
istio-operator-568849cc4f-zw4nh   1/1     Running   0          2m4s   10.130.0.16   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>
{% endhighlight %}


#### Install Service Mesh Control Plane

1. Create SM control plane project

{% highlight console %}
$ oc adm new-project istio-system --display-name="Service Mesh System"
Created project istio-system
{% endhighlight %}

{:start="2"}
2. Create a CRD definition like the follwoing:

{% highlight console %}
apiVersion: maistra.io/v1
kind: ServiceMeshControlPlane
metadata:
  name: service-mesh-installation
spec:
  threeScale:
    enabled: false

  istio:
    global:
      mtls: false
      disablePolicyChecks: false
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 128Mi

    gateways:
      istio-egressgateway:
        autoscaleEnabled: false
      istio-ingressgateway:
        autoscaleEnabled: false
        ior_enabled: false

    mixer:
      policy:
        autoscaleEnabled: false

      telemetry:
        autoscaleEnabled: false
        resources:
          requests:
            cpu: 100m
            memory: 1G
          limits:
            cpu: 500m
            memory: 4G

    pilot:
      autoscaleEnabled: false
      traceSampling: 100.0

    kiali:
      dashboard:
        user: admin
        passphrase: redhat
    tracing:
      enabled: true

$ oc apply -f service-mesh.yaml -n istio-system
servicemeshcontrolplane.maistra.io/service-mesh-installation created
{% endhighlight %}

Wait for containers to come up from 5 to 10 minutes.

{% highlight console %}
$  oc get po -o wide -n istio-system
NAME                                      READY   STATUS    RESTARTS   AGE     IP            NODE                                         NOMINATED NODE   READINESS GATES
grafana-798f5c4695-7rcf2                  2/2     Running   0          6m33s   10.130.0.23   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>
istio-citadel-749b5f749-znm9d             1/1     Running   0          10m     10.129.0.17   ip-10-0-146-42.eu-west-1.compute.internal    <none>           <none>
istio-egressgateway-7467b4d5-z9fqf        1/1     Running   0          7m33s   10.130.0.20   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>
istio-galley-87959d49b-rfsp9              1/1     Running   0          9m56s   10.130.0.18   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>
istio-ingressgateway-6f69c9bd-zhgct       1/1     Running   0          7m33s   10.130.0.21   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>
istio-pilot-766fbd856-ckjn9               2/2     Running   0          8m29s   10.129.0.20   ip-10-0-146-42.eu-west-1.compute.internal    <none>           <none>
istio-policy-5bd4648bd9-j4bfl             2/2     Running   0          9m29s   10.130.0.19   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>
istio-sidecar-injector-75748d7d88-z4w4m   1/1     Running   0          7m2s    10.130.0.22   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>
istio-telemetry-57d749b69-6nns6           2/2     Running   0          9m28s   10.129.0.19   ip-10-0-146-42.eu-west-1.compute.internal    <none>           <none>
jaeger-5f4666947c-rxbr4                   2/2     Running   0          10m     10.129.0.18   ip-10-0-146-42.eu-west-1.compute.internal    <none>           <none>
kiali-64cd8b9bbc-dpkhb                    1/1     Running   0          5m35s   10.130.0.24   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>
prometheus-7ddcc755fb-q9n6l               2/2     Running   0          10m     10.130.0.17   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>
{% endhighlight %}

{:start="3"}
3. Create ServiceMeshMemberRoll with the projects which need to take part of the service mesh

{% highlight console %}
$ echo "apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
spec:
  members:
  - coolapp
" > $HOME/service-mesh-roll.yaml

$ oc apply -f service-mesh-roll.yaml -n istio-system
servicemeshmemberroll.maistra.io/default created

%%%
$ oc adm policy add-role-to-user edit user1 -n istio-system
clusterrole.rbac.authorization.k8s.io/edit added: "user1"
...
{% endhighlight %}

### Deploying Test Workload

To test service mesh I am using a set of microservices that were created for training purposes (gtihub.com/gpe-mw-training/ocp-service-mesh-foundations) and I cloned into my own repository.

These 3 services are "gateway", "partner" and "catalog" and they should interact in the following manner.

[gateway] => [partner] => [catalog]

The service mesh component Pilot will create graph modeling the discovered microservices.

From previous section I have included **coolapp** namespace in the service mesh member roll definition:

{% highlight console %}
$ oc get servicemeshmemberrolls.maistra.io
NAME      MEMBERS
default   [coolapp]

$ oc get servicemeshmemberrolls.maistra.io  -o yaml
apiVersion: v1
items:
- apiVersion: maistra.io/v1
  kind: ServiceMeshMemberRoll
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"maistra.io/v1","kind":"ServiceMeshMemberRoll","metadata":{"annotations":{},"name":"default","namespace":"istio-system"},"spec":{"members":["coolapp"]}}
    creationTimestamp: "2020-06-24T00:42:13Z"
    finalizers:
    - maistra.io/istio-operator
    generation: 2
    name: default
    namespace: istio-system
    ownerReferences:
    - apiVersion: maistra.io/v1
      kind: ServiceMeshControlPlane
      name: service-mesh-installation
      uid: 7c38cdc8-1d1b-4eae-a8f3-51191b2e06c7
    resourceVersion: "496392"
    selfLink: /apis/maistra.io/v1/namespaces/istio-system/servicemeshmemberrolls/default
    uid: 3f6ca692-6a6a-47fb-97f3-7dd2e9d90427
  spec:
    members:
    - coolapp
  status:
    meshGeneration: 1
    observedGeneration: 2
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
{% endhighlight %}

 - Create the coolapp namespace to deploy the microservices

{% highlight console %}
$ oc new-project coolapp
Now using project "coolapp" on server "https://api.cluster-f2a8.f2a8.sandbox663.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

{% endhighlight %}

 - Create the catalog microservice

{% highlight console %}
$ oc create -f lab/ocp-service-mesh-foundations/catalog/kubernetes/catalog-service-template.yml -n coolapp
deployment.extensions/catalog-v1 created

$ oc create -f ~/lab/ocp-service-mesh-foundations/catalog/kubernetes/Service.yml -n coolapp
service/catalog created

$ oc get all -n coolapp
NAME                              READY   STATUS    RESTARTS   AGE
pod/catalog-v1-6d6b478456-9vsm2   2/2     Running   0          95s

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/catalog   ClusterIP   172.30.72.240   <none>        8080/TCP   3s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/catalog-v1   1/1     1            1           95s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/catalog-v1-6d6b478456   1         1         1       95s
[moddi-redhat.com@clientvm 0 ~]$ oc get pods -w
NAME                          READY   STATUS    RESTARTS   AGE
catalog-v1-6d6b478456-9vsm2   2/2     Running   0          2m5s
{% endhighlight %}

 - Note that we have two containers for the catalog pod

{% highlight console %}
$ oc get pod/catalog-v1-6d6b478456-9vsm2  -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: istio-cni
    k8s.v1.cni.cncf.io/networks-status: |-
      [{
          "name": "openshift-sdn",
          "interface": "eth0",
          "ips": [
              "10.130.0.32"
          ],
          "dns": {},
          "default-route": [
              "10.130.0.1"
          ]
      },{
          "name": "istio-cni",
          "dns": {}
      }]
    openshift.io/scc: restricted
    sidecar.istio.io/inject: "true"
    sidecar.istio.io/status: '{"version":"1482941f6bcd31c05ffef23cc131fe8686c0f23452a318bdd16c547d8993ea19","annotations":{"k8s.v1.cni.cncf.io/networks":"istio-cni"},"initContainers":null,"containers":["istio-proxy"],"volumes":["istio-envoy","istio-certs"],"imagePullSecrets":null}'
  creationTimestamp: "2020-06-24T16:44:10Z"
  generateName: catalog-v1-6d6b478456-
  labels:
    app: catalog
    application: catalog
    deployment: catalog
    failure-domain.beta.kubernetes.io/region: eu-west-1
    failure-domain.beta.kubernetes.io/zone: eu-west-1a
    pod-template-hash: 6d6b478456
    version: v1
  name: catalog-v1-6d6b478456-9vsm2
  namespace: coolapp
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: catalog-v1-6d6b478456
    uid: 55819e8d-3137-47b1-8bfc-a65e111d63b5
  resourceVersion: "498102"
  selfLink: /api/v1/namespaces/coolapp/pods/catalog-v1-6d6b478456-9vsm2
  uid: 623a9e8f-21b8-4316-a13b-9e27a6d40841
spec:
  containers:
  - env:
    - name: KUBERNETES_NAMESPACE
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.namespace
    image: docker.io/rhtgptetraining/foundation-catalog-service:1.0.0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 2
      httpGet:
        path: /health
        port: 8080
        scheme: HTTP
      initialDelaySeconds: 60
      periodSeconds: 10
      successThreshold: 1
      timeoutSeconds: 1
    name: catalog
    ports:
    - containerPort: 8080
      protocol: TCP
    readinessProbe:
      failureThreshold: 3
      httpGet:
        path: /health
        port: 8080
        scheme: HTTP
      initialDelaySeconds: 20
      periodSeconds: 10
      successThreshold: 1
      timeoutSeconds: 1
    resources:
      limits:
        cpu: 250m
        memory: 500Mi
      requests:
        cpu: 125m
        memory: 500Mi
    securityContext:
      capabilities:
        drop:
        - KILL
        - MKNOD
        - SETGID
        - SETUID
      privileged: false
      runAsUser: 1000580000
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-rsww6
      readOnly: true
  - args:
    - proxy
    - sidecar
    - --domain
    - $(POD_NAMESPACE).svc.cluster.local
    - --configPath
    - /etc/istio/proxy
    - --binaryPath
    - /usr/local/bin/envoy
    - --serviceCluster
    - catalog.$(POD_NAMESPACE)
    - --drainDuration
    - 45s
    - --parentShutdownDuration
    - 1m0s
    - --discoveryAddress
    - istio-pilot.istio-system:15010
    - --zipkinAddress
    - zipkin.istio-system:9411
    - --connectTimeout
    - 10s
    - --proxyAdminPort
    - "15000"
    - --concurrency
    - "2"
    - --controlPlaneAuthPolicy
    - NONE
    - --statusPort
    - "15020"
    - --applicationPorts
    - "8080"
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.namespace
    - name: INSTANCE_IP
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: status.podIP
    - name: ISTIO_META_POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    - name: ISTIO_META_CONFIG_NAMESPACE
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.namespace
    - name: ISTIO_META_INTERCEPTION_MODE
      value: REDIRECT
    - name: ISTIO_METAJSON_ANNOTATIONS
      value: |
        {"openshift.io/scc":"restricted","sidecar.istio.io/inject":"true"}
    - name: ISTIO_METAJSON_LABELS
      value: |
        {"app":"catalog","application":"catalog","deployment":"catalog","pod-template-hash":"6d6b478456","version":"v1"}
    image: registry.redhat.io/openshift-service-mesh/proxyv2-rhel8:1.0.0
    imagePullPolicy: IfNotPresent
    name: istio-proxy
    ports:
    - containerPort: 15090
      name: http-envoy-prom
      protocol: TCP
    readinessProbe:
      failureThreshold: 30
      httpGet:
        path: /healthz/ready
        port: 15020
        scheme: HTTP
      initialDelaySeconds: 1
      periodSeconds: 2
      successThreshold: 1
      timeoutSeconds: 1
    resources:
      limits:
        cpu: 500m
        memory: 128Mi
      requests:
        cpu: 100m
        memory: 128Mi
    securityContext:
      capabilities:
        drop:
        - KILL
        - SETUID
        - SETGID
        - MKNOD
      readOnlyRootFilesystem: true
      runAsUser: 1000580001
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /etc/istio/proxy
      name: istio-envoy
    - mountPath: /etc/certs/
      name: istio-certs
      readOnly: true
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-rsww6
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  imagePullSecrets:
  - name: default-dockercfg-s5w62
  nodeName: ip-10-0-143-141.eu-west-1.compute.internal
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext:
    fsGroup: 1000580000
    seLinuxOptions:
      level: s0:c24,c14
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  - effect: NoSchedule
    key: node.kubernetes.io/memory-pressure
    operator: Exists
  volumes:
  - name: default-token-rsww6
    secret:
      defaultMode: 420
      secretName: default-token-rsww6
  - emptyDir:
      medium: Memory
    name: istio-envoy
  - name: istio-certs
    secret:
      defaultMode: 420
      optional: true
      secretName: istio.default
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2020-06-24T16:44:10Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2020-06-24T16:45:01Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2020-06-24T16:45:01Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2020-06-24T16:44:10Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: cri-o://d960af13dbbcd330583bc8ec51d27d15c99dfd1cef464d71a41c6bc1b8670de7
    image: docker.io/rhtgptetraining/foundation-catalog-service:1.0.0
    imageID: docker.io/rhtgptetraining/foundation-catalog-service@sha256:0243b9d9b01754414651430ad3dbd2d1770d70be407af52a11b14eef5abf6c01
    lastState: {}
    name: catalog
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2020-06-24T16:44:30Z"
  - containerID: cri-o://620101e6a5c6aef48b37c5f38acc0c42c82c4af0ab4288cb891dfe1501a36d80
    image: registry.redhat.io/openshift-service-mesh/proxyv2-rhel8:1.0.0
    imageID: registry.redhat.io/openshift-service-mesh/proxyv2-rhel8@sha256:1638fe91c51af0fb8d364d5d07a0786249f77def8ee088fce5a9137ab2967ccb
    lastState: {}
    name: istio-proxy
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2020-06-24T16:44:30Z"
  hostIP: 10.0.143.141
  phase: Running
  podIP: 10.130.0.32
  podIPs:
  - ip: 10.130.0.32
  qosClass: Burstable
  startTime: "2020-06-24T16:44:10Z"

$ oc create -f ~/lab/ocp-service-mesh-foundations/partner/kubernetes/partner-service-template.yml -n coolapp
deployment.extensions/partner-v1 created
$ oc create -f ~/lab/ocp-service-mesh-foundations/partner/kubernetes/Service.yml -n coolapp
service/partner created

$ oc get all -n coolapp
NAME                              READY   STATUS    RESTARTS   AGE
pod/catalog-v1-6d6b478456-9vsm2   2/2     Running   0          6m44s
pod/partner-v1-77bd558948-zskt5   1/2     Running   0          34s

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/catalog   ClusterIP   172.30.72.240   <none>        8080/TCP   5m12s
service/partner   ClusterIP   172.30.203.23   <none>        8080/TCP   9s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/catalog-v1   1/1     1            1           6m44s
deployment.apps/partner-v1   0/1     1            0           34s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/catalog-v1-6d6b478456   1         1         1       6m44s
replicaset.apps/partner-v1-77bd558948   1         1         0       34s

$ oc get all -n coolapp
NAME                              READY   STATUS    RESTARTS   AGE
pod/catalog-v1-6d6b478456-9vsm2   2/2     Running   0          7m12s
pod/partner-v1-77bd558948-zskt5   2/2     Running   0          62s

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/catalog   ClusterIP   172.30.72.240   <none>        8080/TCP   5m40s
service/partner   ClusterIP   172.30.203.23   <none>        8080/TCP   37s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/catalog-v1   1/1     1            1           7m12s
deployment.apps/partner-v1   1/1     1            1           62s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/catalog-v1-6d6b478456   1         1         1       7m12s
replicaset.apps/partner-v1-77bd558948   1         1         1       62s

$ oc create -f ~/lab/ocp-service-mesh-foundations/gateway/kubernetes/gateway-service-template.yml -n coolapp
deployment.extensions/gateway created

$ oc create -f ~/lab/ocp-service-mesh-foundations/gateway/kubernetes/Service.yml -n coolapp
service/gateway created

$ oc get all -n coolapp
NAME                              READY   STATUS    RESTARTS   AGE
pod/catalog-v1-6d6b478456-9vsm2   2/2     Running   0          8m59s
pod/gateway-6b57696f69-jgtsf      1/2     Running   0          49s
pod/partner-v1-77bd558948-zskt5   2/2     Running   0          2m49s

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/catalog   ClusterIP   172.30.72.240   <none>        8080/TCP   7m27s
service/gateway   ClusterIP   172.30.80.191   <none>        8080/TCP   40s
service/partner   ClusterIP   172.30.203.23   <none>        8080/TCP   2m24s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/catalog-v1   1/1     1            1           8m59s
deployment.apps/gateway      0/1     1            0           49s
deployment.apps/partner-v1   1/1     1            1           2m49s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/catalog-v1-6d6b478456   1         1         1       8m59s
replicaset.apps/gateway-6b57696f69      1         1         0       49s
replicaset.apps/partner-v1-77bd558948   1         1         1       2m49s
{% endhighlight %}


Now I will create the Istio ingress gateway for the mesh.

{% highlight console %}
$ cat service-mesh-gw.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - '*'
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ingress-gateway
spec:
  hosts:
  - '*'
  gateways:
  - ingress-gateway
  http:
  - match:
    - uri:
        exact: /
    route:
    - destination:
        host: gateway
        port:
          number: 8080

$ oc apply -f service-mesh-gw.yaml  -n coolapp
gateway.networking.istio.io/ingress-gateway created
virtualservice.networking.istio.io/ingress-gateway created

{% endhighlight %}

 - Identifying the URL of the ingress gateway

{% highlight console %}
$ oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}{"\n"}'
istio-ingressgateway-istio-system.apps.cluster-f2a8.f2a8.sandbox663.opentlc.com

$ export GATEWAY_URL=$(!!)
export GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}{"\n"}')

$ echo $GATEWAY_URL
istio-ingressgateway-istio-system.apps.cluster-f2a8.f2a8.sandbox663.opentlc.com

{% endhighlight %}

 - Now lets create some requets

{% highlight console %}
$ curl $GATEWAY_URL
gateway => partner => catalog v1 from '6d6b478456-9vsm2': 1
$ curl $GATEWAY_URL
gateway => partner => catalog v1 from '6d6b478456-9vsm2': 2
$ curl $GATEWAY_URL
gateway => partner => catalog v1 from '6d6b478456-9vsm2': 3
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^      ^^^^^^^^^^^^^^^^^^  ^
   names of the microservices              pod name       request number
{% endhighlight %}


#### Deploy a new version of a microservice

The next test is to create a new version of catalog microservice and let the service mesh to route to it.

{% highlight console %}
$ oc create -f lab/ocp-service-mesh-foundations/catalog-v2/kubernetes/catalog-service-template.yml -n coolapp
deployment.extensions/catalog-v2 created
{% endhighlight %}

Now we have two catalog versions running

{% highlight console %}
$ oc get pods -o wide --show-labels
NAME                          READY   STATUS    RESTARTS   AGE     IP            NODE                                         NOMINATED NODE   READINESS GATES   LABELS
catalog-v1-6d6b478456-9vsm2   2/2     Running   0          2d15h   10.130.0.32   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>            app=catalog,application=catalog,deployment=catalog,failure-domain.beta.kubernetes.io/region=eu-west-1,failure-domain.beta.kubernetes.io/zone=eu-west-1a,pod-template-hash=6d6b478456,version=v1
catalog-v2-58b586459d-wvmk8   2/2     Running   0          5m58s   10.130.0.52   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>            app=catalog,application=catalog,deployment=catalog,failure-domain.beta.kubernetes.io/region=eu-west-1,failure-domain.beta.kubernetes.io/zone=eu-west-1a,pod-template-hash=58b586459d,version=v2
gateway-6b57696f69-jgtsf      2/2     Running   0          2d14h   10.130.0.34   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>            app=gateway,application=gateway,deployment=gateway,failure-domain.beta.kubernetes.io/region=eu-west-1,failure-domain.beta.kubernetes.io/zone=eu-west-1a,pod-template-hash=6b57696f69,version=v1
partner-v1-77bd558948-zskt5   2/2     Running   0          2d14h   10.130.0.33   ip-10-0-143-141.eu-west-1.compute.internal   <none>           <none>            app=partner,application=partner,deployment=partner,failure-domain.beta.kubernetes.io/region=eu-west-1,failure-domain.beta.kubernetes.io/zone=eu-west-1a,pod-template-hash=77bd558948,version=v1
{% endhighlight %}


The service that I created originally for catalog v1 now has two endpoints, one for each version because both versions share the label __application=catalog__ .

{% highlight console %}
$ oc describe service catalog
Name:              catalog
Namespace:         coolapp
Labels:            app=catalog
Annotations:       <none>
Selector:          application=catalog
Type:              ClusterIP
IP:                172.30.72.240
Port:              http  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.130.0.32:8080,10.130.0.52:8080
Session Affinity:  None
Events:            <none>
{% endhighlight %}


By default OpenShift is loadbalancing in a round-robin fashion between the two now.

{% highlight console %}
$ curl $GATEWAY_URL
gateway => partner => catalog v2 from '58b586459d-wvmk8': 1
$ curl $GATEWAY_URL
gateway => partner => catalog v1 from '6d6b478456-9vsm2': 13
$ curl $GATEWAY_URL
gateway => partner => catalog v2 from '58b586459d-wvmk8': 2
$ curl $GATEWAY_URL
gateway => partner => catalog v1 from '6d6b478456-9vsm2': 14
{% endhighlight %}


At this point I can define rules for the service mesh in order to route the traffic in a better way.
Service Mesh supports 5 types of rules. The VirtualService object represents the logic to distribute the traffic among backends.
In this case I am requesting to route 100% of the traffic to the subset of backend hosts named __version-v2__.


{% highlight console %}
$ cat lab/ocp-service-mesh-foundations/istiofiles/virtual-service-catalog-v2.yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v2
      weight: 100

$ oc create -f lab/ocp-service-mesh-foundations/istiofiles/virtual-service-catalog-v2.yml -n coolapp
virtualservice.networking.istio.io/catalog created
{% endhighlight %}

However since there is no such subset yet created the curl to the external endpoint fails with 503.

{% highlight console %}
$ curl -v $GATEWAY_URL
* About to connect() to istio-ingressgateway-istio-system.apps.cluster-f2a8.f2a8.sandbox663.opentlc.com port 80 (#0)
*   Trying 3.248.149.209...
* Connected to istio-ingressgateway-istio-system.apps.cluster-f2a8.f2a8.sandbox663.opentlc.com (3.248.149.209) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: istio-ingressgateway-istio-system.apps.cluster-f2a8.f2a8.sandbox663.opentlc.com
> Accept: */*
> 
< HTTP/1.1 503 Service Unavailable
< x-application-context: application
< content-type: text/plain;charset=UTF-8
< content-length: 30
< date: Sat, 27 Jun 2020 15:19:47 GMT
< x-envoy-upstream-service-time: 273
< server: istio-envoy
< Set-Cookie: cd10b69e39387eb7ec9ac241201ab1ab=293892b8cc7b2c8eb074304350b1a070; path=/; HttpOnly
< 
gateway => 503 partner => 503
* Connection #0 to host istio-ingressgateway-istio-system.apps.cluster-f2a8.f2a8.sandbox663.opentlc.com left intact
{% endhighlight %}


Next step is to create this subsets now (one for each version) then. The subsets are created based on the node label __version__.

{% highlight console %}
$ cat  ~/lab/ocp-service-mesh-foundations/istiofiles/destination-rule-catalog-v1-v2.yml 
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  creationTimestamp: null
  name: catalog
spec:
  host: catalog
  subsets:
  - labels:
      version: v1
    name: version-v1
  - labels:
      version: v2
    name: version-v2
---

$ oc create -f ~/lab/ocp-service-mesh-foundations/istiofiles/destination-rule-catalog-v1-v2.yml
destinationrule.networking.istio.io/catalog created
{% endhighlight %}


Retrying that request now redirects all traffic to v2.

{% highlight console %}
$ curl $GATEWAY_URL
gateway => partner => catalog v2 from '58b586459d-wvmk8': 3
$ curl $GATEWAY_URL
gateway => partner => catalog v2 from '58b586459d-wvmk8': 4
{% endhighlight %}


Effectively catalog v2 is now responding.
Finally I am going to redirect 100% of the traffic to __version-v1__ subset.

{% highlight console %}
$ cat  ~/lab/ocp-service-mesh-foundations/istiofiles/virtual-service-catalog-v1.yml 
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v1
      weight: 100
---

$ oc replace -f ~/lab/ocp-service-mesh-foundations/istiofiles/virtual-service-catalog-v1.yml
virtualservice.networking.istio.io/catalog replace
{% endhighlight %}


__NOTE:__ since the rule already existed I need to use replace and not create.


Verify that the traffic is effectively reaching v1 catalog microservice:

{% highlight console %}
$ curl $GATEWAY_URL
gateway => partner => catalog v1 from '6d6b478456-9vsm2': 15
$ curl $GATEWAY_URL
gateway => partner => catalog v1 from '6d6b478456-9vsm2': 16
{% endhighlight %}



Lastly, I can split traffic 70% to catalog v1 and 30% to catalog v2.

{% highlight console %}
$ cat ~/lab/ocp-service-mesh-foundations/istiofiles/virtual-service-catalog-v1_and_v2_70_30.yml 
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  creationTimestamp: null
  name: catalog
spec:
  hosts:
  - catalog
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v1
      weight: 70
    - destination:
        host: catalog
        subset: version-v2
      weight: 30
---
$ oc replace -f  ~/lab/ocp-service-mesh-foundations/istiofiles/virtual-service-catalog-v1_and_v2_70_30.yml  -n coolapp
virtualservice.networking.istio.io/catalog replaced
$ oc describe VirtualService catalog -n coolapp
Name:         catalog
Namespace:    coolapp
Labels:       <none>
Annotations:  <none>
API Version:  networking.istio.io/v1alpha3
Kind:         VirtualService
Metadata:
  Creation Timestamp:  2020-06-27T08:50:48Z
  Generation:          3
  Resource Version:    2751982
  Self Link:           /apis/networking.istio.io/v1alpha3/namespaces/coolapp/virtualservices/catalog
  UID:                 75c92a7b-a685-4563-a0de-4d491a7fa97b
Spec:
  Hosts:
    catalog
  Http:
    Route:
      Destination:
        Host:    catalog
        Subset:  version-v1
      Weight:    70
      Destination:
        Host:    catalog
        Subset:  version-v2
      Weight:    30
Events:          <none>

{% endhighlight %}

Until next time.


