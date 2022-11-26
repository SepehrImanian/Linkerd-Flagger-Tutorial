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

> The packet will arrive on the **PREROUTING chain** and will be immediately routed to the **redirect chain**. If its destination port matches any of the **inbound ports to skip**, then it will be **forwarded directly to the application process**, bypassing the proxy., Redirection is done by changing the incoming packet’s **destination header**, the target port will be **replaced with 4143, which is the proxy’s inbound port**. ,The proxy will process the packet and **produce a new one** that will be **forwarded to the service**; it will be able to get **the original target (IP:PORT) of the inbound packet** by using a **special socket option SO_ORIGINAL_DST** ,The new packet will be routed through the **OUTPUT chain**, from there it will be **sent to the application**.

![iptables Inbound connections](./images/iptables-Inbound-connections.png)

### Outbound connections

When a **packet leaves a pod**, it will first traverse the **OUTPUT chain**, the first **default chain an outgoing packet traverses in the nat table**.

**To redirect outgoing packets** through the outbound side of the proxy, the sidecar container will again **create a new chain**

any packet that traverses the **OUTPUT chain** should be **forwarded to** our **PROXY_INIT_OUTPUT chain** to be processed.

* **ignore-proxy-uid:** any packets **owned by the proxy** (whose user id is 2102), will **skip processing and return to** the previous **(OUTPUT) chain**.
* **ignore-loopback:** if the packet is sent over the **loopback interface (lo)**, it will **skip processing and return to the previous chain**.
* **ignore-port:** will ignore processing packets whose **destination ports** are included in the **skip-outbound-ports** install option.
* **redirect-all-outgoing:** the last rule in the chain, it will **redirect all outgoing TCP packets to port 4140**, the **proxy’s outbound port**

> the **packet’s header** (**SO_ORIGINAL_DST**) will be **re-written** to target the **outbound port**. The proxy will process the packet and then forward it to its destination

![iptables Outbound connections](./images/iptables-outbound-connections.png)

### Rules table
