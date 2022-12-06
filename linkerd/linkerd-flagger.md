# Linkerd Flagger

---------------------------------------------------------------------------------------------------

## Linkerd SMI

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

## Automated Canary Releases

We can combine **traffic splitting** with Linkerd’s automatic **golden metrics** telemetry and **drive traffic decisions based on the observed metrics**.

### Install Flagger

While **Linkerd** will be managing the **actual traffic routing**, **Flagger** automates the process of creating new Kubernetes resources,
**watching metrics** and incrementally **sending users over to the new version**

add **CRD** and **RBACK** and **a controller** configured to interact with the **Linkerd control plane**:

```bash
kubectl apply -k github.com/fluxcd/flagger/kustomize/linkerd
kubectl -n linkerd rollout status deploy/flagger
```

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo
  namespace: test
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  service:
    port: 9898
  analysis:
    interval: 10s
    threshold: 5
    stepWeight: 10
    maxWeight: 100
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
```
