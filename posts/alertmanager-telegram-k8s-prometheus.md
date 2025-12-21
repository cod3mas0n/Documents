---
title: Alertmanager with Telegram for Prometheus-Stack in K8S
published: true
description: Add Telegram Receiver for alertmanager for prometheus-stack in k8s cluster
tags: 'alertmanager,prometheus,monitoring,kubernetes'
cover_image: 'https://raw.githubusercontent.com/cod3mas0n/Documents/refs/heads/main/posts/assets/alertmanager-to-telegram.webp'
canonical_url: null
id: 2320569
date: '2025-03-09T12:06:12Z'
---

### [How to create a Telegram bot][1]

- To create a Telegram bot, you'll need to have the Telegram app installed on your computer. If you don't have it already, you can download it from the Telegram website.
- Connect to BotFather and hit the `/newbot` and follow the instruction

    BotFather is a bot created by Telegram that allows you to create and manage your own bots. To connect to BotFather, search for "@BotFather" in the Telegram app and click on the result to start a conversation.

In case you’d like a more detailed walkthrough, we suggest you take a look at this amazing [tutorial][2] prepared by Telegram.

Save the bot access token, it will be used in seting up telegram receiver in alertmanager.Also, join the bot to a channel/group to get the messages.
</br>
To get a info about the bot
> The `bot` prefix has to be in the URL before the <BOT_ACCESS_TOKEN></br>
> bot<BOT_ACCESS_TOKEN>.

- `BOT_ACCESS_TOKEN`: it is provided by the `"@BotFather"`.

```bash
curl https://api.telegram.org/bot<BOT_ACCESS_TOKEN>/getMe | jq .
```

```json
{
    "ok": true,
  "result": {
      "id": <some_id>,
    "is_bot": true,
    "first_name": <"bot_name">,
    "username": <"bot_username">,
    "can_join_groups": true,
    "can_read_all_group_messages": false,
    "supports_inline_queries": false,
    "can_connect_to_business": false,
    "has_main_web_app": false
  }
}
```

</br>

- `CHAT_ID`: After joining the bot to a channel/group , get the `chat_id` which you want to send message on behalf of the bot to that channel/group .

```bash
curl https://api.telegram.org/bot<BOT_ACCESS_TOKEN>/getUpdates | jq .
```

</br>

To Send a test message

```bash
curl -X POST "https://api.telegram.org/bot<BOT_ACCESS_TOKEN>/sendMessage" -d "chat_id=<CHAT_ID>" -d "text=Hello, World!"
```

</br>

Let's say the `prometheus-stack` is installed via `helm` or its `operator` in K8S cluster.</br>
The release name will be used in many objects like `AlertmanagerConfig`, `ServiceMonitor`, `PodMonitor`, etc ... which the prometheus operator be able to select objects and add configs to the prometheus or alertmanager .

get release list in the `prometheus` namespace

```bash
helm list -n prometheus
```

```text
NAME            	NAMESPACE 	REVISION	UPDATED                                  	STATUS  	CHART                       	APP VERSION
prometheus-stack	prometheus	1      	2025-03-08 23:34:47.082601345 +0330 +0330	deployed	kube-prometheus-stack-68.4.4	v0.79.2
```

</br>

## `AlertmanagerConfig` to define the `Telegram` receiver

```text
GROUP:      monitoring.coreos.com
KIND:       AlertmanagerConfig
VERSION:    v1alpha1

DESCRIPTION:
    AlertmanagerConfig configures the Prometheus Alertmanager,
    specifying how alerts should be grouped, inhibited and notified to external
    systems.

```

### `telegramConfigs`

```bash
kubectl explain AlertmanagerConfig.spec.receivers.telegramConfigs
```

```text
GROUP:      monitoring.coreos.com
KIND:       AlertmanagerConfig
VERSION:    v1alpha1

FIELD: telegramConfigs <[]Object>


DESCRIPTION:
    List of Telegram configurations.
    TelegramConfig configures notifications via Telegram.
    See
    https://prometheus.io/docs/alerting/latest/configuration/#telegram_config

  botToken	<Object>
    Telegram bot token. It is mutually exclusive with `botTokenFile`.
    The secret needs to be in the same namespace as the AlertmanagerConfig
    object and accessible by the Prometheus Operator.

    Either `botToken` or `botTokenFile` is required.

  chatID	<integer> -required-
    The Telegram chat ID.

  parseMode	<string>
  enum: MarkdownV2, Markdown, HTML
    Parse mode for telegram message

  sendResolved	<boolean>
    Whether to notify about resolved alerts.

```

