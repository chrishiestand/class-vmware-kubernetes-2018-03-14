# Pod Networking

Look at `simple_pod.yaml` and note the env section.

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: simple-api
    labels:
      lab: podlab
spec:
    containers:
    - name: simpleservice
      image: mhausenblas/simpleservice:0.5.0
      ports:
      - containerPort: 9876
      env:
      - name: DEMO_GREETING
        value: "Hello from the environment!!"
```

Create the pod from the manifest.

```
kubectl apply -f simple_pod.yaml
```

Confirm the pod is running:

```
# "po" is not a type, kubectl supports several aliases, including abbreviations:
kubectl get po -o wide

# and plural or singular
kubectl get pods -o wide

```

Note the IP for the next step:

```
simple-api   1/1       Running   0          5m        10.20.1.25   gke-kubelab1-default-pool-c2e20d4a-ln1l
```

## Access web service

Because we have no yet exposed the `simple-api` service externally it is only accessible through cluster nodes. So we are going to use a jump container to log in and run commands on the internal cluster network.

### Create a jump pod

```
kubectl run -i -t --rm jumpserver --image=satoms/jumpserver:v1 --restart=Never
```

NOTE: The `--restart` policy flag on the kubectl run command determines whether the command will start a Deployment (`--restart=Always`), a Job (`--restart=OnFailure`), or a bare pod (`--restart=Never`). The `-i` and `-t` flags function similarly to Docker flags to instantiate an interactive TTY in the foreground for the pod container. The `--rm` flag ensures that the pod resources are deleted when the pod container exits.

Using the jumpserver let’s `curl` the `simple-api` POD on the `IP`.

```bash
curl http://<IP>:9876/info
```

You should get something like:
```json
{"host": "10.20.1.25:9876", "version": "0.5.0", "from": "10.20.1.26"}
```

We can also call the /env endpoint to request the service echo all of its runtime environment variables:

```
curl http://<IP>:9876/env
```

```json
{
  "version": "0.5.0",
  "env": "{'KUBERNETES_PORT_443_TCP': 'tcp://10.23.240.1:443', 'NGINX_PORT': 'tcp://10.23.243.185:80', 'NGINX_PORT_80_TCP_PORT': '80', 'HOME': '/root', 'PATH': '/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin', 'NPSERV_PORT_80_TCP_ADDR': '10.23.250.255', 'KUBERNETES_SERVICE_PORT': '443', 'LANG': 'C.UTF-8', 'NPSERV_PORT_80_TCP_PROTO': 'tcp', 'PYTHON_VERSION': '2.7.13', 'KUBERNETES_SERVICE_HOST': '10.23.240.1', 'PYTHON_PIP_VERSION': '9.0.1', 'NGINX_PORT_80_TCP_PROTO': 'tcp', 'DEMO_GREETING': 'Hello from the environment!!', 'REFRESHED_AT': '2017-04-24T13:50', 'NPSERV_PORT': 'tcp://10.23.250.255:80', 'GPG_KEY': 'C01E1CAD5EA2C4F0B8E3571504C367C218ADD4FF', 'NGINX_PORT_80_TCP': 'tcp://10.23.243.185:80', 'NPSERV_SERVICE_PORT': '80', 'NPSERV_SERVICE_HOST': '10.23.250.255', 'KUBERNETES_PORT_443_TCP_ADDR': '10.23.240.1', 'NGINX_SERVICE_HOST': '10.23.243.185', 'NPSERV_PORT_80_TCP_PORT': '80', 'KUBERNETES_PORT': 'tcp://10.23.240.1:443', 'KUBERNETES_SERVICE_PORT_HTTPS': '443', 'NPSERV_PORT_80_TCP': 'tcp://10.23.250.255:80', 'KUBERNETES_PORT_443_TCP_PROTO': 'tcp', 'HOSTNAME': 'simple-api', 'NGINX_SERVICE_PORT': '80', 'KUBERNETES_PORT_443_TCP_PORT': '443', 'NGINX_PORT_80_TCP_ADDR': '10.23.243.185'}"
}
```

Notice the environment set in the POD manifest has been injected into the environment.

```
'DEMO_GREETING': 'Hello from the environment!!'
```


### Expose simple-api service externally

So back on your normal lab VM shell, if we want to access this pod without using a jump container you might think we'd want to "expose" the pod and map the container port to a Kubernetes port. Expose is just syntactic sugar for creating a service.

```
kubectl expose pod simple-api --port=4000 --target-port=9876 --name=simple-api
```

Check out the service:

```
kubectl get svc simple-api
```

You’ll see the `ClusterIP` displayed.

```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
simple-api   ClusterIP   10.23.248.120   <none>        4000/TCP   11m
```

Try connecting to the ClusterIP from your lab VM:

```bash
curl --connect-timeout 2 http://10.23.248.120:4000/info
curl: (28) Connection timed out after 2001 milliseconds
```

It still doesn't work does it?

Let's try that from our jump shell again.

```
curl --connect-timeout 2 http://10.23.248.120:4000/info
{"host": "10.23.248.120:4000", "version": "0.5.0", "from": "10.20.1.27"}
```

The jump container works again, what gives? The problem is that ClusterIP is just that - a **cluster* ip. It's a virtual IP and the routing for it only exists on kubernetes nodes.

Let's try again with a network load balancer:
```
kubectl delete svc simple-api

