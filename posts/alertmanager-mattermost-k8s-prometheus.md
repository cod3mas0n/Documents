---
title: Mattermost Receiver For Alertmanager in Prometheus-Stack in K8S
published: true
description: Add Mattermost Receiver for alertmanager in prometheus-stack in k8s
tags: 'alertmanager,prometheus,monitoring,kubernetes'
---

### Why

Since Prometheus-Stack Alertmanager doesn’t include Mattermost as a built-in receiver, we will integrate it using a webhook via creating a Python relay application that accepts Alertmanager webhook notifications, formats them into a Mattermost-compatible payload, and sends them to a Mattermost incoming webhook URL.

Then we will deploy this relay inside Kubernetes using a Deployment and expose it with a Service so Alertmanager can reliably reach it.

Finally, we will configure Alertmanager using an AlertmanagerConfig resource to define a new receiver and route, so alerts matching our rules (severity/namespace/etc.) get delivered to Mattermost through the relay.

### How

- Prometheus Alert Rules trigger alerts.
- Alertmanager processes and routes the alerts.
- Alertmanager sends webhook notifications to the Python Relay Service.
- The relay formats the alert payload.
- The relay forwards the formatted message to Mattermost Incoming Webhook.
- Alerts appear in the designated Mattermost channel.

### Steps

- Create an [Incoming Webhook in Mattermost][mattermost-incoming-webhook]. This webhook URL will be used by the Python relay to deliver alert messages to the desired channel.
   > Ensure you securely store the generated webhook URL, as it will be injected into the relay application via environment variables or Kubernetes secrets.

- [Python Relay][python-relay-app] Application

    > A lightweight webhook server that receives Alertmanager alerts and forwards them to Mattermost.

- Build the Docker Image [Dockerfile][dockerfile]

    Once the relay application is complete, build the image and push it to a container registry accessible by your Kubernetes cluster.

- Deploy in Kubernetes: [K8S Manifests][k8s-manifests]
- Configure Alertmanager with [AlertmanagerConfig][alertmanagerconfig-manifest]
   > Ensure having a [label][alertmanagerconfig-label] in AlertmanagerConfig to select via prometheus operator

- Add the Alertmanagerconfig to prometheus `values.yaml`

    ```yaml
    ⋮
    alertmanager:
    alertmanagerSpec:
        alertmanagerConfigSelector:
        matchLabels:
            alertmanagerConfig: alertmanager-mattermost
    ⋮
    ```

[mattermost-incoming-webhook]:https://docs.mattermost.com/integrations-guide/incoming-webhooks.html
[python-relay-app]: https://github.com/cod3mas0n/k8s-alertmanager-mattermost/blob/main/src/app.py
[dockerfile]: https://github.com/cod3mas0n/k8s-alertmanager-mattermost/blob/main/src/Dockerfile
[k8s-manifests]: https://github.com/cod3mas0n/k8s-alertmanager-mattermost/tree/main/kubernetes
[alertmanagerconfig-manifest]: https://github.com/cod3mas0n/k8s-alertmanager-mattermost/blob/main/kubernetes/alertmanagerConfig.yaml
[alertmanagerconfig-label]: https://github.com/cod3mas0n/k8s-alertmanager-mattermost/blob/main/kubernetes/alertmanagerConfig.yaml#L6-L7
