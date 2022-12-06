# Linkerd Practical

## Add services to Linkerd

services that communicate using certain **non-HTTP** protocols (including MySQL, SMTP, Memcache, and others)

Meshing a Kubernetes resource is typically done by annotating the resource (or its namespace) with this Kubernetes annotation.
This annotation triggers automatic **proxy injection** when the resources are **created or updated**.

```bash
linkerd.io/inject: enabled
```

Linkerd provides a **linkerd inject** text transform command will add this annotation to a given Kubernetes manifest.

* Adding the annotation to existing pods does not automatically mesh them. For existing pods, after adding the annotation you will also need to **recreate or update**
  the resource (e.g. by using **kubectl rollout restart** to perform a rolling update) to trigger proxy injection.

```bash
cat deployment.yml | linkerd inject - | kubectl apply -f - # deployment

kubectl get -n NAMESPACE deploy -o yaml | linkerd inject - | kubectl apply -f -# namespace
```

to verify that your services have been added to the mesh

```bash
kubectl -n NAMESPACE get po -o jsonpath='{.items[0].spec.containers[*].name}'
```
---------------------------------------------------------------------------------------------------

## Distributed tracing with Linkerd

Unlike most features of a service mesh, distributed tracing requires modifying the **source of your application**.

```
linkerd jaeger install | kubectl apply -f -

linkerd jaeger check

linkerd jaeger dashboard
```

---------------------------------------------------------------------------------------------------

## Debugging HTTP applications with per-route metrics

* **Service Profiles**

**Service profiles** provide Linkerd with some additional information about your services.
These define the **routes** that you’re serving and, among other things, allow for the **collection of metrics on a per route** basis.

One of the easiest ways to get **service profiles setup** is by using existing **OpenAPI (Swagger)** specs.

```bash
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/booksapp/webapp.swagger \
  | linkerd -n booksapp profile --open-api - webapp \
  | kubectl -n booksapp apply -f -
```

Each live request will show up with what **:authority** or Host header is being seen as well as the **:path** and **rt_route** being used.
his will watch all the **live requests** flowing through webapp

```bash
linkerd viz tap -n booksapp deploy/webapp -o wide | grep req
```

```
req id=9:1 proxy=out src=10.233.66.150:60106 dst=10.233.66.149:7002 tls=true :method=GET :authority=books:7002 :path=/books/10117.json src_control_plane_ns=linkerd src_deployment=webapp src_namespace=booksapp src_pod=webapp-99c5fb779-kkdfc src_pod_template_hash=99c5fb779 src_serviceaccount=webapp src_tls=loopback dst_control_plane_ns=linkerd dst_deployment=books dst_namespace=booksapp dst_pod=books-544fd57755-hgpd6 dst_pod_template_hash=544fd57755 dst_server_id=books.booksapp.serviceaccount.identity.linkerd.cluster.local dst_service=books dst_serviceaccount=books dst_tls=true rt_route=GET /books/{id}.json
```

To see the metrics that have accumulated:
The [DEFAULT] route is a catch all for anything that does not match the service profile.
```bash
linkerd viz -n booksapp routes svc/webapp
```

```
ROUTE                       SERVICE   SUCCESS      RPS   LATENCY_P50   LATENCY_P95   LATENCY_P99
GET /                        webapp   100.00%   0.5rps          56ms          96ms          99ms
GET /authors/{id}            webapp   100.00%   0.5rps          27ms          39ms          40ms
POST /authors/{id}/delete    webapp   100.00%   0.5rps          46ms          91ms          98ms
POST /authors/{id}/edit      webapp         -        -             -             -             -
POST /books                  webapp    52.46%   2.0rps          33ms          87ms          98ms
POST /books/{id}/edit        webapp    46.15%   1.1rps          50ms          95ms          99ms
[DEFAULT]                    webapp         -        -             -             -             -

```

Profiles can be used to observe **outgoing requests** as well as **incoming requests**.

```
linkerd viz -n booksapp routes deploy/webapp --to svc/books
```
* **Retries**

**Add Retries for increase success rate of requests**

**This improves success rate, at the cost of additional latency**

```
kubectl -n booksapp edit sp/authors.booksapp.svc.cluster.local
```

add isRetryable to a specific route:
```yaml
spec:
  routes:
  - condition:
      method: HEAD
      pathRegex: /authors/[^/]*\.json
    name: HEAD /authors/{id}.json
    isRetryable: true ### ADD THIS LINE ###
```

Linkerd will begin to retry requests to this route automatically , to see :

```
linkerd viz -n booksapp routes deploy/books --to svc/authors -o wide
```

