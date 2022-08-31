# Cluster Setup

% oc version
Client Version: 4.10.27
Server Version: 4.12.0-ec.2
Kubernetes Version: v1.24.0+ed93380

% oc get machinesets -n openshift-machine-api
NAME                                            DESIRED   CURRENT   READY   AVAILABLE   AGE
amcdermo-2022-08-31-1-bhn4f-worker-us-east-2a   1         1         1       1           25m
amcdermo-2022-08-31-1-bhn4f-worker-us-east-2b   1         1         1       1           25m
amcdermo-2022-08-31-1-bhn4f-worker-us-east-2c   1         1         1       1           25m

% oc scale --replicas=5 -n openshift-machine-api machinesets/amcdermo-2022-08-31-1-bhn4f-worker-us-east-2a
machineset.machine.openshift.io/amcdermo-2022-08-31-1-bhn4f-worker-us-east-2a scaled

% oc scale --replicas=5 -n openshift-machine-api machinesets/amcdermo-2022-08-31-1-bhn4f-worker-us-east-2b
machineset.machine.openshift.io/amcdermo-2022-08-31-1-bhn4f-worker-us-east-2b scaled

% oc scale --replicas=5 -n openshift-machine-api machinesets/amcdermo-2022-08-31-1-bhn4f-worker-us-east-2c
machineset.machine.openshift.io/amcdermo-2022-08-31-1-bhn4f-worker-us-east-2c scaled

% oc get -n openshift-machine-api machinesets
NAME                                            DESIRED   CURRENT   READY   AVAILABLE   AGE
amcdermo-2022-08-31-1-bhn4f-worker-us-east-2a   5         5         5       5           31m
amcdermo-2022-08-31-1-bhn4f-worker-us-east-2b   5         5         5       5           31m
amcdermo-2022-08-31-1-bhn4f-worker-us-east-2c   5         5         5       5           31m

# Following the instructions

And now following the steps in https://github.com/frobware/openshift-keda-examples

Install KEDA via the console ...

% oc get pods -n openshift-keda
NAME                                                  READY   STATUS    RESTARTS   AGE
custom-metrics-autoscaler-operator-5c6df9445b-pttdw   1/1     Running   0          62s

Create KEDA controller instance (in the console)

% oc get pods -n openshift-keda
NAME                                                  READY   STATUS    RESTARTS   AGE
custom-metrics-autoscaler-operator-5c6df9445b-pttdw   1/1     Running   0          2m33s
keda-metrics-apiserver-7bb57b45b9-wzgfh               1/1     Running   0          47s
keda-operator-bd446d79c-dmvxw                         1/1     Running   0          47s

The remainder of the instructions should be done in the
openshift-ingress-operator namespace.

% oc project openshift-ingress-operator
Now using project "openshift-ingress-operator" on server "https://api.amcdermo-2022-08-31-1354.devcluster.openshift.com:6443".

% cd /tmp

% git clone https://github.com/frobware/openshift-keda-examples && cd openshift-keda-examples
Cloning into 'openshift-keda-examples'...
remote: Enumerating objects: 65, done.
remote: Counting objects: 100% (65/65), done.
remote: Compressing objects: 100% (50/50), done.
remote: Total 65 (delta 21), reused 57 (delta 13), pack-reused 0
Receiving objects: 100% (65/65), 1.51 MiB | 3.75 MiB/s, done.
Resolving deltas: 100% (21/21), done.

Now regurgitating from Step 2 in the README^^

2. Enable OpenShift monitoring for user-defined projects

% oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
configmap/cluster-monitoring-config created

3. Create a Service Account

% oc create serviceaccount thanos
serviceaccount/thanos created

% oc describe serviceaccount thanos
Name:                thanos
Namespace:           openshift-ingress-operator
Labels:              <none>
Annotations:         <none>
Image pull secrets:  thanos-dockercfg-ws8h8
Mountable secrets:   thanos-dockercfg-ws8h8
Tokens:              thanos-token-ppwgj
Events:              <none>

