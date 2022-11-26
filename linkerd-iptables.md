## Linkerd Iptables

**init container** is used to set up **iptables rules** at the **start of an injected pod’s lifecycle**.
**linkerd-init** will create two chains in the nat table: **PROXY_INIT_REDIRECT**, and **PROXY_INIT_OUTPUT**

### Inbound connections

When a **packet arrives in a pod**, it will typically be processed by the **PREROUTING chain**, a **default chain attached to the nat table**.
The **sidecar container** will create a new chain to process **inbound packets**, called **PROXY_INIT_REDIRECT**.
The **sidecar container** creates **a rule (install-proxy-init-prerouting)** to send packets from the **PREROUTING chain to our redirect chain**

The **redirect chain** will be configured with two more rules:

* **ignore-port:** will ignore processing packets whose destination ports are included in the **skip-inbound-ports** install option.
* **proxy-init-redirect-all:** will redirect all incoming TCP packets through the proxy, on port **4143**.

> The packet will arrive on the **PREROUTING chain** and will be immediately routed to the **redirect chain**. If its destination port matches any of the **inbound ports to skip**, then it will be f**orwarded directly to the application process**, bypassing the proxy., Redirection is done by changing the incoming packet’s **destination header**, the target port will be **replaced with 4143, which is the proxy’s inbound port**.

![iptables Inbound connections](./images/iptables-Inbound-connections.png)

### Outbound connections
### Rules table
