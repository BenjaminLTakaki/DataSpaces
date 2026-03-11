# FIWARE Data Space Connector: Deployment and Operations Guide

## 1. System Overview and Requirements

This deployment establishes a secure data exchange environment consisting of a Data Provider, a Data Consumer, and a Trust Anchor. The infrastructure is orchestrated using K3s (Kubernetes) and utilizes APISIX gateways for traffic management.

### Minimum Specifications

| Resource | Requirement |
| :--- | :--- |
| Operating System | Ubuntu 24.04 |
| Memory | Minimum 24 GB RAM dedicated to the cluster |
| Processor | 4 vCores |
| Storage | 100 GB disk space |

### Deployment Flow

![FIWARE Deployment Flow](https://mermaid.ink/svg/Zmxvd2NoYXJ0IFRECiAgICBTdGFydChbU3RhcnRdKSAtLT4gUDFbUGhhc2UgMTogRW52aXJvbm1lbnQgUHJvdmlzaW9uaW5nXQoKICAgIHN1YmdyYXBoIFBoYXNlMVtQaGFzZSAxOiBFbnZpcm9ubWVudCBQcm92aXNpb25pbmddCiAgICAgICAgZGlyZWN0aW9uIFRCCiAgICAgICAgUzFfMVtTdGVwIDE6IENvcmUgVG9vbCBJbnN0YWxsYXRpb248YnI-YXB0IHVwZGF0ZS91cGdyYWRlPGJyPkluc3RhbGw6IHBpbmcsIGdpdCwganEsIEpESywgbmFubywgd2dldCwgbWF2ZW5dCiAgICAgICAgUzFfMltTdGVwIDI6IEszcyBJbml0aWFsaXphdGlvbjxicj5JbnN0YWxsIHlxPGJyPkluc3RhbGwgSzNzIGVuZ2luZTxicj5Db25maWd1cmUga3ViZWN0bCBwZXJtaXNzaW9uc10KICAgICAgICBTMV8zW1N0ZXAgMzogV29ya3NwYWNlIGFuZCBCdWlsZDxicj5DcmVhdGUgL2Zpd2FyZSBkaXJlY3Rvcnk8YnI-Q2xvbmUgZGF0YS1zcGFjZS1jb25uZWN0b3IgcmVwbzxicj5TZXQgS1VCRUNPTkZJRzxicj5tdm4gY2xlYW4gaW5zdGFsbF0KICAgICAgICBTMV8xIC0tPiBTMV8yIC0tPiBTMV8zCiAgICBlbmQKCiAgICBQMSAtLT4gUzFfMQogICAgUzFfMyAtLT4gUDJbUGhhc2UgMjogU2VjdXJpdHkgQ29uZmlndXJhdGlvbl0KCiAgICBzdWJncmFwaCBQaGFzZTJbUGhhc2UgMjogU2VjdXJpdHkgQ29uZmlndXJhdGlvbl0KICAgICAgICBkaXJlY3Rpb24gVEIKICAgICAgICBTMl8xW1N0ZXAgMTogQ2VydGlmaWNhdGUgR2VuZXJhdGlvbjxicj5SdW4gZ2VuZXJhdGUtY2VydHMuc2hdCiAgICAgICAgUzJfMltTdGVwIDI6IFNlY3JldCBJbmplY3Rpb248YnI-Q3JlYXRlIG5hbWVzcGFjZXM8YnI-Q3JlYXRlIFRMUyBzZWNyZXQgZm9yIGluZnJhPGJyPkNyZWF0ZSBzaWduaW5nIGtleXMgZm9yIHByb3ZpZGVyIGFuZCBjb25zdW1lcl0KICAgICAgICBTMl8xIC0tPiBTMl8yCiAgICBlbmQKCiAgICBQMiAtLT4gUzJfMQogICAgUzJfMiAtLT4gUDNbUGhhc2UgMzogUmVzb3VyY2UgRGVwbG95bWVudF0KCiAgICBzdWJncmFwaCBQaGFzZTNbUGhhc2UgMzogUmVzb3VyY2UgRGVwbG95bWVudF0KICAgICAgICBkaXJlY3Rpb24gVEIKICAgICAgICBTM18xW0RlcGxveSBLOHMgTWFuaWZlc3RzPGJyPmluZnJhIC8gdHJ1c3QtYW5jaG9yPGJyPnByb3ZpZGVyIC8gY29uc3VtZXJdCiAgICAgICAgUzNfMltJbmZyYXN0cnVjdHVyZSBPcHRpbWl6YXRpb248YnI-SW5jcmVhc2UgaW5vdGlmeSBsaW1pdHM8YnI-UGVyc2lzdCBzeXNjdGwgc2V0dGluZ3M8YnI-UmVzdGFydCBzdG9yYWdlIHByb3Zpc2lvbmVyXQogICAgICAgIFMzXzEgLS0-IFMzXzIKICAgIGVuZAoKICAgIFAzIC0tPiBTM18xCiAgICBTM18yIC0tPiBQNFtQaGFzZSA0OiBDbHVzdGVyIE1hbmFnZW1lbnRdCgogICAgc3ViZ3JhcGggUGhhc2U0W1BoYXNlIDQ6IENsdXN0ZXIgTWFuYWdlbWVudF0KICAgICAgICBkaXJlY3Rpb24gVEIKICAgICAgICBTNF8xW1N0YXR1cyBNb25pdG9yaW5nPGJyPmt1YmVjdGwgZ2V0IHBvZHMgLUFdCiAgICAgICAgUzRfMntBbGwgUG9kcyBSZWFkeT99CiAgICAgICAgUzRfMSAtLT4gUzRfMgogICAgICAgIFM0XzIgLS0gTm8gLS0-IFM0XzNbVHJvdWJsZXNob290IC8gV2FpdF0KICAgICAgICBTNF8zIC0tPiBTNF8xCiAgICAgICAgUzRfMiAtLSBZZXMgLS0-IFM0XzRbU2VydmljZSBMaWZlY3ljbGU8YnI-SGFsdDogc3lzdGVtY3RsIHN0b3AgazNzPGJyPlJlc3VtZTogc3lzdGVtY3RsIHN0YXJ0IGszczxicj5EZWNvbW1pc3Npb246IGszcy11bmluc3RhbGwuc2hdCiAgICBlbmQKCiAgICBQNCAtLT4gUzRfMQogICAgUzRfMiAtLSBZZXMgLS0-IFA1W1BoYXNlIDU6IEZ1bmN0aW9uYWwgVmVyaWZpY2F0aW9uXQoKICAgIHN1YmdyYXBoIFBoYXNlNVtQaGFzZSA1OiBGdW5jdGlvbmFsIFZlcmlmaWNhdGlvbl0KICAgICAgICBkaXJlY3Rpb24gVEIKICAgICAgICBTNV8xW1JlcXVlc3QgT0lEQyBUb2tlbjxicj5QT1NUIHRvIEtleWNsb2FrIENvbnN1bWVyPGJyPnVzZXI6IGVtcGxveWVlIC8gcGFzczogdGVzdF0KICAgICAgICBTNV8ye1Rva2VuIFJldHVybmVkP30KICAgICAgICBTNV8xIC0tPiBTNV8yCiAgICAgICAgUzVfMiAtLSBZZXMgLS0-IFM1XzMoW0Vudmlyb25tZW50IFJlYWR5XSkKICAgICAgICBTNV8yIC0tIE5vIC0tPiBTNV80W0RlYnVnIEF1dGggRmxvd10KICAgIGVuZAoKICAgIFA1IC0tPiBTNV8xCgogICAgc3R5bGUgU3RhcnQgZmlsbDojNENBRjUwLGNvbG9yOiNmZmYKICAgIHN0eWxlIFM1XzMgZmlsbDojNENBRjUwLGNvbG9yOiNmZmYKICAgIHN0eWxlIFAxIGZpbGw6IzE5NzZEMixjb2xvcjojZmZmCiAgICBzdHlsZSBQMiBmaWxsOiMxOTc2RDIsY29sb3I6I2ZmZgogICAgc3R5bGUgUDMgZmlsbDojMTk3NkQyLGNvbG9yOiNmZmYKICAgIHN0eWxlIFA0IGZpbGw6IzE5NzZEMixjb2xvcjojZmZmCiAgICBzdHlsZSBQNSBmaWxsOiMxOTc2RDIsY29sb3I6I2ZmZg==)

---

## 2. Phase 1: Environment Provisioning

Install the core toolset and the Kubernetes engine directly on the host system.

### Step 1: Core Tool Installation

Ensure all system repositories are up to date and install the required development and networking packages.

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get autoremove -y
sudo apt-get install iputils-ping git jq default-jdk nano wget maven -y
```

### Step 2: Kubernetes (K3s) Initialization

Install yq for YAML manipulation and initialize the K3s engine.

```bash
# Install yq
sudo wget https://github.com/mikefarah/yq/releases/download/v4.45.1/yq_linux_amd64 -O /usr/bin/yq && sudo chmod +x /usr/bin/yq

# Install K3s engine
curl -sfL https://get.k3s.io | sh -

# Configure local user permissions for kubectl
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

### Step 3: Workspace and Build

Prepare the directory structure and compile the Data Space Connector source code.

```bash
sudo mkdir /fiware && sudo chown -R $USER:$USER /fiware
cd /fiware
git clone https://github.com/FIWARE/data-space-connector.git
cd data-space-connector

# Set environment variable and compile
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
mvn clean install -Plocal -DskipTests -Ddocker.skip
```

---

## 3. Phase 2: Security Configuration

Cryptographic identity is central to the Data Space. This phase generates the required certificates and stores them as Kubernetes Secrets.

### Step 1: Certificate Generation

```bash
cd /fiware/data-space-connector/helpers/certs/
chmod +x generate-certs.sh
./generate-certs.sh
```

### Step 2: Secret Injection

Create the namespaces and provision the signing keys to the relevant segments of the cluster.

```bash
# Provision namespaces
kubectl apply -f /fiware/data-space-connector/target/k3s/namespaces/

# Create TLS secret for infrastructure
kubectl create secret tls local-wildcard \
  --cert=./out/client-consumer/certs/client-chain-bundle.cert.pem \
  --key=./out/client-consumer/private/client.key.pem -n infra

# Create role-specific signing keys
kubectl create secret generic signing-key --from-file=./out/client-provider/private/client.key.pem -n provider
kubectl create secret generic signing-key --from-file=./out/client-consumer/private/client.key.pem -n consumer
```

---

## 4. Phase 3: Resource Deployment

Deploy the primary Data Space components using the pre-compiled Kubernetes manifests.

```bash
cd /fiware/data-space-connector
kubectl apply -f ./target/k3s/infra/ -R
kubectl apply -f ./target/k3s/trust-anchor/trust-anchor/ -R
kubectl apply -f ./target/k3s/dsc-provider/data-space-connector/ -R
kubectl apply -f ./target/k3s/dsc-consumer/data-space-connector/ -R
```

### Infrastructure Optimization

Adjust system file-watch limits to support high container density and restart the storage provisioner.

```bash
sudo sysctl fs.inotify.max_user_instances=512
sudo sysctl fs.inotify.max_user_watches=65536

# Make limits persistent
echo "fs.inotify.max_user_instances=512" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_watches=65536" | sudo tee -a /etc/sysctl.conf

# Refresh storage controller
kubectl rollout restart deployment local-path-provisioner -n local-path-storage
```

---

## 5. Phase 4: Cluster Management & Maintenance

Monitor health and manage system resource consumption through service controls.

### Status Monitoring

To view the status of all deployed components and verify readiness:

```bash
kubectl get pods -A
```

### Service Lifecycle

To manage the background processes and reclaim system resources when the environment is not in use:

| Action | Command |
| :--- | :--- |
| Halt Services | `sudo systemctl stop k3s` |
| Resume Services | `sudo systemctl start k3s` |
| Total Decommission | `/usr/local/bin/k3s-uninstall.sh` |

---

## 6. Functional Verification

Once the cluster reaches 100% readiness, verify the authentication flow by requesting an OpenID Connect token from the Consumer Keycloak instance.

```bash
export ACCESS_TOKEN=$(curl -s -k -x localhost:8888 -X POST \
  https://keycloak-consumer.127.0.0.1.nip.io/realms/test-realm/protocol/openid-connect/token \
  --data grant_type=password --data client_id=account-console \
  --data username=employee --data scope=openid \
  --data password=test | jq '.access_token' -r)

# Output the token to confirm successful communication
echo "${ACCESS_TOKEN}"
```

> **Note:** If the token is returned, the environment is successfully established and ready for the Data Consumer use case steps.
