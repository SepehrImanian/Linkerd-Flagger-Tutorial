# Insrall Linkerd Plan

## Install Linkerd

```
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
linkerd version
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd check
```
---------------------------------------------------------------------------------------------------

## Automatically Rotating Webhook TLS Credentials


The **Linkerd control plane** contains several components, called **webhooks**, which are called **directly by Kubernetes** itself.

The **traffic from Kubernetes to the Linkerd webhooks** is secured with TLS and therefore each of the webhooks requires a secret containing TLS credentials.
These certificates are different from the ones that the Linkerd proxies use to secure pod-to-pod communication and use a completely separate trust chain.

when Linkerd is installed, TLS credentials are automatically generated for all of the webhooks, 
If these certificates expire or need to be regenerated for any reason, performing a Linkerd upgrade

### Install Cert manager and Linkerd

```bash
# control plane core
kubectl create namespace linkerd
kubectl label namespace linkerd \
  linkerd.io/is-control-plane=true \
  config.linkerd.io/admission-webhooks=disabled \
  linkerd.io/control-plane-ns=linkerd
kubectl annotate namespace linkerd linkerd.io/inject=disabled

# viz (ignore if not using the viz extension)
kubectl create namespace linkerd-viz
kubectl label namespace linkerd-viz linkerd.io/extension=viz

# jaeger (ignore if not using the jaeger extension)
kubectl create namespace linkerd-jaeger
kubectl label namespace linkerd-jaeger linkerd.io/extension=jaeger
```

### Save the signing key pair as a Secret

create a **signing key pair** which will be used to sign each of the **webhook certificates**:
```bash
step certificate create webhook.linkerd.cluster.local ca.crt ca.key \
  --profile root-ca --no-password --insecure --san webhook.linkerd.cluster.local

kubectl create secret tls webhook-issuer-tls --cert=ca.crt --key=ca.key --namespace=linkerd

# ignore if not using the viz extension
kubectl create secret tls webhook-issuer-tls --cert=ca.crt --key=ca.key --namespace=linkerd-viz

# ignore if not using the jaeger extension
kubectl create secret tls webhook-issuer-tls --cert=ca.crt --key=ca.key --namespace=linkerd-jaeger
```

### Create Issuers referencing the secrets

we can create **cert-manager “Issuer”** resources that reference them:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: webhook-issuer
  namespace: linkerd
spec:
  ca:
    secretName: webhook-issuer-tls
---
# ignore if not using the viz extension
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: webhook-issuer
  namespace: linkerd-viz
spec:
  ca:
    secretName: webhook-issuer-tls
---
# ignore if not using the jaeger extension
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: webhook-issuer
  namespace: linkerd-jaeger
spec:
  ca:
    secretName: webhook-issuer-tls
```

### Issuing certificates and writing them to secrets

create **cert-manager "Certificate"** resources which **use the Issuers to generate the desired certificates**:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-policy-validator
  namespace: linkerd
spec:
  secretName: linkerd-policy-validator-k8s-tls
  duration: 24h
  renewBefore: 1h
  issuerRef:
    name: webhook-issuer
    kind: Issuer
  commonName: linkerd-policy-validator.linkerd.svc
  dnsNames:
  - linkerd-policy-validator.linkerd.svc
  isCA: false
  privateKey:
    algorithm: ECDSA
    encoding: PKCS8
  usages:
  - server auth
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-proxy-injector
  namespace: linkerd
spec:
  secretName: linkerd-proxy-injector-k8s-tls
  duration: 24h
  renewBefore: 1h
  issuerRef:
    name: webhook-issuer
    kind: Issuer
  commonName: linkerd-proxy-injector.linkerd.svc
  dnsNames:
  - linkerd-proxy-injector.linkerd.svc
  isCA: false
  privateKey:
    algorithm: ECDSA
  usages:
  - server auth
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-sp-validator
  namespace: linkerd
spec:
  secretName: linkerd-sp-validator-k8s-tls
  duration: 24h
  renewBefore: 1h
  issuerRef:
    name: webhook-issuer
    kind: Issuer
  commonName: linkerd-sp-validator.linkerd.svc
  dnsNames:
  - linkerd-sp-validator.linkerd.svc
  isCA: false
  privateKey:
    algorithm: ECDSA
  usages:
  - server auth
---
# ignore if not using the viz extension
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tap
  namespace: linkerd-viz
spec:
  secretName: tap-k8s-tls
  duration: 24h
  renewBefore: 1h
  issuerRef:
    name: webhook-issuer
    kind: Issuer
  commonName: tap.linkerd-viz.svc
  dnsNames:
  - tap.linkerd-viz.svc
  isCA: false
  privateKey:
    algorithm: ECDSA
  usages:
  - server auth
---
# ignore if not using the viz extension
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-tap-injector
  namespace: linkerd-viz
spec:
  secretName: tap-injector-k8s-tls
  duration: 24h
  renewBefore: 1h
  issuerRef:
    name: webhook-issuer
    kind: Issuer
  commonName: tap-injector.linkerd-viz.svc
  dnsNames:
  - tap-injector.linkerd-viz.svc
  isCA: false
  privateKey:
    algorithm: ECDSA
  usages:
  - server auth
---
# ignore if not using the jaeger extension
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: jaeger-injector
  namespace: linkerd-jaeger
spec:
  secretName: jaeger-injector-k8s-tls
  duration: 24h
  renewBefore: 1h
  issuerRef:
    name: webhook-issuer
    kind: Issuer
  commonName: jaeger-injector.linkerd-jaeger.svc
  dnsNames:
  - jaeger-injector.linkerd-jaeger.svc
  isCA: false
  privateKey:
    algorithm: ECDSA
  usages:
  - server auth
```

