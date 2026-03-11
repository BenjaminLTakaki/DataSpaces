# sovity EDC Community Edition: Netlab Dual-Node Deployment Guide

## 1. System Overview

This deployment establishes a fully functional data space on a single Ubuntu VM, consisting of two isolated sovity EDC instances: a **Data Provider** and a **Data Consumer**.

To bypass Kubernetes native routing conflicts on the VM, this setup explicitly binds the internal pod ports directly to the VM's public network interface (`0.0.0.0`).

### Environment Specifications

| Component | Details |
|---|---|
| **Host Address** | `192.168.120.12` |
| **Orchestrator** | K3s Engine |
| **Provider Node** (default namespace) | Dashboard (`3000`), API (`11002`), DSP (`11003`) |
| **Consumer Node** (consumer namespace) | Dashboard (`4000`), API (`12002`), DSP (`12003`) |

---

## 2. Phase 1: Environment Preparation

Install the K3s engine and configure the host firewall to allow traffic across all required ports.

### Step 1: Install K3s and Configure Permissions

```bash
# Install K3s engine
curl -sfL https://get.k3s.io | sh -

# Configure local user permissions
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 2: Configure the Firewall (UFW)

Open the ports for both the Provider and Consumer interfaces.

```bash
# Provider Ports
sudo ufw allow 3000/tcp
sudo ufw allow 11002/tcp
sudo ufw allow 11003/tcp

# Consumer Ports
sudo ufw allow 4000/tcp
sudo ufw allow 12002/tcp
sudo ufw allow 12003/tcp

sudo ufw reload
```

---

## 3. Phase 2: Provider Node Deployment

The Provider node operates in the **default** Kubernetes namespace.

### Step 1: Deploy Provider PostgreSQL Database

Deploy the database using the standard PostgreSQL 14.2 image.

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:14.2
          env:
            - name: POSTGRES_USER
              value: "postgres"
            - name: POSTGRES_PASSWORD
              value: "password"
            - name: POSTGRES_DB
              value: "edc"
          ports:
            - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
EOF
```

### Step 2: Apply Provider Config and Secrets

Bind the Provider's configuration to the VM's public IP address.

```bash
export VM_IP="192.168.120.12"
export DB_IP=$(kubectl get svc postgres -o jsonpath='{.spec.clusterIP}')

cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: edc-config
data:
  EDC_PARTICIPANT_ID: "provider"
  SOVITY_DEPLOYMENT_KIND: "control-plane-with-integrated-data-plane"
  SOVITY_DATASPACE_KIND: "sovity-mock-iam"
  SOVITY_EDC_FQDN_PUBLIC: "$VM_IP"
  EDC_DSP_CALLBACK_ADDRESS: "http://$VM_IP:11003/api/v1/dsp"
  SOVITY_MANAGEMENT_API_IAM_KIND: "management-iam-api-key"
---
apiVersion: v1
kind: Secret
metadata:
  name: edc-secrets
type: Opaque
stringData:
  API_AUTH_KEY: "super-secret-api-key"
  EDC_API_AUTH_KEY: "super-secret-api-key"
  SOVITY_JDBC_URL: "jdbc:postgresql://$DB_IP:5432/edc"
  SOVITY_JDBC_USER: "postgres"
  SOVITY_JDBC_PASSWORD: "password"
EOF
```

### Step 3: Deploy Provider UI and Backend

```bash
export VM_IP="192.168.120.12"

cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edc-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: edc-backend
  template:
    metadata:
      labels:
        app: edc-backend
    spec:
      containers:
        - name: edc-backend
          image: ghcr.io/sovity/edc-ce:latest
          envFrom:
            - configMapRef: { name: edc-config }
            - secretRef: { name: edc-secrets }
          env:
            - name: EDC_WEB_REST_CORS_ENABLED
              value: "true"
            - name: EDC_WEB_REST_CORS_HEADERS
              value: "origin,content-type,accept,authorization,x-api-key"
            - name: EDC_WEB_REST_CORS_ORIGINS
              value: "http://$VM_IP:3000"
          ports:
            - containerPort: 11001
            - containerPort: 11002
            - containerPort: 11003
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edc-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: edc-ui
  template:
    metadata:
      labels:
        app: edc-ui
    spec:
      containers:
        - name: edc-ui
          image: ghcr.io/sovity/edc-ce-ui:latest
          ports:
            - containerPort: 8080
          env:
            - name: NEXT_PUBLIC_ACTIVE_PROFILE
              value: "sovity-ce"
            - name: NEXT_PUBLIC_MANAGEMENT_API_URL
              value: "http://$VM_IP:11002/api/management"
            - name: NEXT_PUBLIC_MANAGEMENT_API_KEY
              value: "super-secret-api-key"
EOF
```

---

## 4. Phase 3: Consumer Node Deployment

The Consumer node is isolated in a separate Kubernetes namespace (`consumer`) and utilizes a distinct set of ports to prevent overlaps.

### Step 1: Initialize Namespace & Consumer Database

