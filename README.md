# Scale OpenShift ingresscontroller using Custom Metrics Autoscaler (KEDA)

This repo is an example of:

- Installing OpenShift [Custom Metrics Autoscaler](https://cloud.redhat.com/blog/custom-metrics-autoscaler-on-openshift) operator on your cluster
- Creating Custom Metrics Autoscaler resources (triggerauthentication, scaledobject, etc)
- Scaling the default ingresscontroller based on the number of nodes

## Useful links / prior work

- https://github.com/zroubalik/keda-openshift-examples/tree/main/prometheus
- https://access.redhat.com/articles/6718611
- https://issues.redhat.com/browse/RHSTOR-1938

## Prerequisites

1. Clone this repo

```sh
$ git clone https://github.com/frobware/openshift-keda-examples && cd openshift-keda-examples
```

Do all operations in the openshift-ingress-operator namespace:

```sh
$ oc project openshift-ingress-operator
```

# Installing Custom Metrics Autoscaler

This [existing
demo](https://github.com/zroubalik/keda-openshift-examples/tree/main/prometheus/ocp-monitoring)
details the steps that need to be run which I have encapsulated in the
following [setup/script](./setup/setup.sh). The setup script is for
convenience; let's go through all the steps explicitly:

1. Install Custom Metrics Autoscaler from OperatorHub

In the OperatorHub locate and install Custom Metrics Autoscaler.

![Custom Metrics Adapter](screenshots/custom-metrics-adapter.png?raw=true "Custom Metrics Autoscaler")

Once the operator is installed also create a `KedaController`
instance. The documentation in the OperatorHub offers a 1-click option
to do this once the operator is running.

![Create KEDA controller instance](screenshots/create-keda-controller-instance.png?raw=true "Create KEDA controller instance")

A functioning Custom Metrics Autoscaler setup will have 3 pods
running:

```sh
% oc get pods -n openshift-keda
NAME                                                  READY   STATUS    RESTARTS   AGE
custom-metrics-autoscaler-operator-5b865f6b96-p2jqw   1/1     Running   0          20h
keda-metrics-apiserver-7bb57b45b9-vxvmt               1/1     Running   0          20h
keda-operator-bd446d79c-skxjk                         1/1     Running   0          20h
```

2. Enable OpenShift monitoring for user-defined projects

Please refer to the
[documentation](https://docs.openshift.com/container-platform/4.9/monitoring/enabling-monitoring-for-user-defined-projects.html),
or just apply the following ConfigMap:

```sh
$ oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

3. Create a Service Account

You need a Service Account to authenticate with Thanos:

```sh
$ oc create serviceaccount thanos
$ oc describe serviceaccount thanos
Name:                thanos
Namespace:           openshift-ingress-operator
Labels:              <none>
Annotations:         <none>
Image pull secrets:  thanos-dockercfg-b4l9s
Mountable secrets:   thanos-dockercfg-b4l9s
Tokens:              thanos-token-c422q
Events:              <none>
```

4. Define a TriggerAuthentication with the Service Account's token

```sh
$ secret=$(oc get secret | grep thanos-token | head -n 1 | awk '{ print $1 }')
$ echo $secret
thanos-token-c422q

$ oc process TOKEN="$secret" -f - <<EOF | oc apply -f -
apiVersion: template.openshift.io/v1
kind: Template
parameters:
- name: TOKEN
objects:
- apiVersion: keda.sh/v1alpha1
  kind: TriggerAuthentication
  metadata:
    name: keda-trigger-auth-prometheus
  spec:
    secretTargetRef:
    - parameter: bearerToken
      name: \${TOKEN}
      key: token
    - parameter: ca
      name: \${TOKEN}
      key: ca.crt
EOF
```

5. Create a role for reading metrics from Thanos

```sh
$ oc apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: thanos-metrics-reader
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  verbs:
  - get
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
EOF
```

6. Add the role for reading metrics from Thanos to the Service Account

```sh
$ oc adm policy add-role-to-user thanos-metrics-reader -z thanos --role-namespace=openshift-ingress-operator
$ oc adm policy -n openshift-ingress-operator add-cluster-role-to-user cluster-monitoring-view -z thanos
```

The previous `add-cluster-role-to-user` step is only required if you
use cross-namespace queries (which our examples will do); if you are
running through the steps in this README then you will need to run the
`add-cluster-role-to-user` step.

Our scaledobject examples, which all run in the
`openshift-ingress-operator` namespace, use a metric from the
`kube-metrics` namespace (i.e., cross-namespace).

If you don't run both the `oc adm policy` steps above then the
scaledobject will remain in a non-active state due to lack of
permissions:

```
    $ oc get scaledobject
    NAME             SCALETARGETKIND                              SCALETARGETNAME   MIN   MAX   TRIGGERS     AUTHENTICATION                 READY   ACTIVE   FALLBACK   AGE
    ingress-scaler   operator.openshift.io/v1.IngressController   default           1     20    prometheus   keda-trigger-auth-prometheus   True    False    True       99s
```

# Autoscaling deployments

## Scale a simple "hello world" deployment

Before we scale an ingresscontroller we will first demonstrate scaling
a very simple deployment. The goal here is to ensure all the Cluster
Metrics Adapter pieces have been setup correctly.

1. Deploy the `hello-app`

```sh
$ oc apply -f ./hello-app/deployment.yaml

$ oc get all -l app=hello-app
NAME                             READY   STATUS    RESTARTS   AGE
pod/hello-app-6bddd4f888-2lpf4   1/1     Running   0          175m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-app   1/1     1            1           4h13m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-app-6bddd4f888   1         1         1       4h13m
```

2. Deploy the `nodes-ready-app`

This is a simple application that tracks the number of ready nodes in
the cluster, exposing that number as the `nodes_ready` metric. We will
run this deployment in its own namespace to prove that cross namespace
queries are functional:

```sh
$ oc apply -f ./nodes-ready-app/manifests/namespace.yaml
namespace/nodes-ready-app created
```

Ensure all remaining objects are created in the nodes-ready-app
namespace:

```sh
$ oc apply -n nodes-ready-app -f ./nodes-ready-app/manifests
clusterrolebinding.rbac.authorization.k8s.io/clusterrole-binding created
clusterrole.rbac.authorization.k8s.io/clusterrole-node-viewer created
deployment.apps/nodes-ready-app created
service/nodes-ready-app created
servicemonitor.monitoring.coreos.com/nodes-ready-app-service-monitor created
namespace/nodes-ready-app unchanged
serviceaccount/nodes-ready-app created
```

Verify that the nodes-ready-app starts and logs the number of ready
nodes in the cluster:

```sh
$ oc get pods -n nodes-ready-app
NAME                              READY   STATUS    RESTARTS   AGE
nodes-ready-app-7dddbccbf-8mqn6   1/1     Running   0          48s

$ oc logs -n nodes-ready-app nodes-ready-app-7dddbccbf-zrlwt
I0722 10:03:06.431195       1 main.go:87] Setting UPDATE_INTERVAL="5s"
I0722 10:03:06.433676       1 shared_informer.go:255] Waiting for caches to sync for nodes-ready-app
I0722 10:03:07.534151       1 shared_informer.go:262] Caches are synced for nodes-ready-app
I0722 10:03:11.435815       1 main.go:199] 3 nodes, 3 ready
I0722 10:03:16.438525       1 main.go:199] 3 nodes, 3 ready
I0722 10:03:21.440390       1 main.go:199] 3 nodes, 3 ready
```

I have 3 nodes in my cluster, so this looks good!

```sh
$ oc get nodes
NAME                               STATUS   ROLES           AGE   VERSION
master-0.ocp411.int.frobware.com   Ready    master,worker   31d   v1.24.0+9546431
master-1.ocp411.int.frobware.com   Ready    master,worker   31d   v1.24.0+9546431
master-2.ocp411.int.frobware.com   Ready    master,worker   31d   v1.24.0+9546431
```

You should also verify that the `nodes_ready` metric is reported to
prometheus; via the console make a query in Observe => Metrics:

![nodes_ready](screenshots/nodes_ready-metric.png?raw=true "nodes_ready metric")

## Create a ScaledObject to scale the hello-app

Finally we can talk about scaling!

Create the following `ScaledObject`:

```sh
$ oc apply -f - <<EOF
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: hello-app-scaler
spec:
  scaleTargetRef:
    name: hello-app
  minReplicaCount: 1
  maxReplicaCount: 20
  cooldownPeriod: 1
  pollingInterval: 1
  triggers:
  - type: prometheus
    metricType: AverageValue
    metadata:
      serverAddress: https://thanos-querier.openshift-monitoring.svc.cluster.local:9091
      namespace: openshift-ingress-operator
      metricName: 'nodes-ready'
      threshold: '1'
      query: 'nodes_ready{service="nodes-ready-app"}'
      authModes: "bearer"
    authenticationRef:
      name: keda-trigger-auth-prometheus
