# Securing Kubernetes Clusters using RBAC

A hands-on demo that secures a REST API running in Kubernetes using **Role-Based Access Control (RBAC)** and **kube-rbac-proxy** as an authorization sidecar.

---

## Architecture

```
curl (Bearer $TOKEN)
      │
      ▼
┌─────────────────────────────────────────┐
│  Pod: grade-submission-api              │
│                                         │
│  ┌──────────────────┐  ┌─────────────┐ │
│  │  kube-rbac-proxy │──│  API :3000  │ │
│  │  :8443 (HTTPS)   │  │  (Node.js)  │ │
│  └──────────────────┘  └─────────────┘ │
└─────────────────────────────────────────┘
      │
      ├─► TokenReview API   (validates Bearer token)
      └─► SubjectAccessReview API  (checks RBAC policy)
```

Incoming requests hit kube-rbac-proxy on port `8443`. The proxy:
1. Validates the Bearer token via the Kubernetes **TokenReview** API.
2. Checks authorization via the **SubjectAccessReview** API against the RBAC rules.
3. Forwards only authorized requests to the upstream API on port `3000`.

---

## Files

| File | Purpose |
|------|---------|
| `00-namespace.yaml` | Creates the `grade-demo` namespace |
| `01-grade-submission-api-deployment.yaml` | Deploys the API container alongside a kube-rbac-proxy sidecar |
| `02-grade-submission-api-service.yaml` | Exposes the deployment on port `8443` |
| `03-rbac.yaml` | `ServiceAccount` + `ClusterRole` + `ClusterRoleBinding` for the API caller — defines **what requests are allowed** |
| `04-secret.yaml` | Long-lived token secret bound to `grade-service-account` (used as `$TOKEN`) |
| `05-rbac-proxy.yaml` | `ServiceAccount` + `ClusterRole` + `ClusterRoleBinding` for the proxy sidecar — grants it permission to call `tokenreviews` and `subjectaccessreviews` |

---

## How It Works

Two service accounts serve distinct roles:

**`grade-service-account`** (caller identity)
- Its token is exported as `$TOKEN` and sent in every `curl` request.
- The `ClusterRole` in `03-rbac.yaml` grants it `get` and `create` on `/*`.
- `create` is the RBAC verb kube-rbac-proxy maps to HTTP `POST`.

**`grade-service-proxy`** (proxy identity)
- Used by the kube-rbac-proxy sidecar container inside the pod.
- Granted permission to call `tokenreviews` and `subjectaccessreviews` so it can validate and authorize incoming requests.

---

## Setup

### Prerequisites
- A running Kubernetes cluster (e.g. `minikube`, `kind`, Docker Desktop)
- `kubectl` configured

### Deploy

```bash
kubectl apply -f .
```

### Get the Bearer token

```bash
TOKEN=$(kubectl get secret grade-service-account-token \
  -n grade-demo \
  -o jsonpath='{.data.token}' | base64 --decode)
```

### Port-forward the service

```bash
kubectl port-forward svc/grade-submission-api 8443:8443 -n grade-demo
```

### Test

```bash
# GET all grades
curl -k https://localhost:8443/grades \
  -H "Authorization: Bearer $TOKEN"

# POST a new grade
curl -k -X POST https://localhost:8443/grades \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name": "Ron", "subject": "Charms", "score": 82}'
```

---

## Key Concepts

- **RBAC** — Kubernetes-native authorization using `Roles`/`ClusterRoles` bound to subjects via `RoleBindings`/`ClusterRoleBindings`.
- **kube-rbac-proxy** — A sidecar proxy that enforces RBAC in front of any HTTP workload without modifying the app itself.
- **TokenReview** — API used to validate a Bearer token and resolve it to a subject (service account).
- **SubjectAccessReview** — API used to check whether a subject is permitted to perform a given action.
- **RBAC verb mapping** — HTTP methods map to RBAC verbs: `GET→get`, `POST→create`, `PUT→update`, `DELETE→delete`, `PATCH→patch`.
