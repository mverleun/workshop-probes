# Kubernetes probes

Developers write great applications. But even the best applications are not always able to handle requests.

* If an application is just launched it might need to perform some initialization routines before it is ready to process requests.
* An application might be in a state where it can not continue due to external blocking factors such as database connections that have dropped, filesystems that have filled up etc.
* An application might be shutting down. This could be because of an update or of a scaling action.

Unless we help Kubernetes with readiness probes it will not know about the state of an application and assume that the app is up.

And at the same time even the best applications might run into situations that require the an app to restart. This could be the case if an application has problems recovering from a lost connection. Of if some internal routines generate impressive stacktraces that are unrecoverable.

For these situations we have liveness probes.

The readiness probe is also very important when doing rolling upgrades of applications. It will delay the removal of an old version of a pod as long as the new version is not ready.

## Prequisites

In order to experience the behaviour of resource limits the following setup is expected:

* minikube running local with 4 Cores and 8GiB of memory. (`minikube start --memory=8G --cpus=4`)
* minikube metrics-server enabled (`minikube addons enable metrics-server`)
* network connectivity to download images

> Create the minikube cluster if you haven't done it yet. Once it is started open a terminal and enter the command `kubectl get events --watch` to receive constant updates on what is happening inside the cluster.

## Probe sequence

Liveness probes and readiness probes have been around for quite some time. They have one short coming and that is how to deal with slow starting pods.
For slow starting pods the startup probe is introduced. It requires Kubernetes version 1.18 or above.
The startup probe waits for a condition to occur and then passes on control to the readiness and liveness probes.

``` text
                    |     
                    |       
                    V       
               +---------+ 
               | Startup | 
               | probe   | 
               +---------+ 
                    |       
                    |
                    /\
                   /  \
                  /    \
                 /      \
                /        \
               /          \
              /            \
             /              \
            |                |
    +-------+                +--------+
    |       |                |        |
    |       V                V        |
    |  +----------+    +-----------+  |
    |  | Liveness |    | Readiness |  |
    |  | probe    |    | probe     |  |
    |  +----------+    +-----------+  |
    |       |                |        |
    +-------+                +--------+
```

## Services without (ready) pods

Let's prepare our environment a bit and experience what might happen if there is no pod to handle requests.

Deploy the service definition. This definition will listen on port 8080 and forward the traffic to one or more upstream pods.

``` bash
❯ kubectl apply -f 01-service-for-webserver.yaml
service/apache created
```

In a seperate terminal create a tunnel into the minikube cluster:

``` bash
❯ minikube tunnel
🏃  Starting tunnel for service apache.
```

Keep this session running!

Have a closer look at this service and notice the endpoints. Do this from another terminal:

``` bash
❯ kubectl apply -f 01-service-for-webserver.yaml
service/apache created

❯ kubectl describe service apache
Name:                     apache
Namespace:                default
Labels:                   app=apache
Annotations:              <none>
Selector:                 app=apache
Type:                     LoadBalancer
IP Families:              <none>
IP:                       10.97.208.116
IPs:                      10.97.208.116
Port:                     web  8080/TCP
TargetPort:               80/TCP
NodePort:                 web  31718/TCP
Endpoints:                <none>
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason  Age    From                Message
  ----    ------  ----   ----                -------
  Normal  Type    4m20s  service-controller  ClusterIP -> LoadBalancer
```

There are no endpoints active yet. You can check this by opening the url <http://localhost:8080/> with a browser, postman, insomnia, curl or wget after opening a port forward. Note the ampersand at the end.

> Some browsers will automatically redirect you from a http to a https session. If that happens try to navigate to <http://127.0.0.1:8080/> instead.

```bash
# Port forward first
kubectl port-forward service/apache 8080 &


❯ curl <http://localhost:8080/>
curl: (7) Failed to connect to localhost port 8080: Connection refused
```

## A deployment without probes

Deploy 3 pods that will receive traffic from the service that was deployed recently and check the status of the deployment and the service:

```bash
❯ kubectl apply -f 02-webserver-without-probes.yaml
deployment.apps/apache created

❯ kubectl describe deployments.apps apache
Name:                   apache
... omitted
Selector:               app=apache
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
... omitted

❯ kubectl describe service apache
Name:              apache
... omitted
Selector:          app=apache
... omitted
Port:              web  8080/TCP
TargetPort:        80/TCP
Endpoints:         172.17.0.2:80
Session Affinity:  None
Events:            <none>
```

Notice that the service shows 1 endpoint which matches the status of the replicas ( 1 available ).

Feel free to validate that it actually works, you might have to reopen the port forward again:

``` bash
❯ curl http://localhost:8080/
Handling connection for 8080
<html><body><h1>It works!</h1></body></html>
```

> Due to the way the port-forwarding works we will not rely on it for futher testing.

Cleanup:

``` bash
❯ kubectl delete -f 02-webserver-without-probes.yaml
deployment.apps "apache" deleted
```

## A deployment with probes

Again deploy 3 pods. This time with probes.

Before continuing enter the following command in a seperate terminal to see all the events, keep this running:

``` bash
❯ kubectl get events --watch
LAST SEEN   TYPE      REASON                         OBJECT                         MESSAGE
10m         Normal    Scheduled                      pod/apache-598f4c9bf9-757z8    Successfully assigned default/apache-598f4c9bf9-757z8 to minikube
9m59s       Normal    Pulling                        pod/apache-598f4c9bf9-757z8    Pulling image "httpd"
9m56s       Normal    Pulled                         pod/apache-598f4c9bf9-757z8    Successfully pulled image "httpd" in 3.40699297s
9m56s       Normal    Created                        pod/apache-598f4c9bf9-757z8    Created container httpd
9m54s       Normal    Started                        pod/apache-598f4c9bf9-757z8    Started container httpd
3m23s       Normal    Killing                        pod/apache-598f4c9bf9-757z8    Stopping container httpd
42m         Normal    Killing                        pod/apache-598f4c9bf9-n5t26    Stopping container httpd
... omitted
```

