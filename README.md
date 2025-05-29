# Vault + Kubernetes Authentication & CSI Integration
**Environment:** Ubuntu 24.04 + Docker Compose + Kubernetes Cluster (Secrets Store CSI Driver)
![image](https://github.com/user-attachments/assets/5883517a-345f-45f9-b170-280c26e50fbe)

## 1. Prepare Vault Project Directory
```bash
mkdir -p ~/vault-project/vault-config
mkdir -p ~/vault-project/vault-data
cd ~/vault-project
```

## 2. Vault Configuration File (`vault-config/config.hcl`)
```hcl
# vi vault-config/config.hcl
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}

storage "file" {
  path = "/vault/file"
}

ui = true
api_addr = "http://192.168.55.40:8200"
```

## 3. Docker Compose File (`docker-compose.yml`)
```yaml
# vi docker-compose.yml
services:
  vault:
    image: hashicorp/vault:latest
    container_name: vault-project-vault-1
    ports:
      - "8200:8200"
    cap_add:
      - IPC_LOCK
    volumes:
      - ./vault-config:/vault/config:ro
      - ./vault-data:/vault/file
    environment:
      VAULT_ADDR: "http://192.168.55.40:8200"
    command: vault server -config=/vault/config/config.hcl
```

## 4. Start Vault
```bash
docker compose up -d
docker ps
```

## 5. Initialize Vault
```bash
docker exec -it vault-project-vault-1 vault operator init -address=http://127.0.0.1:8200
```
⚠️ **Important:** Save the unseal keys and root token in a secure location.

### 5.1 Unseal Vault
Open your browser, navigate to:
```
http://192.168.55.40:8200/ui
```
Log in and unseal Vault v1.19.4

## 6. Enable Kubernetes Auth Method in Vault Container
```bash
docker exec -it vault-project-vault-1 sh
VAULT_ADDR="http://127.0.0.1:8200"
vault login  ## insert Root Token here
vault auth enable kubernetes
```

## 7. Create Kubernetes Service Account (`vault-sa.yaml`)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-auth-sa
  namespace: default
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-auth-secret
  annotations:
    kubernetes.io/service-account.name: vault-auth-sa
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: secret-reader-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault-auth-sa
  namespace: default
```

```bash
kubectl apply -f vault-sa.yaml
```

## 8. Configure Kubernetes Auth in Vault UI
Navigate to:
```
http://192.168.55.40:8200
```
Go to: Access > Authentication Methods > Kubernetes > Configuration Tab > Configure

![image](https://github.com/user-attachments/assets/935a5b17-68fa-4933-9922-115db44e4933)

- **Kubernetes Host:** `https://192.168.55.24:6443`
- **Kubernetes CA Certificate:**
```bash
cat /etc/kubernetes/pki/ca.crt
```
Paste the contents.
- **Token Reviewer JWT:**
```bash
export K8S_TOKEN=$(kubectl get secret vault-auth-secret -o jsonpath="{.data.token}" | base64 -d)
echo $K8S_TOKEN
```
Paste the token.

## 9. Create Vault Secret
```bash
docker exec -it vault-project-vault-1 sh
VAULT_ADDR="http://127.0.0.1:8200"
vault login  ## insert Root Token here

vault secrets enable -path=/secret kv-v2
vault kv put secret/db-pass pwd="admin@123"
vault kv get secret/db-pass
```

### 9.1 Create Vault Policy
```bash
vault policy write internal-app - <<EOF
# For KV version 2, use the 'data/' prefix
path "secret/data/db-pass" {
  capabilities = ["read"]
}
path "secret/metadata/db-pass" {
  capabilities = ["read"]
}
EOF

vault policy read internal-app
```

### 9.2 Bind Vault Role to Kubernetes SA
```bash
vault write auth/kubernetes/role/database     bound_service_account_names=vault-auth-sa     bound_service_account_namespaces=default     policies=internal-app     ttl=20m

vault read auth/kubernetes/role/database
```

## 10. Install Vault CSI Driver on Kubernetes
```bash
# Add Secrets Store CSI Driver repo
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

# Install the driver
helm upgrade -i csi secrets-store-csi-driver/secrets-store-csi-driver     --set syncSecret.enabled=true     --set enableSecretRotation=true     --set rotationPollInterval=30s

# Verify installation
kubectl get ds
kubectl api-resources | grep -i csi
kubectl get csidriver
kubectl get csinodes
kubectl get po
```

### Install Vault CSI Provider
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com

helm install vault hashicorp/vault     --set "server.enabled=false"     --set "injector.enabled=false"     --set "csi.enabled=true"
```

## 11. Create SecretProviderClass (`spc-crd-vault.yaml`)
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-database
spec:
  provider: vault
  parameters:
    vaultAddress: "http://192.168.55.40:8200"
    roleName: "database"
    objects: |
      - objectName: "db-pass"
        secretPath: "secret/data/db-pass"
        secretKey: "pwd"
```

```bash
kubectl apply -f spc-crd-vault.yaml
kubectl api-resources | grep -i secret
kubectl get secretproviderclasses
kubectl describe secretproviderclasses vault-database
```

## 12. Create Pod That Mounts Vault Secret (`webapp.yaml`)
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: webapp2
spec:
  serviceAccountName: vault-auth-sa
  containers:
  - image: jweissig/app:0.0.1
    name: webapp
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets-store"
      readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "vault-database"
```

```bash
kubectl apply -f webapp.yaml
kubectl get pod
```

---

##  Secret Usage Flow: From Vault to Pod

### 1.  Store Secret in Vault
```bash
vault kv put secret/db-pass pwd="admin@123"
```

### 2.  Vault Policy and Role
```hcl
path "secret/data/db-pass" {
  capabilities = ["read"]
}
```

```bash
vault write auth/kubernetes/role/database     bound_service_account_names=vault-auth-sa     bound_service_account_namespaces=default     policies=internal-app     ttl=20m
```

### 3.  Define SecretProviderClass
```yaml
parameters:
  vaultAddress: "http://192.168.55.40:8200"
  roleName: "database"
  objects: |
    - objectName: "db-pass"
      secretPath: "secret/data/db-pass"
      secretKey: "pwd"
```

### 4.  Mount Secret in Pod
```yaml
volumeMounts:
  - name: secrets-store-inline
    mountPath: "/mnt/secrets-store"
```

```bash
kubectl exec -it webapp2 -- sh
cat /mnt/secrets-store/db-pass
# Output: admin@123
```

###  Application Usage (Example in Python)
```python
file = open("/mnt/secrets-store/db-pass").read().strip()
```

##  Key Benefits
- No secret stored in etcd.
- Mounted as file, not environment variable.
- Dynamic and ephemeral from Vault.
- Restricted access via service account.
