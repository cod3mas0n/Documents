---
title: Securing Node Exporter Metrics
published: true
description: Scrape metrics from nodes outside of a Kubernetes cluster using Encrypted and Authenticated Node Exporter.
tags: 'nodeexporter,prometheus,monitoring,kubernetes'
cover_image: 'https://raw.githubusercontent.com/cod3mas0n/Documents/refs/heads/main/posts/assets/securing-node-exporter-metrics.webp'
canonical_url: null
id: 2319305
date: '2025-03-08T19:36:55Z'
---

_There’s a need to scrape metrics from nodes outside of a Kubernetes cluster using Node Exporter. But what if these nodes are exposed to the internet? In such cases, it’s crucial to implement both encryption and authentication to secure the communication._

## Steps to Secure and Scrape Node Exporter Metrics

Node Exporter Configuration for Secure Metrics Exposure.

- Encryption: Generating and Implementing SSL/TLS Certificates.
- Authentication: Setting Up Basic Authentication for Secure Access.

Prometheus Configuration to Scrape Metrics from External Nodes in Kubernetes

- with `Helm` `values.yaml`.
- with the `ScrapeConfig` Resource.

Setup Firewall or `iptables` to allow trusted IPs.

---

## Node Exporter Configuration

### Encryption

Create a directory at `/opt/node_exporter` to store the certificates and `config.yml` file. The `node_exporter` can be set up and run using Docker and Docker Compose, binary installation via Ansible playbook, or manually.

```bash
node_exporter
└── configs
```

Generating self-signed SSL/TLS certificate and private key with `openssl` .

```bash
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /opt/node_exporter/configs/node_exporter.key -out /opt/node_exporter/configs/node_exporter.crt -subj "/C=US/ST=California/L=Oakland/O=MyOrg/CN=localhost" -addext "subjectAltName = DNS:localhost"
```

### Authentication

Create a hashed password with `htpasswd` from `httpd-tools` in RedHat-based and `apache2-utils` package in Debian-based distributions.

```bash
htpasswd -nBC 12 "" | tr -d ':\n'

# -B Use bcrypt hashing for passwords. This is currently considered to be very secure.
# -n Display the results on standard output rather than updating a file.
# -C This flag is only allowed in combination with -B (bcrypt hashing). It sets the computing time used for thebcrypt algorithm (higher is more secure but slower, default: 5, valid: 4 to 17).
```

Or with a programming language, here is a Python script that uses `python3-bcrypt` to prompt for a password and hash it.

```python
#!/usr/bin/python3

import getpass
import bcrypt

password = getpass.getpass("password: ")
hashed_password = bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt())
print(hashed_password.decode())
```

Save that script as `gen-pass.py` and run it python3 `gen-pass.py` , which should prompt you for a password.

```text
password:
$2b$12$hNf2lSsxfm0.i4a.1kVpSOVyBCfIB51VRjgBUyv6kdnyTlgWj81Ay
```

### Setup and Configure the `node_exporter`

Use your beloved editor, mine is vim and add certificates and authentication into `/opt/node_exporter/configs/config.yml` .

```yaml
tls_server_config:
  cert_file: /etc/node_exporter/node_exporter.crt
  key_file: /etc/node_exporter/node_exporter.key
basic_auth_users:
  <USER>: <HASHED-PASSWD>
```

The `node_exporter` will be running with the `config.yml` , so add the `--web.config.file` argument to command in `/opt/node_exporter/docker-compose.yml` , Also the configs directory is going to be mounted as `/etc/node_exporter` in container.

```yaml
services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
      - ./configs:/etc/node_exporter:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
      - '--web.config.file=/etc/node_exporter/config.yml'
    ports:
      - 9100:9100
```

Change the ownership of the `/opt/node_exporter/configs` directory to `nobody` according to the [Dockerfile][4], for the permission denied Problem.

```bash
chown -R nobody:nobody /opt/node_exporter/configs
```

Run the `docker-compos.yml` , I'd like to use the exact `docker-compose` path

```bash
docker compose -f /opt/node_exporter/configs/docker-compose.yml up -d
```

To see node-exporter service logs, Again the exact path :)

```bash
docker compose -f /opt/node_exporter/configs/docker-compose.yml logs -f
```

Output should be `msg="TLS is enabled." http2=true address=[::]:9100`

```text
node-exporter | time=2025-03-02T22:18:16.472Z level=INFO source=node_exporter.go:141 msg=zfs
node-exporter | time=2025-03-02T22:18:16.474Z level=INFO source=tls_config.go:347 msg="Listening on" address=[::]:9100
node-exporter | time=2025-03-02T22:18:16.474Z level=INFO source=tls_config.go:383 msg="TLS is enabled." http2=true address=[::]:9100
```

---

## Prometheus Configuration

### Add `scrapeConfig` to the `prometheus-stack` in the K8S cluster

Happy editing helm values.yaml ! I’d like to pull the helm charts to keep and track versioning in a git repository.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm pull prometheus-community/kube-prometheus-stack

tar xvzf kube-prometheus-stack-<version>.tgz # Version on Mar 1 2025 = 69.5.2

mv kube-prometheus-stack kube-prometheus-stack-<version> # To keep and Track Versioning in a Git Repo

cd kube-prometheus-stack-<version> # Happy Editing Helm Values!
```

Encode the certificate `/opt/node_exporter/configs/node_exporter.crt` to create a Secret in the Kubernetes cluster.

```bash
base64 -w 0 /opt/node_exporter/configs/node_exporter.crt