* **Timeouts**

Linkerd can limit how long to wait before failing outgoing requests to another service.

let’s set a 25ms timeout for calls to that route. Your latency numbers will vary depending on the characteristics of your cluster.
```
kubectl -n booksapp edit sp/books.booksapp.svc.cluster.local
```
maximum amount of time a REST client would wait for a response.
Linkerd will now return errors to the webapp REST client when the timeout is reached.

```yaml
spec:
  routes:
  - condition:
      method: PUT
      pathRegex: /books/[^/]*\.json
    name: PUT /books/{id}.json
    timeout: 25ms ### ADD THIS LINE ###
```

```
linkerd viz -n booksapp routes deploy/webapp --to svc/books -o wide
```

---------------------------------------------------------------------------------------------------

## Injecting Faults

```
linkerd viz -n booksapp stat deploy
```

When Linkerd sees traffic going to the books service, it will send 9⁄10 requests to the original service and 1⁄10 to the error injector.

```yaml
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: error-split
  namespace: booksapp
spec:
  service: books
  backends:
  - service: books
    weight: 900m
  - service: error-injector
    weight: 100m
```

this routes command filters to all the requests being issued by webapp destined for the books service itself.

```
linkerd viz -n booksapp routes deploy/webapp --to service/books
```

---------------------------------------------------------------------------------------------------

## Ingress traffic

Linkerd doesn’t provide a built-in ingress. Instead, Linkerd is designed to work with existing Kubernetes ingress solutions.

Combining Linkerd and your ingress solution requires two things:

* Configuring your ingress to support Linkerd. ==> **annotations header**

* Meshing your ingress pods so that they have the Linkerd proxy installed. ==> **add to linkerd**
  Meshing your ingress pods will allow Linkerd to provide features like **L7 metrics** and **mTLS**
  the moment the traffic is inside the cluster


* **Nginx**

```yaml
  annotations:
    nginx.ingress.kubernetes.io/service-upstream: "true"
```
* **Traefik**

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: l5d-header-middleware
  namespace: traefik
spec:
  headers:
    customRequestHeaders:
      l5d-dst-override: "web-svc.emojivoto.svc.cluster.local:80"
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  annotations:
    kubernetes.io/ingress.class: traefik
  creationTimestamp: null
  name: emojivoto-web-ingress-route
  namespace: emojivoto
spec:
  entryPoints: []
  routes:
  - kind: Rule
    match: PathPrefix(`/`)
    priority: 0
    middlewares:
    - name: l5d-header-middleware
    services:
    - kind: Service
      name: web-svc
      port: 80
```

**linkerd figure out from header such as l5d-dst-override, Host, or :authority**

---------------------------------------------------------------------------------------------------

## Troubleshooting

This section provides resolution steps for **common problems reported** with the linkerd check command


[linkerd Troubleshooting](https://linkerd.io/2.12/tasks/troubleshooting/#l5d-existence-crb)


---------------------------------------------------------------------------------------------------

## Graceful Pod Shutdown

When Kubernetes begins to terminate a pod, it starts by sending all containers in that pod a **TERM signal**.
When the **Linkerd proxy sidecar** receives this signal, it will immediately begin a **graceful shutdown** where 
it refuses all new requests and allows existing requests to complete before shutting down.

This means that if the pod’s main container attempts to make any new network calls **after the proxy has received the TERM signal**,
**those network calls will fail**.

Kubernetes will wait **30 seconds** to allow processes to handle the **TERM signal**.
This is known as the grace period within which a process may shut itself down gracefully.
If the grace period time runs out, and the process hasn’t gracefully exited,
the container runtime will send a **KILL signal**, abruptly stopping the process.

**container runtime sends the TERM signal**

Kubernetes also allows **operators of services to define lifecycle hooks** for their containers.
Important in the context of **graceful shutdown** is the **preStop hook**, that will be called when a container is terminated due to:

* An API request.
* Liveness/Readiness probe failure.
* Resource contention.

Configuration options for graceful shutdown:

* **--wait-before-seconds** (install time cli, helm)

This will add a preStop hook to the proxy container to delay its handling of the TERM signal.

* **config.linkerd.io/shutdown-grace-period**

is an annotation that can be used on workloads to configure the graceful shutdown time for the proxy.
the shutdown grace period is 120 seconds.

* **linkerd-await**

The await binary can be used with a **--shutdown** option, in which case,
after the process it has wrapped finished, it will send a shutdown request to the proxy.

[github linkerd-await](https://github.com/linkerd/linkerd-await)


Before Kubernetes terminates a pod, it first removes that pod from the **endpoints resource** of any services that pod is a member of.
This means that clients of that service should stop sending traffic to the pod before it is terminated. However ,certain clients can be
**slow to receive the endpoints update** and may attempt to send requests to the terminating pod after that pod’s proxy has already received
the **TERM signal** and begun graceful shutdown. Those requests will **fail**.


delay the **Linkerd proxy’s** handling of the **TERM signal** for a given number of seconds using a **preStop hook**
```
linkerd inject --wait-before-exit-seconds
```

```yaml
   # application container
    lifecycle:
      preStop:
        exec:
          command:
            - /bin/bash
            - -c
            - sleep 20