### Using these credentials with CLI installation

To **configure Linkerd to use the credentials** from cert-manager rather than generating its own:

```bash
# first, install the Linkerd CRDs
linkerd install --crds | kubectl apply -f -

# install the Linkerd control plane, using the credentials
# from cert-manager
linkerd install \
  --set policyValidator.externalSecret=true \
  --set-file policyValidator.caBundle=ca.crt \
  --set proxyInjector.externalSecret=true \
  --set-file proxyInjector.caBundle=ca.crt \
  --set profileValidator.externalSecret=true \
  --set-file profileValidator.caBundle=ca.crt \
  | kubectl apply -f -

# ignore if not using the viz extension
linkerd viz install \
  --set tap.externalSecret=true \
  --set-file tap.caBundle=ca.crt \
  --set tapInjector.externalSecret=true \
  --set-file tapInjector.caBundle=ca.crt \
  | kubectl apply -f -

# ignore if not using the jaeger extension
linkerd jaeger install
  --set webhook.externalSecret=true \
  --set-file webhook.caBundle=ca.crt \
  | kubectl apply -f -
```
---------------------------------------------------------------------------------------------------

## Automatically Rotating Control Plane TLS Credentials


create the namespace that cert-manager will use to **store its Linkerd-related resources**.
```
kubectl create namespace linkerd
```

### Save the signing key pair as a Secret

using the step tool, create a signing key pair and store it in a Kubernetes Secret
for longer-lived trust anchor certificate, pass the **--not-after** (--not-after=87600h)
```bash
step certificate create root.linkerd.cluster.local ca.crt ca.key \
  --profile root-ca --no-password --insecure &&
  kubectl create secret tls \
    linkerd-trust-anchor \
    --cert=ca.crt \
    --key=ca.key \
    --namespace=linkerd
```

### Create an Issuer referencing the secret

**create root certificate (Trust anchor certificate)**

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: linkerd-trust-anchor
  namespace: linkerd
spec:
  ca:
    secretName: linkerd-trust-anchor
```

### Create a Certificate resource referencing the Issuer

**create Issuer certificate and key**

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-identity-issuer
  namespace: linkerd
spec:
  secretName: linkerd-identity-issuer
  duration: 48h
  renewBefore: 25h
  issuerRef:
    name: linkerd-trust-anchor
    kind: Issuer
  commonName: identity.linkerd.cluster.local
  dnsNames:
  - identity.linkerd.cluster.local
  isCA: true
  privateKey:
    algorithm: ECDSA
  usages:
  - cert sign
  - crl sign
  - server auth
  - client auth
```
cert-manager can now use this Certificate resource to **obtain TLS credentials**,
which will be stored in a **secret named linkerd-identity-issuer**

```bash
kubectl get secret linkerd-identity-issuer -o yaml -n linkerd
```

### Using these credentials with CLI installation

For **CLI installation**, the **Linkerd control plane** should be installed with the **--identity-external-issuer flag**,
which instructs Linkerd to read certificates from the **linkerd-identity-issuer secret**. Whenever certificate and
key stored in the secret are updated,the identity service will automatically detect this change and reload the new credentials.


### Observing the update process

check for the **IssuerUpdated Kubernetes event** to be certain that Linkerd saw the new issuer certificate:
```bash
kubectl get events --field-selector reason=IssuerUpdated -n linkerd
```
---------------------------------------------------------------------------------------------------

## Linkerd SMI

Linkerd supports **SMI’s TrafficSplit** specification which can be used to perform **traffic splitting across services** natively

```bash
curl -sL https://linkerd.github.io/linkerd-smi/install | sh
linkerd smi install | kubectl apply -f -
linkerd smi check
```
---------------------------------------------------------------------------------------------------

## Telemetry and Monitoring

```
linkerd viz install | kubectl apply -f -
```
