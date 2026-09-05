# ReplicaSet — Complete Reference

> Ensures a specified number of pod replicas run at all times.
> Uses **set-based selectors** — more powerful than ReplicationController.
> In practice: Deployments manage ReplicaSets. You rarely create RS directly.

---

## What is a ReplicaSet?

| Feature | Detail |
|---|---|
| **API Version** | `apps/v1` |
| **Kind** | `ReplicaSet` |
| **Short name** | `rs` |
| **Scope** | Namespaced |
| **Self-healing?** | Yes — replaces failed/deleted pods automatically |
| **Selector type** | Set-based (`matchLabels` + `matchExpressions`) |
| **Managed by** | Deployment (usually) — rarely created standalone |
| **Rolling updates?** | No — use Deployment for that |

---

## RC → RS → Deployment — Evolution

```
ReplicationController  (v1, legacy)
    equality selector only | no rolling update
          │
          ▼
    ReplicaSet  (apps/v1)
    set-based selectors | still no rolling update
          │
          ▼
    Deployment  (apps/v1)  ← production standard
    wraps RS | adds rolling update + rollback + revision history
```

| Feature | ReplicationController | ReplicaSet | Deployment |
|---|---|---|---|
| API version | `v1` | `apps/v1` | `apps/v1` |
| Selector | Equality only | `matchLabels` + `matchExpressions` | `matchLabels` + `matchExpressions` |
| Rolling update | No (deprecated `rolling-update`) | No | Yes (built-in) |
| Rollback | No | No | Yes |
| Revision history | No | No | Yes |
| Use directly? | No (legacy) | Rarely | Yes — always |

---

## How ReplicaSet Works

```
kubectl apply -f rs.yaml
        │
        ▼
ReplicaSet created — watches pods matching spec.selector
        │
        ├─ actual < desired ──► creates pods from spec.template
        │
        └─ actual > desired ──► deletes excess pods
                │
                ▼
        Desired count maintained continuously
        (loop runs in the background via the RS controller)
```

---

## YAML Structure

```yaml
apiVersion: apps/v1            # 🔴 not v1 — apps/v1 for RS
kind: ReplicaSet               # 🔴
metadata:
  name: nginx-rs               # 🔴 DNS-1123 name
  namespace: default           # optional
  labels:
    app: nginx-app             # labels on RS object — not the pod selector
spec:
  replicas: 3                  # 🔴 desired pod count (default 1 if omitted)
  selector:                    # 🔴 how RS identifies its pods — IMMUTABLE after create
    matchLabels:               # ⭐ equality match — must be subset of template labels
      app: nginx-app
    matchExpressions:          # ⭐ set-based match (optional — used with matchLabels)
      - key: env
        operator: In           # In | NotIn | Exists | DoesNotExist
        values:
          - prod
          - staging
  template:                    # 🔴 pod blueprint
    metadata:
      labels:
        app: nginx-app         # 🔴 must satisfy spec.selector
        env: prod              # must satisfy matchExpressions key/values
    spec:
      containers:
        - name: nginx-container  # 🔴 unique per pod
          image: nginx:1.25      # 🔴 pin version — no :latest
          ports:
            - containerPort: 80  # informational only
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "250m"
              memory: "128Mi"
```

---

## Field Reference

### Metadata Fields

| Field | Required | Rule | Example |
|---|---|---|---|
| `metadata.name` | Yes | DNS-1123, lowercase, ≤253 chars | `nginx-rs` |
| `metadata.namespace` | No | Defaults to `default` | `production` |
| `metadata.labels` | No | Labels on the RS object — not used for pod selection | `app: nginx-app` |

### Spec Fields

| Field | Required | Description | Key Rule |
|---|---|---|---|
| `spec.replicas` | No (default 1) | Desired pod count | Integer ≥ 0 |
| `spec.selector` | Yes | How RS finds pods | **Immutable** after creation |
| `spec.selector.matchLabels` | No* | Equality label match | *Must have matchLabels or matchExpressions |
| `spec.selector.matchExpressions` | No* | Set-based label match | Supports In, NotIn, Exists, DoesNotExist |
| `spec.template` | Yes | Pod blueprint | Labels must satisfy `spec.selector` |
| `template.metadata.labels` | Yes | Pod labels | Must be a superset of selector |
| `template.spec.containers` | Yes | Container list | Min 1 required |

---

## Selector — matchLabels vs matchExpressions ⭐

```
spec.selector
 ├── matchLabels      (equality — key must equal value)
 └── matchExpressions (set-based — key matches a condition)
```

Both blocks are ANDed together — pod must satisfy ALL conditions.

### matchLabels
```yaml
matchLabels:
  app: nginx-app    # pod must have label app=nginx-app
  tier: frontend    # AND label tier=frontend
```

### matchExpressions operators

| Operator | Meaning | Example |
|---|---|---|
| `In` | Label value is one of the listed values | `env In [prod, staging]` |
| `NotIn` | Label value is NOT in the listed values | `env NotIn [dev]` |
| `Exists` | Label key exists (any value) | `key: canary, operator: Exists` |
| `DoesNotExist` | Label key does not exist | `key: canary, operator: DoesNotExist` |

```yaml
matchExpressions:
  - key: env
    operator: In
    values: [prod, staging]     # pod env label must be prod OR staging

  - key: canary
    operator: DoesNotExist      # pod must NOT have a canary label
```