kubectl expose pod simple-api --port=80 --target-port=9876 --name=simple-api --type=LoadBalancer
service "simple-api" exposed
```

```
kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE
kubernetes   ClusterIP      10.23.240.1     <none>         443/TCP          21h
simple-api   LoadBalancer   10.23.245.116   35.230.123.32  80:32758/TCP   1m
```

If `EXTERNAL-IP` reads `<pending>` then wait a minute and try again, it needs to be provisioned by the service controller.

```
curl --connect-timeout 2 http://35.230.123.32/info
{"host": "35.230.123.32", "version": "0.5.0", "from": "10.20.1.1"}
```

There, that's better. One strange thing though - it says from "10.20.1.1", but that isn't your VM's IP address. You'll either see an address from the pod CIDR, 10.20.1/8 or from the VM, e.g. `10.138.0.1`.

This is because the load balancer is terminated at the ClusterIP which could be either node. One node has the pod, and the other node does not.

If the load balancer sends your request to the node **with** the pod, the request will immediately get routed to the node's container network (10.20.1/8) and terminate at the pod.

If the load balancer sends your request to the node **without** the pod, the request will be forwarded to the node **with** the pod. Then you will see the IP of the node e.g. `10.138.0.1`.

Alright, let's clean up:

```
kubectl delete svc simple-api
kubectl delete pod simple-api
```


## Use container probes for health checks

Every Pod resource has a `restartPolicy` that applies to all of the containers for the pod. By default, the `restartPolicy` is `Always` , which means that the kubelet on the node hosting that pod will automatically restart the pod’s containers if they exit or fail. This is the correct policy for applications that are not expected to terminate, like web applications or services like the simple-api.

It’s sometimes the case that long-running applications can end up in a problematic state without crashing, and the kubelet will fail to recognize the need to intervene. For this reason, Kubernetes allows you to definite different types of probe operations against containers, which the kubelet will execute periodically to assess container state.

### Create POD with liveness probe

Take a look at simple_liveness.yaml:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

Let's apply it:

```
kubectl apply -f simple_liveness.yaml
```

Then use kubectl get with the --watch option to list the pod details and then watch for changes. This will enable us to see updates to the status of running Pods as they are witnessed by Kubernetes.

```
kubectl get pods liveness-exec --watch -o wide
```

In the configuration file, you can see that the Pod has a single Container. The `periodSeconds` field specifies that the kubelet should perform a liveness probe every 5 seconds. The `initialDelaySeconds` field tells the kubelet that it should wait 5 second before performing the first probe. To perform a probe, the kubelet executes the command `cat /tmp/healthy` in the Container. If the command succeeds, it returns 0, and the kubelet considers the Container to be alive and healthy. If the command returns a non-zero value, the kubelet kills the Container and restarts it.

When the Container starts, it executes this command:
```bash
/bin/sh -c "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600"
```

For the first 30 seconds of the Container’s life, there is a `/tmp/healthy` file. So during the first 30 seconds, the command `cat /tmp/healthy` returns a success code. After 30 seconds, `cat /tmp/healthy` returns a failure code.

Within 30 seconds, view the Pod events:
```
kubectl describe pod liveness-exec

...
Normal  Scheduled              23s   default-scheduler                                 Successfully assigned liveness-exec to gke-kubelab1-default-pool-9e771e91-cbzt
Normal  SuccessfulMountVolume  22s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  MountVolume.SetUp succeeded for volume "default-token-4x6jb"
Normal  Pulling                22s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  pulling image "k8s.gcr.io/busybox"
Normal  Pulled                 22s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  Successfully pulled image "k8s.gcr.io/busybox"
Normal  Created                22s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  Created container
Normal  Started                22s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  Started container
```


After 35 seconds, view the Pod events again. At the bottom of the output, there are messages indicating that the liveness probes have failed, and the containers have been killed and recreated.
```
kubectl describe pod liveness-exec

