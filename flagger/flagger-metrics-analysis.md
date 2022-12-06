# Flagger Metrics Analysis

## Builtin metrics

As part of the analysis process, Flagger can validate **service level objectives (SLOs)** like
**availability, error rate percentage, average response time** and any other objective based on
app specific metrics, If a **drop in performance is noticed during the SLOs analysis**,
the release will be **automatically rolled back** with minimum impact to end-users.

Flagger comes with two **builtin metric checks: HTTP request success rate and duration**.

```yaml
  analysis:
    metrics:
    - name: request-success-rate
      interval: 1m
      # minimum req success rate (non 5xx responses)
      # percentage (0-100)
      thresholdRange:
        min: 99
    - name: request-duration
      interval: 1m
      # maximum req duration P99
      # milliseconds
      thresholdRange:
        max: 500
```

specify a **range of accepted values** with **thresholdRange** and **the window size or the time series** with **interval**.

## Custom metrics

Using a **MetricTemplate** custom resource, you configure Flagger to connect to a **metric provider** and **run a query that returns a float64 value**.
The query result is used to validate the **canary based on the specified threshold range**.

```yaml
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: not-found-percentage
  namespace: flagger
spec:
  provider:
    type: prometheus
    address: http://prometheus.jojo-system:9090
    secretRef:
        name: prom-basic-auth
  query: |
    100 - sum(
        rate(
            istio_requests_total{
              reporter="destination",
              destination_workload_namespace="{{ namespace }}",
              destination_workload="{{ target }}",
              response_code!="404"
            }[{{ interval }}]
        )
    )
    /
    sum(
        rate(
            istio_requests_total{
              reporter="destination",
              destination_workload_namespace="{{ namespace }}",
              destination_workload="{{ target }}"
            }[{{ interval }}]
        )
    ) * 100
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: prom-basic-auth
  namespace: flagger
data:
  username: your-user
  password: your-password
```

The above configuration validates the canary by checking if the **HTTP 404 req/sec percentage is below 5 percent of the total traffic**,
**If the 404s rate reaches the 5% threshold, then the canary fails**.
