# Linkerd Architecture

## Architecture

* **Control Plane**

  * **Destination Service:**
    It is used to fetch **service discovery** information to fetch policy information about which types of requests are allowed; to fetch **service profile** information used to inform **per-route metrics, retries, and timeouts**; 
  * **Identity Service:**
    The identity service acts as a **TLS Certificate Authority** that accepts CSRs from proxies and returns signed certificates. These certificates are issued at proxy initialization time and are used for proxy-to-proxy connections to implement mTLS.
  * **Proxy Injector:**
    The proxy injector is a Kubernetes admission controller that receives a **webhook** request every time a **pod is created**. This injector inspects resources for a **Linkerd-specific annotation** (**linkerd.io/inject: enabled**). When that annotation exists, the injector mutates the pod’s specification and adds the **proxy-init** and **linkerd-proxy** containers to the pod, along with the relevant start-time configuration

* **Data Plane**

comprises ultralight micro-proxies which are deployed as **sidecar containers inside application pods**
These proxies transparently intercept TCP connections to and from each pod, thanks to **iptables** rules put in place by the **linkerd-init** (or, alternatively, by **Linkerd’s CNI plugin**).

   * **Proxy**
The Linkerd2-proxy is an ultralight, transparent micro-proxy written in Rust.
designed specifically for the service mesh use case and is not designed as a general-purpose proxy.

   * **Linkerd-init Container**
The linkerd-init container is added to each meshed pod as a Kubernetes **init container** that runs before any other containers are started. It uses **iptables** to route all TCP traffic to and from the pod through the proxy. Linkerd’s init container can be run in different modes which determine what **iptables variant** is used.


![mtls](./../images/control-plane.png)

