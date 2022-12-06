# Flagger Webhooks

## Webhooks

Flagger will call each **webhook URL** and determine from the **response status code (HTTP 2xx)** if the canary is **failing or not**.

There are several types of hooks:

* **confirm-rollout**

> hooks are **executed before scaling up the canary deployment** , The **rollout is paused** until the hook returns a **successful HTTP status code**.

* **pre-rollout**

hooks are **executed before routing traffic to canary**.

* **rollout**

hooks are **executed during the analysis on each iteration before the metric checks**.

* **confirm-traffic-increase**

hooks are **executed right before the weight on the canary is increased**.

* **confirm-promotion**

hooks are **executed before the promotion step**.

* **post-rollout**

hooks are **executed after the canary has been promoted or rolled back.**

* **rollback**
* **event**

hooks are **executed every time Flagger emits a Kubernetes event**.
every action that Flagger takes during a canary deployment will be sent as JSON via an HTTP POST request.


```yaml
  analysis:
    webhooks:
      - name: "start gate"
        type: confirm-rollout
        url: http://flagger-loadtester.test/gate/approve
      - name: "helm test"
        type: pre-rollout
        url: http://flagger-helmtester.flagger/
        timeout: 3m
        metadata:
          type: "helmv3"
          cmd: "test podinfo -n test"
      - name: "load test"
        type: rollout
        url: http://flagger-loadtester.test/
        timeout: 15s
        metadata:
          cmd: "hey -z 1m -q 5 -c 2 http://podinfo-canary.test:9898/"
      - name: "traffic increase gate"
        type: confirm-traffic-increase
        url: http://flagger-loadtester.test/gate/approve
      - name: "promotion gate"
        type: confirm-promotion
        url: http://flagger-loadtester.test/gate/approve
      - name: "notify"
        type: post-rollout
        url: http://telegram.bot:8080/
        timeout: 5s
        metadata:
          some: "message"
      - name: "rollback gate"
        type: rollback
        url: http://flagger-loadtester.test/rollback/check
      - name: "send to Slack"
        type: event
        url: http://event-recevier.notifications/slack
        metadata:
          environment: "test"
          cluster: "flagger-test"
```