# man base64, `-w`
# -w, --wrap=COLS
# wrap encoded lines after COLS character (default 76). Use 0 to disable line wrapping
```

### Secret of the Certificate

Create Secret, it has to be in the same namespace as the Prometheus namespace.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: node01-node-exporter-crt
  namespace: prometheus
type: Opaque
data:
  node01.node_exporter.crt: "<base64-encoded-cert>"
```

Apply the Secret

```bash
kubectl apply -f node01-node-exporter-secret.yaml -n prometheus
```

### Include the Secret

include secret with `prometheus.psometheusSpec.secret` in `values.yaml` ; As the comment says secrets will be mounted in `/etc/prometheus/secrets` .

in this case the `node01-node-exporter-crt` could be found in `/etc/prometheus/secrets/node01-node-exporter-crt/node01.node_exporter.crt` in the Prometheus pods.

```yaml
---
prometheus:
 prometheusSpec:
    ## Secrets is a list of Secrets in the same namespace as the Prometheus object, which shall be mounted into the Prometheus Pods.
    ## The Secrets are mounted into /etc/prometheus/secrets/. Secrets changes after initial creation of a Prometheus object are not
    ## reflected in the running Pods. To change the secrets mounted into the Prometheus Pods, the object must be deleted and recreated
    ## with the new list of secrets.
    ##
    secrets:
      - node01-node-exporter-crt
```

### Add `additionalScrapeConfigs`

include `external-nodes` job with `additionalScrapeConfigs` in `values.yaml` , the `scheme` has to be `https` , set the `tls_config` and `basic_auth`.

```yaml
---
prometheus:
 prometheusSpec:
    ## AdditionalScrapeConfigs allows specifying additional Prometheus scrape configurations. Scrape configurations
    ## are appended to the configurations generated by the Prometheus Operator. Job configurations must have the form
    ## as specified in the official Prometheus documentation:
    ## https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config. As scrape configs are
    ## appended, the user is responsible to make sure it is valid. Note that using this feature may expose the possibility
    ## to break upgrades of Prometheus. It is advised to review Prometheus release notes to ensure that no incompatible
    ## scrape configs are going to break Prometheus after the upgrade.
    ## AdditionalScrapeConfigs can be defined as a list or as a templated string.
    ##
  additionalScrapeConfigs:
      - job_name: external-nodes
        scheme: https
        tls_config:
          ca_file: /etc/prometheus/secrets/node01-node-exporter-crt/node01.node_exporter.crt
          ## the certificate is Self-Signed so we need to skip verification
          insecure_skip_verify: true
        ## User and Password in the authentication step we have created
        basic_auth:
          username: <USER> # the user you set in the node-exporter in the node
          password: <Plain-Text-Password> # the plain text password which you hashed in the Encryption step
        static_configs:
          - targets:
              - "<Remote-HOST-IP>:9100"
            labels:
              instance: "node01"
```

Apply changes with `helm upgrade`

```bash
helm upgrade --install prometheus-stack . -n prometheus --values values.yaml
```

</br>

## The `scrapeconfigs.monitoring.coreos.com` CRD

Instead of doing the below steps you can create a `ScrapeConfig` resource and avoid to have many changes in `values.yaml`.

- [Secret of the Certificate][1]
- [Include the Secret][2]
- [Add `additionalScrapeConfigs`][3]

The `Secret` can contain many keys to use in the `ScrapeConfig`.

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: external-nodes-secrets
  namespace: prometheus
type: Opaque
data:
  node01.node_exporter.crt: "<base64-encoded-cert>"
  # # the user you set in the node-exporter in the node
  node01-basicauth-username: "<base64-encoded-username>"
  # # the password you set in the node-exporter in the node
  node01-basicauth-password: "<base64-encoded-password>"

---
apiVersion: monitoring.coreos.com/v1alpha1
kind: ScrapeConfig
metadata:
  namespace: prometheus
  name: external-nodes-scfg
  labels:
    release: prometheus-stack
spec:
  jobName: external-nodes
  scheme: HTTPS
  tlsConfig:
    ca:
      secret:
        name: external-nodes-secrets
        key: node01.node_exporter.crt
    insecureSkipVerify: true
  basicAuth:
    username:
      name: external-nodes-secrets
      key: node01-basicauth-username
    password:
      name: external-nodes-secrets
      key: node01-basicauth-password
  staticConfigs:
    - targets:
        - "<Remote-HOST-IP>:9100"
      labels:
        instance: "node01"

```

---

## Firewall or `iptables`

Encryption/Authentication alongside the `iptables` rule is the best practice. It could be done with `iptables` or `firewalls` to allow trusted IPs to scrape the node on the `9100` port.

I assume the egress IP of the Kubernetes cluster is a single IP, otherwise, the IP range should be allowed via iptables .

```bash
# Allow only IP <K8S-Egress-IP> to access port 9100 (TCP)
iptables -A INPUT -p tcp -d <HOST-IP> --dport 9100 -s <K8S-Egress-IP> -m state --state NEW,ESTABLISHED -m comment --comment "Allow IP <K8S-Egress-IP> to access the 9100 port " -j ACCEPT

# Block all other incoming traffic to port 9100
# Instead of DROP, it's better to use REJECT, it sends a response back to the source IP, informing it that the connection is rejected. DROP would silently discard the packets without notifying the source.
iptables -A INPUT -p tcp --dport 9100 -m state --state NEW,ESTABLISHED -m comment --comment "Reject all other traffic to 9100 port " -j REJECT
```

[1]: #secret-of-the-certificate
[2]: #include-the-secret
[3]: #add-additionalscrapeconfigs
[4]: https://github.com/prometheus/node_exporter/blob/master/Dockerfile#L11
