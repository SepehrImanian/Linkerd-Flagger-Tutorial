# Flagger Alerting

> Flagger can be configured to send alerts to various chat platforms. You can define a global alert provider at install time or configure alerts on a per canary basis.

## Flagger configuration

Flagger can be configured to send Slack notifications:

```bash
helm upgrade -i flagger flagger/flagger \
--set slack.url=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK \
--set slack.proxy=my-http-proxy.com \ # optional http/s proxy
--set slack.channel=general \
--set slack.user=flagger \
--set clusterName=my-cluster
```

> To make the alerting move flexible, the canary analysis can be extended with a **list of alerts that reference an alert provider**. For each alert, users can configure the severity level. The alerts section **overrides** the global setting.

```yaml
apiVersion: flagger.app/v1beta1
kind: AlertProvider
metadata:
  name: on-call
  namespace: flagger
spec:
  type: slack
  channel: on-call-alerts
  username: flagger
  # webhook address (ignored if secretRef is specified)
  # or https://slack.com/api/chat.postMessage if you use token in the secret
  address: https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
  # optional http/s proxy
  proxy: http://my-http-proxy.com
  # secret containing the webhook address (optional)
  secretRef:
    name: on-call-url
---
apiVersion: v1
kind: Secret
metadata:
  name: on-call-url
  namespace: flagger
data:
  address: <encoded-url>
  token: <encoded-token>
```

```yaml
  analysis:
    alerts:
      - name: "on-call Slack"
        severity: error
        providerRef:
          name: on-call
          namespace: flagger
      - name: "qa Discord"
        severity: warn
        providerRef:
          name: qa-discord
      - name: "dev MS Teams"
        severity: info
        providerRef:
          name: dev-msteams
```

## Prometheus Alert Manager

You can use Alertmanager to trigger alerts when a canary deployment failed:

```yaml
- alert: canary_rollback
  expr: flagger_canary_status > 1
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "Canary failed"
    description: "Workload {{ $labels.name }} namespace {{ $labels.namespace }}"
```