The `botToken` has to be a `Secret` in the same `namespace` as the `AlertmanagerConfig`. lets say we name the secret file name `telegram-bot-token-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: telegram-bot-token-secret # secret name will be referred in botToken in AlertmanagerConfig
  namespace: prometheus # same `namespace` as the `AlertmanagerConfig`.
type: Opaque
data:
  # bot-token will be used as key for botToken in AlermanagerConfig.
  bot-token: <Base64-encoded-BOT-TOKEN>
  # example
  # echo -n "<BOT_ACCESS_TOKEN>" | base64 (Token without bot prefix, exactly the token get from bot_father)

```

- Create the `Secret`

```bash
kubectl apply -f telegram-bot-token-secret.yaml -n prometheus
```

</br>

### `AlertmanagerConfig`

Create a new file named `alertmanagerconfig-telegram.yaml` with contents:

```yaml
---
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: alert-config-telegram
  namespace: prometheus
  labels:
    release: prometheus-stack # the release label should be the exact label of the prometheus helm release label
    alertmanagerConfig: alertmanager-telegram # custom label and also set this label to alertmanager in the prometheus-stack values.yaml
spec:
  route:
    groupBy: ["alertname","job","namespace"] # it could be other labels to be grouped by
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 1h
    receiver: "telegram"
    routes:
    - receiver: "telegram"
      matchers:
        - name: severity
          matchType: "=~"
          value: "warning|critical"
  receivers:
    - name: telegram
      telegramConfigs:
        - botToken:
            name: telegram-bot-token-secret # refer to the secret name has been created before
            key: bot-token # bot-token data in the secret
          chatID: <CHAT_ID>
          parseMode: 'MarkdownV2' # The template I provide is in Markdown Mode
          disableNotifications: false
          sendResolved: true

```

- Create the `AlertmanagerConfig`

```bash
kubectl apply -f alertmanagerconfig-telegram.yaml -n prometheus
```

</br>

### The `alertmanager` changes in the`prometheus-stack` `values.yaml`

- Templates in `alertmanager.templateFiles` will be mounted in `/etc/alertmanager/config/*.tmpl`.

  > The `telegram.tmpl` will be mounted as `/etc/alertmanager/config/telegram.tmpl`, with defining the `{{ define "telegram.default.message" }}` there is no need to define `message` in the `AlertmanagerConfig.spec.receivers.telegramConfigs`. with that the default template for the telegram will be overwritten.

  ```yaml
  alertmanager:
    templateFiles:
      telegram.tmpl: |-
        {{ define "telegram.default.message" }}
        {{- if eq .Status "firing" -}}
            {{- if eq .CommonLabels.severity "critical" -}}
                🔴 Alert: {{ .CommonLabels.alertname }}
            {{- else if eq .CommonLabels.severity "warning" -}}
                🟠 Alert: {{ .CommonLabels.alertname }}
            {{- else -}}
                ⚪️ Alert: {{ .CommonLabels.alertname }}
            {{- end }}
        Status: 🔥 FIRING
        Severity: {{ if eq .CommonLabels.severity "critical" }}🔴 {{ .CommonLabels.severity | title }}{{ else if eq .CommonLabels.severity "warning" }}🟠 {{ .CommonLabels.severity | title }}{{ else }}⚪️ {{ .CommonLabels.severity | title }}{{ end }}
        {{- else if eq .Status "resolved" -}}
            ⚪️ Alert: {{ .CommonLabels.alertname }}
        Status: ✅ RESOLVED
        Severity: {{ if eq .CommonLabels.severity "critical" }}🟢 {{ .CommonLabels.severity | title }}{{ else if eq .CommonLabels.severity "warning" }}🟢 {{ .CommonLabels.severity | title }}{{ else }}⚪️ {{ .CommonLabels.severity | title }}{{ end }}
        {{- end }}

        {{- range .Alerts -}}

        {{- if .Labels.job }}
        Job: `{{ .Labels.job }}`
        {{- end }}

        {{- if .Labels.namespace }}
        Namespace: `{{ .Labels.namespace }}`
        {{- end }}

        {{- if .Labels.instance }}
        Instance: `{{ .Labels.instance }}`
        {{- end }}

        {{- if .Annotations.runbook_url }}
        [RunbookURL]({{ .Annotations.runbook_url }})

        {{- end }}
        {{- end }}
        {{ end }}
  ```

  - Example Message

    ```text
    🟠 Alert: TargetDown
    Status: 🔥 FIRING
    Severity: 🟠 Warning
    Job: kube-proxy
    Namespace: kube-system
    RunbookURL
    ```