# for entire pod
terminationGracePeriodSeconds: 160
```

**pod container preStop time < proxy sidecar time < terminationGracePeriodSeconds (configured for the entire pod)**

---------------------------------------------------------------------------------------------------

## Configuring Proxy Concurrency

The Linkerd data plane’s **proxies are multithreaded**, and are capable of running a variable number
of worker threads so that their resource usage matches the application workload.


the primary method of tuning proxy resource usage is **limiting the number of worker threads** used by the proxy to forward traffic. 

### There are multiple methods for doing this:

* **Using the proxy-cpu-limit Annotation**

When the environment variable configured by the **proxy-cpu-limit annotation** is unset,
the proxy will run a number of **worker threads equal to the number of CPU cores available**.

```bash
linkerd install --proxy-cpu-limit 2 | kubectl apply -f -
```

```bash
kind: Deployment
apiVersion: apps/v1
metadata:
  name: my-deployment
  # ...
spec:
  template:
    metadata:
      annotations:
        config.linkerd.io/proxy-cpu-limit: '1'
```

Unlike Kubernetes CPU limits and requests, which can be expressed in milliCPUs,
the **proxy-cpu-limit annotation** should be expressed in **whole numbers of CPU cores**.
Fractional values will be rounded up to the nearest whole number.

---------------------------------------------------------------------------------------------------

## Proxy Configuration

Linkerd provides a set of **annotations** that can be used to **override the data plane proxy’s configuration**.

[proxy configuration annotations list](https://linkerd.io/2.12/reference/proxy-configuration/#)

```bash
spec:
  template:
    metadata:
      annotations:
        config.linkerd.io/proxy-cpu-limit: "1"
        config.linkerd.io/proxy-cpu-request: "0.2"
        config.linkerd.io/proxy-memory-limit: 2Gi
        config.linkerd.io/proxy-memory-request: 128Mi
```

### Ingress Mode

it will **route requests** based on their **:authority, Host, or l5d-dst-override headers** instead of **their original destination**.

The proxy can be made to run in ingress mode by used the **linkerd.io/inject: ingress** annotation
rather than the default **linkerd.io/inject: enabled** annotation. This can also be done with 
the **--ingress** flag in the inject CLI command:

```bash
kubectl get deployment <ingress-controller> -n <ingress-namespace> -o yaml | linkerd inject --ingress - | kubectl apply -f -
```

---------------------------------------------------------------------------------------------------

## Using the Debug Sidecar

Linkerd provides a debug sidecar with some helpful tooling. Similar to how proxy sidecar injection works.

you add a **debug sidecar** to a pod by setting the **config.linkerd.io/enable-debug-sidecar: "true"** annotation at **pod creation time**.
For convenience, the **linkerd inject** command provides an **--enable-debug-sidecar** option that does this annotation for you.

> The debug sidecar image contains **tshark, tcpdump, lsof, and iproute2**. Once installed, it starts automatically logging all incoming and outgoing traffic with tshark, which can then be viewed with **kubectl logs**. Alternatively, you can use **kubectl exec** to access the container and run commands directly.

```
kubectl -n emojivoto get deploy/voting -o yaml \
  | linkerd inject --enable-debug-sidecar - \
  | kubectl apply -f -
```

inspect the HTTP headers of the requests, you could run something like this:

```
kubectl -n emojivoto exec -it \
  $(kubectl -n emojivoto get pod -l app=voting-svc \
    -o jsonpath='{.items[0].metadata.name}') \
  -c linkerd-debug -- tshark -i any -f "tcp" -V -Y "http.request"

kubectl -n emojivoto exec -it \
  $(kubectl -n emojivoto get pod -l app=voting-svc \
   -o jsonpath='{.items[0].metadata.name}') \
   -c linkerd-debug -- tshark -i any -f "tcp" -V \
   -Y "(tcp.srcport == 4143 and tcp.dstport == 50416) or tcp.port == 8080"
```








































