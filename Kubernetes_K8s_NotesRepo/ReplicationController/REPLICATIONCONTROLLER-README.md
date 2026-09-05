# ReplicationController — Complete Reference

> Ensures a specified number of pod replicas are running at all times.
> **Legacy object** — superseded by ReplicaSet (and Deployment). Still exam-relevant and found in older clusters.

---

## What is a ReplicationController?

| Feature | Detail |
|---|---|
| **API Version** | `v1` (core group — not `apps/v1`) |
| **Kind** | `ReplicationController` |
| **Short name** | `rc` |
| **Scope** | Namespaced |
| **Self-healing?** | Yes — replaces failed/deleted pods automatically |
| **Selector type** | Equality-based only (`key: value`) — no `matchLabels` / `matchExpressions` |
| **Superseded by** | `ReplicaSet` (set-based selectors) → `Deployment` (adds rollouts) |

---

## RC vs ReplicaSet vs Deployment

```
ReplicationController  (v1, legacy)
       │  equality selector only
       │  no rolling update built-in
       ▼
  ReplicaSet  (apps/v1)
       │  set-based selectors (matchLabels, matchExpressions)
       │  still no rolling update on its own
       ▼
  Deployment  (apps/v1)  ← USE THIS in production
       │  wraps ReplicaSet
       │  adds rolling updates, rollbacks, revision history
```

| Feature | ReplicationController | ReplicaSet | Deployment |
|---|---|---|---|
| API version | `v1` | `apps/v1` | `apps/v1` |
| Selector type | Equality only | Set-based | Set-based |
| Rolling update | Manual (`kubectl rolling-update`) | No | Yes (built-in) |
| Rollback | No | No | Yes |
| Use in prod? | No (legacy) | Rarely (managed by Deployment) | Yes |

---

## How RC Works

```
kubectl apply -f rc.yaml
       │
       ▼
ReplicationController created
       │
       ├── Watches pods matching spec.selector
       │
       ├── Too few pods? ──► Creates new pods from spec.template
       │
       └── Too many pods? ──► Deletes excess pods
               │
               ▼
       Desired count maintained at all times
```

---

## YAML Structure

```yaml
apiVersion: v1                      # 🔴 core group — not apps/v1
kind: ReplicationController         # 🔴
metadata:
  name: nginx-rc                    # 🔴 DNS-1123 compliant name
  namespace: default                # optional
  labels:
    app: nginx-app                  # labels on the RC object itself
spec:
  replicas: 3                       # 🔴 desired pod count (default: 1 if omitted)
  selector:                         # 🔴 RC finds pods using this — equality only
    app: nginx-app                  # must exactly match template.metadata.labels
  template:                         # 🔴 pod blueprint
    metadata:
      labels:
        app: nginx-app              # 🔴 MUST match spec.selector
    spec:
      containers:
        - name: nginx-container     # 🔴 unique per pod
          image: nginx:1.25         # 🔴 pin the version
          ports:
            - containerPort: 80     # informational
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
```

---

## Field Reference

### Metadata Fields

| Field | Required | Rule | Example |
|---|---|---|---|
| `metadata.name` | Yes | DNS-1123, ≤253 chars, lowercase | `nginx-rc` |
| `metadata.namespace` | No | Defaults to `default` | `production` |
| `metadata.labels` | No | Labels on the RC object itself | `app: nginx-app` |

### Spec Fields

| Field | Required | Description | Key Rule |
|---|---|---|---|
| `spec.replicas` | No (default: 1) | Desired pod count | Integer ≥ 0 |
| `spec.selector` | Yes | Label equality match to find pods | Must exactly match `template.metadata.labels` |
| `spec.template` | Yes | Pod blueprint (same as a Pod spec) | Labels must satisfy `spec.selector` |
| `template.metadata.labels` | Yes | Labels on created pods | Must match `spec.selector` — no match = RC broken |
| `template.spec.containers` | Yes | List of containers | Min 1 container required |
| `containers[].name` | Yes | Container name | Unique per pod |
| `containers[].image` | Yes | Docker image | Pin a version tag |

---

## Selector — Equality Only ⭐

RC uses simple key=value matching. No advanced expressions.

```yaml
# RC selector (equality only)
selector:
  app: nginx-app         # finds pods where label app=nginx-app

# What RC CANNOT do (this is ReplicaSet syntax — will fail):
selector:
  matchLabels:
    app: nginx-app
  matchExpressions:
    - key: env
      operator: In
      values: [prod, staging]
```