### RC vs RS Selector — Side by Side

```yaml
# ReplicationController — flat equality only
selector:
  app: nginx-app

# ReplicaSet — structured, supports expressions
selector:
  matchLabels:
    app: nginx-app
  matchExpressions:
    - key: env
      operator: In
      values: [prod]
```

---

## Pod Ownership — How RS Adopts Pods ⭐

RS owns any pod in the same namespace whose labels satisfy the selector — even pods it didn't create.

```
Existing pod: labels {app: nginx-app, env: prod}
       │
RS with matching selector is created (replicas: 3)
       │
RS counts existing pod as 1 of 3 → creates only 2 more
```

> Accidental adoption risk: always use unique label combinations per controller.

**Ownership is via `ownerReferences`** on the pod:
```yaml
# kubectl get pod <name> -o yaml
metadata:
  ownerReferences:
    - apiVersion: apps/v1
      kind: ReplicaSet
      name: nginx-rs
```

---

## RS Managed by Deployment ⭐

```
Deployment (nginx-deploy)
    │
    ├── ReplicaSet v1 (nginx-deploy-7d4f8b)  ← old version (scaled to 0)
    │
    └── ReplicaSet v2 (nginx-deploy-9c2a1d)  ← current version (replicas: 3)
              │
              ├── Pod nginx-deploy-9c2a1d-abc
              ├── Pod nginx-deploy-9c2a1d-def
              └── Pod nginx-deploy-9c2a1d-ghi
```

- Deployment creates a new RS for every rollout
- Old RSes are kept (scaled to 0) for rollback
- RS name = `<deploy-name>-<pod-template-hash>`

---

## Scaling

```bash
# Declarative — edit YAML and apply (preferred)
kubectl apply -f rs.yaml

# Imperative
kubectl scale rs nginx-rs --replicas=5

# Scale to zero — stops all pods, RS object stays
kubectl scale rs nginx-rs --replicas=0
```

> Scaling an RS managed by a Deployment: **scale the Deployment**, not the RS — the Deployment will override RS replicas.

---

## kubectl Commands for ReplicaSet ⭐

| Command | What it does |
|---|---|
| `kubectl get rs` | List all ReplicaSets in current namespace |
| `kubectl get rs -o wide` | Show selector and container info |
| `kubectl describe rs nginx-rs` | Full details, selector, pod status, events |
| `kubectl apply -f rs.yaml` | Create or update RS |
| `kubectl delete rs nginx-rs` | Delete RS and all its pods |
| `kubectl delete rs nginx-rs --cascade=orphan` | Delete RS, keep pods running |
| `kubectl scale rs nginx-rs --replicas=5` | Scale up/down |
| `kubectl get pods -l app=nginx-app` | List pods owned by RS selector |
| `kubectl get rs nginx-rs -o yaml` | Inspect live YAML |
| `kubectl edit rs nginx-rs` | Edit RS in-place |
| `kubectl get rs -l app=nginx-app` | Find RS by label |

---

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `selector does not match template labels` | matchLabels keys not present in template labels | Ensure template labels are a superset of selector |
| `field is immutable: spec.selector` | Tried to change selector after creation | Delete RS and recreate with new selector |
| RS not owning expected pods | Pod labels don't satisfy matchExpressions | Check each expression — all must match |
| Pods not created | `replicas: 0` or selector mismatch | Check selector vs template labels; scale up |
| RS fighting Deployment | Scaled RS directly instead of Deployment | Always scale the Deployment — it overrides RS |
| Accidental pod adoption | Another pod has overlapping labels | Use unique label sets; check ownerReferences |
| `no matches for kind ReplicaSet in version v1` | Used wrong apiVersion | Must be `apps/v1`, not `v1` |

---

## ReplicaSet vs Deployment — When to Use Which

| Use RS directly | Use Deployment |
|---|---|
| You need a fixed set of pods with no updates | Any real app that needs updates |
| Custom controller managing RS for you | 99% of production workloads |
| Testing/learning the RS concept | When you need rollback or rollout history |

> **Rule of thumb:** if you're asking "should I use RS or Deployment?" — use Deployment.

---

## Best Practices

- **Never create standalone RS in production** — wrap it in a Deployment
- **Immutable selector** — plan your labels before creating; changing selector requires delete + recreate
- **Template labels must satisfy selector** — every key in matchLabels must appear in template labels
- **Use unique labels** — prevent accidental pod adoption by other controllers
- **Always set `resources.requests` and `limits`** — required for stable scheduling and OOM prevention
- **Don't scale RS directly** if it's managed by a Deployment — the Deployment will override it
- **Pin image versions** — never `:latest` in controller templates
- **matchExpressions with `Exists`/`DoesNotExist`** — useful for canary or feature-flag pod selection

---

## selector Immutability — Key Rule ⭐

```
Create RS with selector app=nginx
       │
Try to change selector to app=frontend
       │
       ▼
ERROR: spec.selector: field is immutable
```

- Delete RS and recreate with the new selector
- Use `--cascade=orphan` to keep pods running during the switch
- **Plan selectors carefully before first apply**

---

*ReplicaSet is the engine behind Deployment's self-healing — understand it and Deployment becomes transparent.*