EOF
```

Let's now verify that our scaledobject was created successfully:

```sh
$ oc get scaledobject
NAME               SCALETARGETKIND      SCALETARGETNAME   MIN   MAX   TRIGGERS     AUTHENTICATION                 READY   ACTIVE   FALLBACK   AGE
hello-app-scaler   apps/v1.Deployment   hello-app         1     20    prometheus   keda-trigger-auth-prometheus   True    True     False      19s
```

If both the `Ready` and `Active` columns report `True` then the
scaledobject has been created successfully. If either of those columns
report as `False` then double-check that the metric is query-able in
prometheus.

Under the hood scaling is implemented via a `HorizontalPodAutoscaler`:

```sh
$ oc get hpa
NAME                        REFERENCE              TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-hello-app-scaler   Deployment/hello-app   1/1 (avg)   1         20        3          2m30s
```

And we can see here that it has already scaled our hello-app
deployment to 3 replicas.

To verify we should see 3 hello-app pods listed:

```sh
$ oc get pods
NAME                                READY   STATUS    RESTARTS   AGE
hello-app-6bddd4f888-2lpf4          1/1     Running   0          3h27m
hello-app-6bddd4f888-bdjtm          1/1     Running   0          3m16s
hello-app-6bddd4f888-smwkj          1/1     Running   0          3m16s
ingress-operator-78dbffb964-8d9lg   2/2     Running   0          3d21h
```

We can test scale up/down a little further by poking the
`nodes-ready-app` to report a different value for the number of ready
nodes.

```sh
$ oc get pods -n nodes-ready-app
NAME                              READY   STATUS    RESTARTS   AGE
nodes-ready-app-7dddbccbf-zrlwt   1/1     Running   0          24m
```

Set the number of ready nodes to be 50.

```sh
$ oc rsh -n nodes-ready-app nodes-ready-app-7dddbccbf-zrlwt curl -d "ready=50" http://localhost:8080/poke
poked=50
```

The nodes-ready-app will now continuously report 50 ready nodes until
reset.

```sh
$ oc logs -n nodes-ready-app nodes-ready-app-7dddbccbf-zrlwt
I0722 10:28:31.936741       1 main.go:199] 3 nodes, 3 ready
I0722 10:28:36.936911       1 main.go:199] 3 nodes, 3 ready
I0722 10:28:41.939688       1 main.go:199] 3 nodes, 3 ready
I0722 10:28:46.940989       1 main.go:195] poked mode: 50 ready
I0722 10:28:51.941307       1 main.go:195] poked mode: 50 ready
I0722 10:28:56.944603       1 main.go:195] poked mode: 50 ready
# This overriden value can be reset by poking 0.
```

What happened to our scaledobject, the underlying HPA, and the
hello-app deployment?

```sh
$ oc get scaledobject
NAME               SCALETARGETKIND      SCALETARGETNAME   MIN   MAX   TRIGGERS     AUTHENTICATION                 READY   ACTIVE   FALLBACK   AGE
hello-app-scaler   apps/v1.Deployment   hello-app         1     20    prometheus   keda-trigger-auth-prometheus   True    True     False      9m27s

