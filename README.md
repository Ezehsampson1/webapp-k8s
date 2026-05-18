# webapp — Kubernetes GitOps Deployment

A GitOps-friendly Kubernetes deployment for `webapp` (nginx) across `staging` and `production` environments, using Kustomize for manifest management and Sealed Secrets for safe secret handling in a public repository.

---

## Repository Structure

```
k8s/
├── base/
│   ├── deployment.yaml       
│   ├── service.yaml          
│   └── kustomization.yaml    
├── overlays/
│   ├── staging/
│   │   ├── namespace.yaml        
│   │   ├── replicas-patch.yaml  
│   │   ├── sealed-secret.yaml    
│   │   └── kustomization.yaml    
│   └── production/
│       ├── namespace.yaml        
│       ├── replicas-patch.yaml   
│       ├── sealed-secret.yaml    
│       └── kustomization.yaml    
└── infrastructure/
    └── sealed-secrets-controller.yaml  
```

---

## Prerequisites

- `kubectl` configured against your target cluster
- `kustomize` v5+ (or `kubectl` v1.27+ which bundles kustomize)
- Sealed Secrets controller installed in the cluster (see [Secret Management](#secret-management) below)
- `kubeseal` CLI for sealing new secrets

---

## Deploying to Each Environment

> **Note:** The Sealed Secrets controller must be installed first — see [Secret Management](#secret-management).  
> The `sealed-secret.yaml` files contain placeholder ciphertext. Before deploying, replace them with values encrypted by your cluster's actual key (see instructions below).

### Staging

```bash
kubectl apply -k k8s/overlays/staging
```

This applies, in order: the `staging` namespace, the base Deployment and Service (scoped to `staging`, image pinned to `nginx:1.25`, 1 replica), and the SealedSecret which the controller decrypts into a standard Kubernetes Secret.

### Production

```bash
kubectl apply -k k8s/overlays/production
```

Same process — `production` namespace, `nginx:1.27`, 3 replicas.

### Verify

```bash
# Check rollout status
kubectl rollout status deployment/webapp -n staging
kubectl rollout status deployment/webapp -n production

# Confirm image tag and replica count
kubectl get deployment webapp -n staging -o wide
kubectl get deployment webapp -n production -o wide

# Confirm secret was decrypted correctly
kubectl get secret webapp-secret -n staging
```

---

## Secret Management

### Approach: Sealed Secrets (Bitnami)

Kubernetes `Secret` manifests store values as base64, which is **not encryption** — anyone with `kubectl get secret` access or Git history access can decode them instantly. Committing them to a public repository would expose credentials in plaintext.

**Sealed Secrets** solves this by encrypting the secret with the cluster's public key using asymmetric cryptography (RSA-OAEP). The resulting `SealedSecret` manifest is safe to commit publicly because:

- Only the cluster that holds the matching private key can decrypt it
- A secret sealed for `staging` cannot be decrypted in `production` (namespace-scoped by default)
- The private key never leaves the cluster

This makes the repository genuinely safe to make public, with no `.gitignore` workarounds or placeholder values.

### Installing the Controller (one-time, per cluster)

```bash
# Option A: Helm (recommended for production)
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --version 2.15.3

# Option B: kubectl direct apply
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.3/controller.yaml
```

Verify it's running:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

### Sealing a New Secret

```bash
# 1. Fetch the cluster's public key
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  > pub-cert.pem

# 2. Create the plaintext secret (DO NOT COMMIT THIS FILE)
kubectl create secret generic webapp-secret \
  --namespace staging \
  --from-literal=DB_PASSWORD='your-actual-db-password' \
  --from-literal=API_KEY='your-actual-api-key' \
  --dry-run=client -o yaml > /tmp/secret.yaml

# 3. Encrypt it
kubeseal --format yaml \
  --cert pub-cert.pem \
  --namespace staging \
  < /tmp/secret.yaml > k8s/overlays/staging/sealed-secret.yaml

# 4. Delete the plaintext immediately
rm /tmp/secret.yaml

# 5. Commit the sealed file — it's safe
git add k8s/overlays/staging/sealed-secret.yaml
git commit -m "chore: update webapp-secret for staging"
```

Repeat for production with `--namespace production`.

---

## Kustomize Build Output

The rendered manifests for each environment (equivalent to `kustomize build k8s/overlays/<env>`) are included below for reviewers without a cluster.

<details>
<summary>Staging build output</summary>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    environment: staging
  name: staging
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: webapp
    environment: staging
  name: webapp
  namespace: staging
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: webapp
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: webapp
    environment: staging
  name: webapp
  namespace: staging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        environment: staging
    spec:
      containers:
      - env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: DB_PASSWORD
              name: webapp-secret
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              key: API_KEY
              name: webapp-secret
        image: nginx:1.25
        name: webapp
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 250m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 128Mi
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: webapp-secret
  namespace: staging
spec:
  encryptedData:
    API_KEY: AgCZ9m1y...
    DB_PASSWORD: AgBY3k2x...
  template:
    metadata:
      name: webapp-secret
      namespace: staging
    type: Opaque
```

</details>

<details>
<summary>Production build output</summary>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    environment: production
  name: production
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: webapp
    environment: production
  name: webapp
  namespace: production
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: webapp
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: webapp
    environment: production
  name: webapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        environment: production
    spec:
      containers:
      - env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: DB_PASSWORD
              name: webapp-secret
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              key: API_KEY
              name: webapp-secret
        image: nginx:1.27
        name: webapp
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 250m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 128Mi
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: webapp-secret
  namespace: production
spec:
  encryptedData:
    API_KEY: AgDA0n2z...
    DB_PASSWORD: AgBZ4l3w...
  template:
    metadata:
      name: webapp-secret
      namespace: production
    type: Opaque
```

</details>

---

## Assumptions and Trade-offs

### Assumptions

- A single cluster hosts both environments, separated by namespace. In a real production setup, staging and production would typically be separate clusters to provide stronger fault and blast-radius isolation.
- The `ClusterIP` service type is intentional — exposing the app via Ingress or LoadBalancer is infrastructure-specific and out of scope for this assessment.
- Resource requests and limits are included in the base deployment as a production-awareness practice; the assessment did not specify them but leaving them absent would be a real-world mistake.

### Trade-offs

**Sealed Secrets vs External Secrets Operator (ESO)**

Sealed Secrets requires no external dependency beyond the cluster itself, which makes it ideal for this assessment. In a team environment with existing cloud infrastructure (AWS, GCP, Azure), ESO pulling from a managed secrets service (AWS Secrets Manager, GCP Secret Manager) would be preferable: secret rotation doesn't require re-sealing and re-committing, and access is audited at the provider level. The right choice depends on what the organisation already operates.

**Sealed Secrets vs SOPS + age**

SOPS is more portable (works outside Kubernetes too) and doesn't require an in-cluster component. The trade-off is that it requires key management discipline — if the decryption key is lost, secrets are unrecoverable. Sealed Secrets delegates key management to the cluster, which aligns well with GitOps patterns where the cluster is the source of truth.

**Single cluster vs multi-cluster**

Namespace-based isolation is simple and sufficient for this assessment. The risk is that a cluster-level failure (or misconfigured RBAC) could allow the staging team to access production secrets. In production, separate clusters eliminate this risk and allow independent upgrade schedules.

---

