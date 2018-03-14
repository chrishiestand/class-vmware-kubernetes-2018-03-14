## Advanced Kubernetes Lab 1 - Advanced Scheduling

### Node Affinity/Anti-Affinity

ssh into your lab vm

```bash
ssh -i <key_file> play@<ip>
```

Get list of nodes:
```
kubectl get nodes
```

You should have 2 nodes:
```
NAME                                      STATUS    ROLES     AGE       VERSION
gke-kubelab1-default-pool-9e771e91-p49w   Ready     <none>    29m       v1.9.3-gke.0
gke-kubelab1-default-pool-c2e20d4a-ln1l   Ready     <none>    29m       v1.9.3-gke.0
```

Set a label on the last node

```
kubectl label nodes gke-kubelab1-default-pool-9e771e91-p49w disktype=ssd
```

Ensure the node label was applied:
```
kubectl get nodes --show-labels
```

```
NAME                                      STATUS    ROLES     AGE       VERSION        LABELS
gke-kubelab1-default-pool-9e771e91-p49w   Ready     <none>    32m       v1.9.3-gke.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/fluentd-ds-ready=true,beta.kubernetes.io/instance-type=g1-small,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,disktype=ssd,failure-domain.beta.kubernetes.io/region=us-west1,failure-domain.beta.kubernetes.io/zone=us-west1-c,kubernetes.io/hostname=gke-kubelab1-default-pool-9e771e91-p49w
gke-kubelab1-default-pool-c2e20d4a-ln1l   Ready     <none>    32m       v1.9.3-gke.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/fluentd-ds-ready=true,beta.kubernetes.io/instance-type=g1-small,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=default-pool,failure-domain.beta.kubernetes.io/region=us-west1,failure-domain.beta.kubernetes.io/zone=us-west1-b,kubernetes.io/hostname=gke-kubelab1-default-pool-c2e20d4a-ln1l
```

Update the YAML file `ssd_pod.yaml` with `nodeSelector`, and save it
```yaml
apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
    labels:
      env: test
  spec:
    containers:
    - name: nginx
      image: nginx
      imagePullPolicy: IfNotPresent
      nodeSelector:
      disktype: ssd
```

Deploy the application:
```
kubectl create -f kubernetes/manifests/ssd_pod.yaml
```

Confirm POD was assigned to node with label:

```
kubectl get pods -o wide
```

## Cleanup

Delete nginx pod

```
kubectl delete pod nginx
```


## More Useful Example

Kubernetes has built-in node labels that can be used without being applied manually first.

Interpod Affinity and AntiAffinity can be even more useful when they are used with higher level collections such as ReplicaSets, Statefulsets, Deployments, etc. One can easily configure that a set of workloads should be co-located in the same defined topology, eg., the same node.

## Colocating or not colocating on the same node

In a three node cluster, a web application has in-memory cache such as redis. We want the web-servers to be co-located with the cache as much as possible, but we don't want the caches located on the same node, or the web servers on the same node (to spread load). Here is the yaml snippet of a simple redis deployment with three replicas and selector label app=store. The deployment has Pod AntiAffinities and Affinities configured to ensure the behavior we want.


## Deploy redis YAML

```
kubectl apply -f kubernetes/manifests/affinity.yaml
```

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: redis-cache
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

The below yaml snippet of the webserver deployment has `podAntiAffinity` and `podAffinity` configured. This informs the scheduler that all its replicas are to be co-located with pods that have selector label `app=store`. This will also ensure that each web-server replica does not co-locate on a single node.

Now let’s deploy the YAML for web servers.

```
kubectl apply -f kubernetes/manifests/anti_and_affinity.yaml
```

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
```

After creating the above two deployments, our three node cluster should look like below:

```
kubctl get pod -o wide
```

```
NAME                           READY     STATUS    RESTARTS   AGE       IP           NODE
redis-cache-6c87cc9cbb-427rn   1/1       Running   0          11m       10.20.0.15   gke-kubelab1-default-pool-9e771e91-p49w
redis-cache-6c87cc9cbb-lhwnd   1/1       Running   0          11m       10.20.1.14   gke-kubelab1-default-pool-c2e20d4a-ln1l
web-server-669585bcf4-76z2b    1/1       Running   0          2m        10.20.0.16   gke-kubelab1-default-pool-9e771e91-p49w
web-server-669585bcf4-gfs5s    1/1       Running   0          2m        10.20.1.15   gke-kubelab1-default-pool-c2e20d4a-ln1l
```

As you can see, both replicas of the web-server are automatically co-located with the cache as expected.

Best practice is to configure these highly available stateful workloads such as Redis with AntiAffinity rules for more guaranteed spreading.


## Never co-located in the same node

A Highly Available database statefulset has one master and three replicas, one may prefer none of the database instances to be co-located on the same node.
Using Anti-Affinity we can specify that each of the DB-REPLICAs will be on separate nodes.

## Cleanup

Delete deployments
```
kubectl delete deployment --all
```


## Taint and Tolerations Lab

Node affinity is a property of pods that attracts them to a set of nodes (either as a preference or a hard requirement). Taints are the opposite – they allow a node to repel a set of pods.

Taints and tolerations work together to ensure that pods are not scheduled onto inappropriate nodes. One or more taints are applied to a node; this marks that the node should not accept any pods that do not tolerate the taints. Tolerations are applied to pods, and allow (but do not require) the pods to schedule onto nodes with matching taints.

You add a taint to a node using `kubectl taint` . For example:

```
kubectl taint nodes node1 key=value:NoSchedule
```
places a taint on node `node1` The taint has key `key` , value `value` , and taint effect `NoSchedule` .

This means that no pod will be able to schedule onto node1 unless it has a matching toleration.

You specify a toleration for a pod in the `PodSpec`. Both of the following tolerations “match” the taint created by the `kubectl taint` line above, and thus a pod with either toleration would be able to schedule onto `node1`:

```yaml
tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

```yaml
tolerations:
  - key: "key"
    operator: "Exists"
    effect: "NoSchedule"
```

NOTE: An empty `key` with operator `Exists` matches all keys ,values and effects which means it will tolerate everything.

```yaml
tolerations:
  - operator: "Exists"
```


An empty effect matches all effects with a key `key`

```yaml
tolerations:
  - key: "key"
    operator: "Exists"
```

## Apply taint to node

In this lab we will be applying a taint and then showing how only PODs with an exception are allowed to be scheduled on that node.

Get nodes:

```
kubectl get nodes
```

```
NAME                                      STATUS    ROLES     AGE       VERSION
gke-kubelab1-default-pool-9e771e91-p49w   Ready     <none>    1h        v1.9.3-gke.0
gke-kubelab1-default-pool-c2e20d4a-ln1l   Ready     <none>    1h        v1.9.3-gke.0
```

Apply a taint to the top node.

```
kubectl taint nodes gke-kubelab1-default-pool-9e771e91-p49w dedicated=lab:NoSchedule
```