$ oc get hpa
NAME                        REFERENCE              TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-hello-app-scaler   Deployment/hello-app   2500m/1 (avg)   1         20        20         9m42s

$ oc get deployment -l app=hello-app
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
hello-app   20/20   20           20          4h51m
```

We topped out 20 replicas because we configured the maximum to be 20
in our scaledobject. Without configuring an upperbound it would have
scaled to 50.

Before we continue let's reset our `nodes-ready-app` to report on the
actual number of nodes in the cluster:

```sh
$ oc rsh -n nodes-ready-app nodes-ready-app-7dddbccbf-zrlwt curl -d "ready=0" http://localhost:8080/poke
```

And once we do that we will see the `hello-app` scale back to 3
replicas:

```sh
$ oc get deployment -l app=hello-app
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
hello-app   3/3     3            3           5h

$ oc get hpa
NAME                        REFERENCE              TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-hello-app-scaler   Deployment/hello-app   1/1 (avg)   1         20        3          19m
```

## Scaling the default ingresscontroller

Now that we know autoscaling works for a simple deployment let's
target an ingresscontroller--which is a custom resource.

First let's verify how many replicas we currently have for the
`default` ingresscontroller; We are expecting 2:

```sh
$ oc get ingresscontroller/default -o yaml | grep replicas:
replicas: 2

