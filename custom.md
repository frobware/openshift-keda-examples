# Experimenting with custom metrics

## Scale a simple "hello world" deployment

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