...
Events:
  Type     Reason                 Age   From                                              Message
  ----     ------                 ----  ----                                              -------
  Normal   Scheduled              53s   default-scheduler                                 Successfully assigned liveness-exec to gke-kubelab1-default-pool-9e771e91-cbzt
  Normal   SuccessfulMountVolume  52s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  MountVolume.SetUp succeeded for volume "default-token-4x6jb"
  Normal   Pulling                52s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  pulling image "k8s.gcr.io/busybox"
  Normal   Pulled                 52s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  Successfully pulled image "k8s.gcr.io/busybox"
  Normal   Created                52s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  Created container
  Normal   Started                52s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  Started container
  Warning  Unhealthy              21s   kubelet, gke-kubelab1-default-pool-9e771e91-cbzt  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```

Wait another 30 seconds, and verify that the Container has been restarted:
```
kubectl get pod liveness-exec
```

The output shows that `RESTARTS` has been incremented:
```
NAME            READY     STATUS    RESTARTS   AGE
liveness-exec   1/1       Running   3          4m
```

### Define a Liveness HTTP request

Another kind of liveness probe uses an HTTP GET request. Here is the pod manifest file used in this lab:

```yaml
apiVersion: v1
  kind: Pod
  metadata:
    labels:
      test: liveness
    name: liveness-http
  spec:
    containers:
    - name: liveness
      image: k8s.gcr.io/liveness
      args:
      - /server
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
          httpHeaders:
          - name: X-Custom-Header
            value: Awesome
        initialDelaySeconds: 3
        periodSeconds: 3
```

In the configuration file, you can see that the Pod has a single Container. The periodSeconds field specifies that the kubelet should perform a liveness probe every 3 seconds. The initialDelaySeconds field tells the kubelet that it should wait 3 seconds before performing the first probe. To perform a probe, the kubelet sends an HTTP GET request to the server that is running in the Container and listening on port 8080. If the handler for the server’s /healthz path returns a success code, the kubelet considers the Container to be alive and healthy. If the handler returns a failure code, the kubelet kills the Container and restarts it.

Any code greater than or equal to 200 and less than 400 indicates success. Any other code
indicates failure.

For the first 10 seconds that the Container is alive, the /healthz handler returns a status of 200. After that, the handler returns a status of 500.

Here is the code for the server written in GOLANG

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
      duration := time.Now().Sub(started)
      if duration.Seconds() > 10 {
          w.WriteHeader(500)
          w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
      } else {
          w.WriteHeader(200)
          w.Write([]byte("ok"))
      }
})
```

The kubelet starts performing health checks 3 seconds after the Container starts. So the first couple of health checks will succeed. But after 10 seconds, the health checks will fail, and the kubelet will kill and restart the Container.

To try the HTTP liveness check, create a Pod:

```
kubectl create -f http-liveness.yaml
```

After 10 seconds, view Pod events to verify that liveness probes have failed and the Container has been restarted:
```
kubectl describe pod liveness-http
```

### Define a TCP liveness probe

A third type of liveness probe uses a TCP Socket. With this configuration, the kubelet will attempt to open a socket to your container on the specified port. If it can establish a connection, the container is considered healthy, if it can’t it is considered a failure.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

As you can see, configuration for a TCP check is quite similar to an HTTP check. This example uses both readiness and liveness probes. The kubelet will send the first readiness probe 5 seconds after the container starts. This will attempt to connect to the goproxy container on port 8080. If the probe succeeds, the pod will be marked as ready. The kubelet will continue to run this check every 10 seconds.

In addition to the readiness probe, this configuration includes a liveness probe. The kubelet will run the first liveness probe 15 seconds after the container starts. Just like the readiness probe, this will attempt to connect to the goproxy container on port 8080. If the liveness probe fails, the container will be restarted.

### Use a named port

You can use a named container port for HTTP or TCP liveness checks:
```yaml
ports:
  - name: liveness-port
    containerPort: 8080
    hostPort: 8080
  livenessProbe:
    httpGet:
      path: /healthz
      port: liveness-port
```

### Define readiness probes

Sometimes, applications are temporarily unable to serve traffic. For example, an application might need to load large data or configuration files during startup. In such cases, you don’t want to kill the application, but you don’t want to send it requests either. Kubernetes provides readiness probes to detect and mitigate these situations. A pod with containers reporting that they are not ready does not receive traffic through Kubernetes Services.

Readiness probes are configured similarly to liveness probes. The only difference is that you use the `readinessProbe` field instead of the `livenessProbe` field.


```yaml
readinessProbe:
    exec:
      command:
      - cat
      - /tmp/healthy
    initialDelaySeconds: 5
    periodSeconds: 5
```

Configuration for HTTP and TCP readiness probes also remains identical to liveness probes.

Readiness and liveness probes can be used in parallel for the same container. Using both can ensure that traffic does not reach a container that is not ready for it, and that containers are restarted when they fail.

Let's cleanup:

```
kubectl delete pod --all
```