$ oc get pods -n openshift-ingress
NAME                             READY   STATUS    RESTARTS   AGE
router-default-7b5df44ff-l9pmm   2/2     Running   0          17h
router-default-7b5df44ff-s5sl5   2/2     Running   0          3d21h
```

Create a new scaledobject targeting the `default` ingresscontroller
deployment:

```sh
$ oc apply -f - <<EOF
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ingress-scaler
spec:
  scaleTargetRef:
    apiVersion: operator.openshift.io/v1
    kind: IngressController
    name: default
    envSourceContainerName: ingress-operator
  minReplicaCount: 1
  maxReplicaCount: 20
  cooldownPeriod: 1
  pollingInterval: 1
  triggers:
  - type: prometheus
    metricType: AverageValue
    metadata:
      serverAddress: https://thanos-querier.openshift-monitoring.svc.cluster.local:9091
      namespace: openshift-ingress-operator
      metricName: 'kube-node-role'
      threshold: '1'
      query: 'sum(kube_node_role{role="worker",service="kube-state-metrics"})'
      authModes: "bearer"
    authenticationRef:
      name: keda-trigger-auth-prometheus
EOF
```

To target a custom resource (i.e., the ingresscontroller) the changes
over and above the different query amount to:

```console
   scaleTargetRef:
     apiVersion: operator.openshift.io/v1
     kind: IngressController
     name: default
     envSourceContainerName: ingress-operator
```

And we're purposely using a different query to prove that we can use
metrics that already exist in a deployed cluster:

```sh
$ oc apply -f ./ingresscontroller/scale-on-kube-node-role.yaml
scaledobject.keda.sh/ingress-scaler created

$ oc get scaledobject
NAME             SCALETARGETKIND                              SCALETARGETNAME   MIN   MAX   TRIGGERS     AUTHENTICATION                 READY   ACTIVE   FALLBACK   AGE
ingress-scaler   operator.openshift.io/v1.IngressController   default           1     20    prometheus   keda-trigger-auth-prometheus   False   True     Unknown    4s

$ oc get hpa
NAME                      REFERENCE                   TARGETS             MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-ingress-scaler   IngressController/default   <unknown>/1 (avg)   1         20        0          7s
```

Waiting a little while we see the `default` ingresscontroler scaled
out to 3 replicas which matches our kube-state-metrics query.

```sh
$ oc get ingresscontroller/default -o yaml | grep replicas:
replicas: 3

$ oc get pods -n openshift-ingress
NAME                             READY   STATUS    RESTARTS   AGE
router-default-7b5df44ff-l9pmm   2/2     Running   0          17h
router-default-7b5df44ff-s5sl5   2/2     Running   0          3d22h
router-default-7b5df44ff-wwsth   2/2     Running   0          66s
```

![kube-state-metrics](screenshots/kube-state-metrics.png?raw=true "kube-state-metrics")