Have a look at the service again. If there are endpoints the cleanup of the deployment is not completed yet.

``` bash
❯ kubectl describe service apache
Name:              apache
... omitted
TargetPort:        80/TCP
Endpoints:         <none>
... omitted
```

> The following steps are timing sensitive. If not performed quick enough the results can be different.

Next deploy pods with probes, observe that the service will _not_ get any endpoints yet because the pods remain in a not ready state forever.

``` bash
❯ kubectl apply -f 03-webserver-with-probes.yaml
deployment.apps/apache created
❯ kubectl describe service apache
Name:              apache
... omitted
Endpoints:
... omitted
```

Also observe the events, there are many messages about failed Readiness probes and Liveness probes.

Let's make a pod ready. We need the name of a pod so let's create a list with names and pick the top one for starters.

``` bash
❯ kubectl get pods
NAME                    READY   STATUS              RESTARTS   AGE
apache-c4fb9756-6bgmk   0/1     ContainerCreating   0          5s
apache-c4fb9756-bhqmt   0/1     ContainerCreating   0          5s
apache-c4fb9756-q8dtw   0/1     Running             0          5s
```

> The pod name `apache-c4fb9756-6bgmk` in the following examples has to be replaced with the name shown on your screen.

Let's satisfy the Readiness Probe first:

``` bash
❯ kubectl exec apache-c4fb9756-6bgmk -- touch /usr/local/apache2/htdocs/readiness
```

In the event log you should see a line like this:  `61m         Normal    Type                           service/apache                 LoadBalancer -> ClusterIP`

Let's check the status:

``` bash
❯ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
apache-c4fb9756-6bgmk   1/1     Running   0          2m7s
apache-c4fb9756-bhqmt   0/1     Running   0          2m7s
apache-c4fb9756-q8dtw   0/1     Running   0          2m7s

❯ kubectl describe deployments.apps apache
Name:                   apache
... onmitted
Replicas:               3 desired | 3 updated | 3 total | 1 available | 2 unavailable
... omitted

❯ kubectl describe service apache
Name:              apache
... omitted
Endpoints:         172.17.0.2:80
... omitted
```

If you wait a while you'll notice that the pods restart every 150 seconds and that after a restart the readiness probe of the first pod failes again.

After restarting the file that affects the readiness probes is gone causing the readiness probe to fail again.

Let's satisfy all the liveness probes, create a file in all three pods. You have to replace the podnames to match yours:

``` bash
❯ kubectl exec apache-c4fb9756-6bgmk -- touch /usr/local/apache2/htdocs/liveness
❯ kubectl exec apache-c4fb9756-bhqmt -- touch /usr/local/apache2/htdocs/liveness
❯ kubectl exec apache-c4fb9756-q8dtw -- touch /usr/local/apache2/htdocs/liveness
```

If you experience problems it could be that there is a 'backoff timeout' active. You'll have to try again and again until your command is excepted.

By now the liveness probe is no longer failing. The event log will no longer report liveness probe related issues, but our servce is still not working:

``` bash
❯ kubectl describe service apache
Name:              apache
... omitted
TargetPort:        80/TCP
Endpoints:
... omitted
```

Let's fix it, one pod at a time:

``` bash
❯ kubectl exec apache-c4fb9756-6bgmk -- touch /usr/local/apache2/htdocs/readiness

❯ kubectl describe service apache
Name:              apache
... omitted
Endpoints:         172.17.0.2:80
... omitted

❯ kubectl exec apache-c4fb9756-bhqmt -- touch /usr/local/apache2/htdocs/readiness

❯ kubectl describe service apache
Name:              apache
... omitted
Endpoints:         172.17.0.2:80,172.17.0.4:80
... omitted
```

That's two out of three...

Cleanup:

``` bash
❯ kubectl delete -f 03-webserver-with-probes.yaml
deployment.apps "apache" deleted

❯ kubectl delete -f 01-service-for-webserver.yaml
service "apache" deleted
```

## But what about deployments that have no (usable) http endpoint?

Let's use scripts... We deploy a pod which gets two script from a configMap. By default the pod will be ready and alive. But we can change this of course.

In a seperate terminal enter a command:

``` bash
kubectl get pods -w
NAME                      READY   STATUS    RESTARTS   AGE
probes-76d848bdbb-4ddq2   1/1     Running   0          98s
```

In another trigger the readiness probes that is in the script:

``` bash
❯ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
probes-76d848bdbb-4ddq2   0/1     Running   5          9m20s

❯ kubectl exec probes-76d848bdbb-4ddq2 -- touch /tmp/notready
```

Your pod name is different. As you can see this pod is in a not ready state and will remain there for a very long time if we do not intervene.

Make the pod ready again:

``` bash
❯ kubectl exec probes-76d848bdbb-4ddq2 -- rm /tmp/notready
```

Let's crash the pod:

``` bash
kubectl exec probes-76d848bdbb-4ddq2 -- touch /tmp/notalive
```

Now it is a matter of time before the pod will restart.

## Summary

There are several probes that play an important role in Kubernetes. One determines if a restart of a pod is required. The other one if the pod is able to perform its duties.
These probes can be configured to use http requests or shell scripts to perform these tasks.
The readiness probe also determines the pace of a rolling update. As soon as a pod reports itself as 'ready' the system will proceed with the next pod.