| Object | Selector Syntax | Supports `In`, `NotIn`, `Exists`? |
|---|---|---|
| ReplicationController | `key: value` directly | No |
| ReplicaSet | `matchLabels` + `matchExpressions` | Yes |

---

## Pod Ownership — How RC Adopts Pods

RC owns any pod in the namespace that matches its `spec.selector` — even pods it didn't create.

```
Existing pod with label app=nginx-app
       │
       ▼
RC with selector app=nginx-app is created
       │
       ▼
RC immediately counts that pod as one of its replicas
(if replicas=3 and 1 exists → RC creates 2 more)
```

> This is a common gotcha — labels must be unique per RC to avoid accidental adoption.

---

## Scaling

```bash
# Declarative (preferred)
# Edit replicas in YAML and kubectl apply -f

# Imperative
kubectl scale rc nginx-rc --replicas=5

# Scale to zero (stops all pods, keeps RC)
kubectl scale rc nginx-rc --replicas=0
```

---

## kubectl Commands for RC ⭐

| Command | What it does |
|---|---|
| `kubectl get rc` | List all RCs in current namespace |
| `kubectl get rc -o wide` | Show with selector and container info |
| `kubectl describe rc nginx-rc` | Full details, selector, pod status, events |
| `kubectl apply -f rc.yaml` | Create or update RC |
| `kubectl delete rc nginx-rc` | Delete RC and all its pods |
| `kubectl delete rc nginx-rc --cascade=orphan` | Delete RC but keep pods running |
| `kubectl scale rc nginx-rc --replicas=5` | Scale up/down |
| `kubectl get pods -l app=nginx-app` | List pods owned by this RC |
| `kubectl edit rc nginx-rc` | Edit RC live in editor |
| `kubectl get rc nginx-rc -o yaml` | Inspect live YAML |

---

## Deleting RC — Cascade Options

| Command | Pods after deletion |
|---|---|
| `kubectl delete rc nginx-rc` | Pods are also deleted |
| `kubectl delete rc nginx-rc --cascade=orphan` | Pods survive — become unmanaged |

> Use `--cascade=orphan` when migrating from RC to Deployment without downtime.

---

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| Pods not created | `spec.selector` doesn't match `template.metadata.labels` | Make selector and template labels identical |
| RC adopts wrong pods | Another RC/pod has the same labels | Use unique labels per RC |
| Stuck at 0 pods | `replicas: 0` set | Scale up: `kubectl scale rc <name> --replicas=N` |
| `matchLabels is not supported` | Used ReplicaSet syntax in RC | RC uses flat `selector: key: value` — no `matchLabels` block |
| Pods not deleted after RC delete | `--cascade=orphan` was used | Delete orphaned pods manually with label selector |
| Image pull error | Bad image name or tag | Fix image field; check tag exists in registry |

---

## Best Practices

- **Don't use RC in new projects** — use Deployment which manages ReplicaSets and adds rolling updates
- **If you must use RC**, always set `replicas` explicitly — don't rely on the default of 1
- **Make labels unique** — a pod accidentally matching two RCs causes unpredictable scaling
- **Always set `resources.requests` and `limits`** — even on RC pods, for stable scheduling
- **Use `--cascade=orphan`** when replacing RC with a Deployment to avoid pod downtime
- **RC selector is immutable after creation** — delete and recreate if you need to change it
- **Pin image versions** — never `:latest` in any controller template

---

## Migration: RC → Deployment (Zero Downtime)

```
Step 1: kubectl delete rc nginx-rc --cascade=orphan
        (pods stay running, now unmanaged)

Step 2: kubectl apply -f deployment.yaml
        (Deployment adopts pods with matching labels via ReplicaSet)

Step 3: Verify
        kubectl get pods -l app=nginx-app
        kubectl rollout status deployment/nginx-deploy
```

---

## Quick Comparison — RC Spec vs ReplicaSet Spec

```yaml
# ReplicationController
spec:
  replicas: 3
  selector:            # flat equality
    app: nginx-app
  template: ...

# ReplicaSet
spec:
  replicas: 3
  selector:            # structured — supports expressions
    matchLabels:
      app: nginx-app
  template: ...
```

---

*ReplicationController is the original self-healing controller — understand it to understand why ReplicaSet and Deployment were built.*
