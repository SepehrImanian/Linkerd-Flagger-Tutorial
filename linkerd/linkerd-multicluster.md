# Linkerd Multicluster

## Installing Multi-cluster Components


### Step 1: Install the multicluster control plane

Multicluster support in Linkerd requires extra installation and configuration
on **top of the default control plane installation**.

On each cluster, run:

```
linkerd multicluster install | \
    kubectl apply -f -
```
```
linkerd multicluster check
```

### Step 2: Link the clusters

To link cluster west to cluster east, you would run:

```
linkerd --context=east multicluster link --cluster-name east |
  kubectl --context=west apply -f -
```