```bash
kubectl create namespace consumer

cat << 'EOF' | kubectl apply -n consumer -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:14.2
          env:
            - name: POSTGRES_USER
              value: "postgres"
            - name: POSTGRES_PASSWORD
              value: "password"
            - name: POSTGRES_DB
              value: "edc-consumer"
          ports:
            - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
EOF
```

### Step 2: Apply Consumer Config and Secrets

```bash
export VM_IP="192.168.120.12"

# Wait 10 seconds for the DB to initialize before running this line:
export DB_IP=$(kubectl get svc postgres -n consumer -o jsonpath='{.spec.clusterIP}')

cat << EOF | kubectl apply -n consumer -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: edc-config
data:
  EDC_PARTICIPANT_ID: "consumer"
  SOVITY_DEPLOYMENT_KIND: "control-plane-with-integrated-data-plane"
  SOVITY_DATASPACE_KIND: "sovity-mock-iam"
  SOVITY_EDC_FQDN_PUBLIC: "$VM_IP"
  EDC_DSP_CALLBACK_ADDRESS: "http://$VM_IP:12003/api/v1/dsp"
  SOVITY_MANAGEMENT_API_IAM_KIND: "management-iam-api-key"
---
apiVersion: v1
kind: Secret
metadata:
  name: edc-secrets
type: Opaque
stringData:
  API_AUTH_KEY: "super-secret-api-key"
  EDC_API_AUTH_KEY: "super-secret-api-key"
  SOVITY_JDBC_URL: "jdbc:postgresql://$DB_IP:5432/edc-consumer"
  SOVITY_JDBC_USER: "postgres"
  SOVITY_JDBC_PASSWORD: "password"
EOF
```

### Step 3: Deploy Consumer UI and Backend

```bash
export VM_IP="192.168.120.12"

cat << EOF | kubectl apply -n consumer -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edc-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: edc-backend
  template:
    metadata:
      labels:
        app: edc-backend
    spec:
      containers:
        - name: edc-backend
          image: ghcr.io/sovity/edc-ce:latest
          envFrom:
            - configMapRef: { name: edc-config }
            - secretRef: { name: edc-secrets }
          env:
            - name: EDC_WEB_REST_CORS_ENABLED
              value: "true"
            - name: EDC_WEB_REST_CORS_HEADERS
              value: "origin,content-type,accept,authorization,x-api-key"
            - name: EDC_WEB_REST_CORS_ORIGINS
              value: "http://$VM_IP:4000"
          ports:
            - containerPort: 11001
            - containerPort: 11002
            - containerPort: 11003
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edc-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: edc-ui
  template:
    metadata:
      labels:
        app: edc-ui
    spec:
      containers:
        - name: edc-ui
          image: ghcr.io/sovity/edc-ce-ui:latest
          ports:
            - containerPort: 8080
          env:
            - name: NEXT_PUBLIC_ACTIVE_PROFILE
              value: "sovity-ce"
            - name: NEXT_PUBLIC_MANAGEMENT_API_URL
              value: "http://$VM_IP:12002/api/management"
            - name: NEXT_PUBLIC_MANAGEMENT_API_KEY
              value: "super-secret-api-key"
EOF
```

---

## 5. Phase 4: Network Binding (Public Access)

Ensure all pods are in a `Running` state across both namespaces before proceeding:

```bash
kubectl get pods -A
```

Execute this block to bind all active deployments to the VM's public network interface.

> **Note:** This block must be re-run if the VM is rebooted or the pods are restarted.

```bash
# 1. Terminate any disconnected or zombie forwarders
sudo killall kubectl
sleep 2

# 2. Map Provider Network
kubectl port-forward deployment/edc-ui 3000:8080 --address 0.0.0.0 &
kubectl port-forward deployment/edc-backend 11002:11002 --address 0.0.0.0 &
kubectl port-forward deployment/edc-backend 11003:11003 --address 0.0.0.0 &

# 3. Map Consumer Network
kubectl port-forward -n consumer deployment/edc-ui 4000:8080 --address 0.0.0.0 &
kubectl port-forward -n consumer deployment/edc-backend 12002:11002 --address 0.0.0.0 &
kubectl port-forward -n consumer deployment/edc-backend 12003:11003 --address 0.0.0.0 &
```

---

## 6. Functional Verification

You can now access both environments simultaneously from any browser on your network.

### Access the Dashboards

- **Provider Node:** <http://192.168.120.12:3000>
- **Consumer Node:** <http://192.168.120.12:4000>

### Testing the Dataspace

1. Open the **Provider Node** dashboard.
2. Navigate to **Assets** and create a new mock data asset.
3. Navigate to **Policies** and create an open access policy.
4. Navigate to **Contract Definitions** and link your Asset to your Policy.
5. Open the **Consumer Node** dashboard.
6. Navigate to **Catalog** and click **Fetch Catalog**.
7. Enter the Provider's DSP URL: `http://192.168.120.12:11003/api/v1/dsp`.
8. The Consumer should now successfully display the Data Offer published by the Provider.