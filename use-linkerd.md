### Add services to Linkerd

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

### Linkerd SMI

Linkerd supports **SMI’s TrafficSplit** specification which can be used to perform **traffic splitting across services** natively

```bash
curl -sL https://linkerd.github.io/linkerd-smi/install | sh
linkerd smi install | kubectl apply -f -
linkerd smi check
```

```yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: backend-split
  namespace: trafficsplit-sample
spec:
  service: backend-svc
  backends:
  - service: backend-svc
    weight: 500
  - service: failing-svc
    weight: 500
```

---------------------------------------------------------------------------------------------------

### Distributed tracing with Linkerd

Unlike most features of a service mesh, distributed tracing requires modifying the **source of your application**.

```
linkerd jaeger install | kubectl apply -f -

linkerd jaeger check

linkerd jaeger dashboard
```

---------------------------------------------------------------------------------------------------

### Debugging HTTP applications with per-route metrics

* **Service Profiles**

**Service profiles** provide Linkerd with some additional information about your services.
These define the **routes** that you’re serving and, among other things, allow for the **collection of metrics on a per route** basis.

One of the easiest ways to get **service profiles setup**is by using existing **OpenAPI (Swagger)** specs.

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

### Injecting Faults

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

### Authorization Policy

#### Server

A Server selects a port on a set of pods in the same namespace as the server.
While the Server resource is similar to a **Kubernetes Service**

When a Server selects a port, all traffic to that port is then **denied by default**, regardless of the default policy.
Thus, to authorize traffic to a port selected by a Server, you must create **AuthorizationPolicies**.


Server Spec:

* **podSelector:** 
A podSelector selects pods in the same namespace.
* **port**
A port name or number. Only ports in a pod spec’s ports are considered.
* **proxyProtocol:**
Configures protocol discovery for inbound connections. Supersedes the **config.linkerd.io/opaque-ports** annotation. 
Must be one of **unknown,HTTP/1,HTTP/2,gRPC,opaque,TLS**. **Defaults to unknown** if not set.


podSelector:

* **matchExpressions:**
matchExpressions is a list of label selector requirements. The requirements are ANDed.

* **matchLabels:**
A Server that selects over pods with a specific label, with gRPC as the proxyProtocol.

```yaml
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  namespace: emojivoto
  name: emoji-grpc
spec:
  podSelector:
    matchLabels:
      app: emoji-svc
  port: grpc
  proxyProtocol: gRPC
```

A Server that selects over pods with matchExpressions, with HTTP/2 as the proxyProtocol, on port 8080.

```yaml
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  namespace: emojivoto
  name: backend-services
spec:
  podSelector:
    matchExpressions:
    - {key: app, operator: In, values: [voting-svc, emoji-svc]}
    - {key: environment, operator: NotIn, values: [dev]}
  port: 8080
  proxyProtocol: "HTTP/2"
```

#### HTTPRoute

* An HTTPRoute represents a subset of traffic handled by a **Server**.
* HTTPRoutes are “attached” to Servers and have match rules which determine which requests match.
  Matches can be based on path, headers, query params, and/or verb.
* **AuthorizationPolicies** may target HTTPRoute resources, thereby authorizing traffic to that HTTPRoute only rather than to the entire Server.


```yaml
apiVersion: policy.linkerd.io/v1beta1
kind: HTTPRoute
metadata:
  name: authors-get-route
  namespace: booksapp
spec:
  parentRefs:
    - name: authors-server
      kind: Server
      group: policy.linkerd.io
  rules:
    - matches:
      - path:
          value: "/authors.json"
        method: GET
      - path:
          value: "/authors/"
          type: "PathPrefix"
        method: GET
```


#### AuthorizationPolicy

* An **AuthorizationPolicy** provides a way to authorize traffic to a **Server** or an **HTTPRoute**. 
* **AuthorizationPolicies** are a replacement for **ServerAuthorizations** which are more flexible because 
  they can target **HTTPRoutes** instead of only being able to target **Servers**.


An AuthorizationPolicy which authorizes clients that satisfy the authors-get-authn authentication to send to the authors-get-route **HTTPRoute**.

```yaml
apiVersion: policy.linkerd.io/v1alpha1
kind: AuthorizationPolicy
metadata:
  name: authors-get-policy
  namespace: booksapp
spec:
  targetRef:
    group: policy.linkerd.io
    kind: HTTPRoute
    name: authors-get-route
  requiredAuthenticationRefs:
    - name: authors-get-authn
      kind: MeshTLSAuthentication
      group: policy.linkerd.io
```

An AuthorizationPolicy which authorizes the webapp ServiceAccount to send to the authors **Server**.

```yaml
apiVersion: policy.linkerd.io/v1alpha1
kind: AuthorizationPolicy
metadata:
  name: authors-policy
  namespace: booksapp
spec:
  targetRef:
    group: policy.linkerd.io
    kind: Server
    name: authors
  requiredAuthenticationRefs:
    - name: webapp
      kind: ServiceAccount
```