% secret=$(oc get secret | grep thanos-token | head -n 1 | awk '{ print $1 }')

% echo $secret
thanos-token-ppwgj

% oc process TOKEN="$secret" -f - <<EOF | oc apply -f -
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
triggerauthentication.keda.sh/keda-trigger-auth-prometheus created

5. Create a role for reading metrics from Thanos

% oc apply -f - <<EOF
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
role.rbac.authorization.k8s.io/thanos-metrics-reader created

6. Add the role for reading metrics from Thanos to the Service Account

% oc adm policy add-role-to-user thanos-metrics-reader -z thanos --role-namespace=openshift-ingress-operator
role.rbac.authorization.k8s.io/thanos-metrics-reader added: "thanos"

% oc adm policy -n openshift-ingress-operator add-cluster-role-to-user cluster-monitoring-view -z thanos
clusterrole.rbac.authorization.k8s.io/cluster-monitoring-view added: "thanos"

And now skipping to the following step:

https://github.com/frobware/openshift-keda-examples#scaling-the-default-ingresscontroller

% oc get ingresscontroller/default -o yaml | grep replicas:
  replicas: 2

% oc get nodes | grep worker
ip-10-0-128-209.us-east-2.compute.internal   Ready    worker                 15m   v1.24.0+ed93380
ip-10-0-132-138.us-east-2.compute.internal   Ready    worker                 15m   v1.24.0+ed93380
ip-10-0-134-167.us-east-2.compute.internal   Ready    worker                 36m   v1.24.0+ed93380
ip-10-0-154-219.us-east-2.compute.internal   Ready    worker                 15m   v1.24.0+ed93380
ip-10-0-157-111.us-east-2.compute.internal   Ready    worker                 15m   v1.24.0+ed93380
ip-10-0-162-236.us-east-2.compute.internal   Ready    worker                 15m   v1.24.0+ed93380
ip-10-0-166-174.us-east-2.compute.internal   Ready    worker                 16m   v1.24.0+ed93380
ip-10-0-171-86.us-east-2.compute.internal    Ready    worker                 38m   v1.24.0+ed93380
ip-10-0-190-42.us-east-2.compute.internal    Ready    worker                 15m   v1.24.0+ed93380
ip-10-0-191-204.us-east-2.compute.internal   Ready    worker                 15m   v1.24.0+ed93380
ip-10-0-194-197.us-east-2.compute.internal   Ready    worker                 38m   v1.24.0+ed93380
ip-10-0-208-231.us-east-2.compute.internal   Ready    worker                 15m   v1.24.0+ed93380
ip-10-0-211-254.us-east-2.compute.internal   Ready    worker                 14m   v1.24.0+ed93380
ip-10-0-212-120.us-east-2.compute.internal   Ready    worker                 14m   v1.24.0+ed93380
ip-10-0-212-210.us-east-2.compute.internal   Ready    worker                 15m   v1.24.0+ed93380

Create a new scaledobject targeting the default ingresscontroller deployment:

% oc apply -f - <<EOF
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
scaledobject.keda.sh/ingress-scaler created

% oc apply -f ./ingresscontroller/scale-on-kube-node-role.yaml
scaledobject.keda.sh/ingress-scaler configured

% oc get scaledobject
NAME             SCALETARGETKIND                              SCALETARGETNAME   MIN   MAX   TRIGGERS     AUTHENTICATION                 READY   ACTIVE   FALLBACK   AGE
ingress-scaler   operator.openshift.io/v1.IngressController   default           1     20    prometheus   keda-trigger-auth-prometheus   True    True     False      55s

% oc get hpa
NAME                      REFERENCE                   TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-ingress-scaler   IngressController/default   3750m/1 (avg)   1         20        15         74s

% oc get ingresscontroller/default -o yaml | grep replicas:
  replicas: 15