</br>

- `alertmanagerConfigSelector`

  ```yaml
  alertmanager:
    alertmanagerSpec:
      ## AlertmanagerConfigs to be selected to merge and configure Alertmanager with.
      ##
      alertmanagerConfigSelector:
        matchLabels:
          alertmanagerConfig: alertmanager-telegram # it is the same label in the AlertmanagerConfig.
  ```

</br>

- `alertmanagerConfigMatcherStrategy` with the `None` Type the `AlertmanagerConfig` will match the all alerts.

  ```bash
  kubectl explain Alertmanager.spec.alertmanagerConfigMatcherStrategy
  ```

  ```text
  GROUP:      monitoring.coreos.com
  KIND:       Alertmanager
  VERSION:    v1

  FIELD: alertmanagerConfigMatcherStrategy <Object>


  DESCRIPTION:
      AlertmanagerConfigMatcherStrategy defines how AlertmanagerConfig objects
      process incoming alerts.

  FIELDS:
  type	<string>
  enum: OnNamespace, None
    AlertmanagerConfigMatcherStrategyType defines the strategy used by
    AlertmanagerConfig objects to match alerts in the routes and inhibition
    rules.

    The default value is `OnNamespace`.

  ```

  ```yaml
  alertmanager:
  alertmanagerSpec:
      ## Defines the strategy used by AlertmanagerConfig objects to match alerts. eg:
      ##
      alertmanagerConfigMatcherStrategy:
      type: None
      ## Example with use OnNamespace strategy
      # alertmanagerConfigMatcherStrategy:
      #   type: OnNamespace
  ```

</br>

After all changes are made in the `values.yaml`, apply changes , in the case you installed the `prometheus-stack` with `helm`

```bash
helm upgrade prometheus-stack . -n prometheus --values values.yaml
```

## Example alerts via `prometheus-rule`

Create a file named `targets-prometheus-rules.yaml` with contents:

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  namespace: prometheus
  name: targets
  labels:
    release: prometheus-stack # this label has to be the release name prometheus-stack
spec:
  groups:
    - name: targets
      rules:
        - alert: PrometheusTargetMissing
          expr: up == 0
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Prometheus target missing \\(instance {{ $labels.instance }}\\)"
            description: "A Prometheus target has disappeared. An exporter might be crashed. \n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

        # node_exporter related metrics for alert
        - alert: HostOutOfMemory
          expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < .10)
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Host out of memory (instance {{ $labels.instance }})
            description: "Node memory is filling up (< 10% left)\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```

```bash
kubectl apply -f targets-prometheus-rules.yaml -n prometheus
```

## Example Message in Telegram

```text
⚪️ Alert: PrometheusTargetMissing
Status: ✅ RESOLVED
Severity: 🟢 Warning
Job: kube-proxy
Namespace: kube-system
Instance: 192.168.173.192:10249
Job: kube-proxy
Namespace: kube-system
Instance: 192.168.173.27:10249
Job: kube-proxy
Namespace: kube-system
Instance: 192.168.173.31:10249
```

```text
🔴 Alert: KubeControllerManagerDown
Status: 🔥 FIRING
Severity: 🔴 Critical
RunbookURL
```

```text
🔴 Alert: KubeSchedulerDown
Status: 🔥 FIRING
Severity: 🔴 Critical
RunbookURL
```

- Read about the [RunbookURL][3]

[1]: https://www.directual.com/lesson-library/how-to-create-a-telegram-bot
[2]: https://core.telegram.org/bots/tutorial
[3]: https://runbooks.prometheus-operator.dev/