An AuthorizationPolicy which authorizes the webapp ServiceAccount to send to all policy “targets” within the booksapp **namespace**.

```yaml
apiVersion: policy.linkerd.io/v1alpha1
kind: AuthorizationPolicy
metadata:
  name: authors-policy
  namespace: booksapp
spec:
  targetRef:
    kind: Namespace
    name: booksapp
  requiredAuthenticationRefs:
    - name: webapp
      kind: ServiceAccount
```

#### MeshTLSAuthentication


* A MeshTLSAuthentication represents a **set of mesh identities**.
* When an **AuthorizationPolicy** has a MeshTLSAuthentication as one of its **requiredAuthenticationRefs**, this means that clients must be
  in the mesh and must have one of the specified identities in order to be authorized to send to the target.


A MeshTLSAuthentication which authenticates the books and webapp mesh identities.

```yaml
apiVersion: policy.linkerd.io/v1alpha1
kind: MeshTLSAuthentication
metadata:
  name: authors-get-authn
  namespace: booksapp
spec:
  identities:
    - "books.booksapp.serviceaccount.identity.linkerd.cluster.local"
    - "webapp.booksapp.serviceaccount.identity.linkerd.cluster.local"
```

A MeshTLSAuthentication which authenticate thes books and webapp mesh identities.
This is an alternative way to specify the same thing as the above example.

```yaml
apiVersion: policy.linkerd.io/v1alpha1
kind: MeshTLSAuthentication
metadata:
  name: authors-get-authn
  namespace: booksapp
spec:
  identityRefs:
    - kind: ServiceAccount
      name: books
    - kind: ServiceAccount
      name: webapp
```

#### NetworkAuthentication

* A NetworkAuthentication represents a **set of IP subnets**.
* When an **AuthorizationPolicy** has a NetworkAuthentication as one of its **requiredAuthenticationRefs**,
  this means that clients must be in one of the specified networks in order to be authorized to send to the target.

```yaml
apiVersion: policy.linkerd.io/v1alpha1
kind: NetworkAuthentication
metadata:
  name: cluster-network
  namespace: booksapp
spec:
  networks:
  - cidr: 10.0.0.0/8
  - cidr: 100.64.0.0/10
  - cidr: 172.16.0.0/12
  - cidr: 192.168.0.0/16
```

#### ServerAuthorization

* A ServerAuthorization provides a way to authorize traffic to **one or more Servers**.

*  AuthorizationPolicy is a more flexible alternative to ServerAuthorization that can target HTTPRoutes as well as Servers.
   Use of **AuthorizationPolicy is preferred**, and ServerAuthorization will be **deprecated in future releases.


A ServerAuthorization that allows meshed clients with *.emojivoto.serviceaccount.identity.linkerd.cluster.local proxy identity i.e. 
all service accounts in the emojivoto namespace.

```yaml
apiVersion: policy.linkerd.io/v1beta1
kind: ServerAuthorization
metadata:
  namespace: emojivoto
  name: emoji-grpc
spec:
  # Allow all authenticated clients to access the (read-only) emoji service.
  server:
    selector:
      matchLabels:
        app: emoji-svc
  client:
    meshTLS:
      identities:
        - "*.emojivoto.serviceaccount.identity.linkerd.cluster.local"
```

A ServerAuthorization that allows any **unauthenticated** clients.

```yaml
apiVersion: policy.linkerd.io/v1beta1
kind: ServerAuthorization
metadata:
  namespace: emojivoto
  name: web-public
spec:
  server:
    name: web-http
  # Allow all clients to access the web HTTP port without regard for
  # authentication. If unauthenticated connections are permitted, there is no
  # need to describe authenticated clients.
  client:
    unauthenticated: true
    networks:
      - cidr: 0.0.0.0/0
      - cidr: ::/0
```
A ServerAuthorization that allows meshed clients with a specific service account.

```yaml
apiVersion: policy.linkerd.io/v1beta1
kind: ServerAuthorization
metadata:
  namespace: emojivoto
  name: prom-prometheus
spec:
  server:
    name: prom
  client:
    meshTLS:
      serviceAccounts:
        - namespace: linkerd-viz
          name: prometheus
```

---------------------------------------------------------------------------------------------------

### Ingress traffic

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

**likerd figure out from header such as l5d-dst-override, Host, or :authority**

---------------------------------------------------------------------------------------------------

### Troubleshooting

This section provides resolution steps for **common problems reported** with the linkerd check command


[linkerd Troubleshooting](https://linkerd.io/2.12/tasks/troubleshooting/#l5d-existence-crb)


---------------------------------------------------------------------------------------------------

### Graceful Pod Shutdown

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
the **TERM signal** and begun graceful shutdown. Those requests will **fail**.4


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

### Restricting Access To Services

Linkerd policy resources can be used to restrict which clients may access a service. 






























































