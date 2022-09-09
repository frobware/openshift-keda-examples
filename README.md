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

# Autoscaling the default ingress controller

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

2. Do all operations in the openshift-ingress-operator namespace:

```sh
$ oc project openshift-ingress-operator
```

3. Enable OpenShift monitoring for user-defined projects

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

4. Create a Service Account

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

5. Define a TriggerAuthentication with the Service Account's token

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

6. Create a role for reading metrics from Thanos

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

7. Add the role for reading metrics from Thanos to the Service Account

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

8. Create a scaledobject for scaling the default ingresscontroller

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
