# 📘 The Complete Kubernetes YAML Handbook & Cheat Sheet

> A beginner-to-advanced reference for writing, validating, and debugging Kubernetes manifests.
> Built for **revision, interview preparation, and day-to-day DevOps work**.

**Kubernetes version baseline:** This guide targets **Kubernetes v1.29 – v1.31** (current stable line as of 2025–2026). API versions and notes call out version differences where relevant. Always confirm with `kubectl api-versions` on your own cluster.

---

## 📑 Table of Contents

1. [How to Use This Guide](#-how-to-use-this-guide)
2. [Universal YAML Anatomy](#-universal-yaml-anatomy-every-object)
3. [Core Workloads](#-core-workloads)
   - [Pod](#1-pod)
   - [ReplicaSet](#2-replicaset)
   - [Deployment](#3-deployment)
   - [StatefulSet](#4-statefulset)
   - [DaemonSet](#5-daemonset)
   - [Job](#6-job)
   - [CronJob](#7-cronjob)
4. [Networking](#-networking)
   - [Service](#8-service)
   - [Ingress](#9-ingress)
   - [NetworkPolicy](#10-networkpolicy)
5. [Configuration & Storage](#-configuration--storage)
   - [ConfigMap](#11-configmap)
   - [Secret](#12-secret)
   - [PersistentVolume (PV)](#13-persistentvolume-pv)
   - [PersistentVolumeClaim (PVC)](#14-persistentvolumeclaim-pvc)
6. [Scaling](#-scaling)
   - [HorizontalPodAutoscaler (HPA)](#15-horizontalpodautoscaler-hpa)
7. [Organization & RBAC](#-organization--rbac)
   - [Namespace](#16-namespace)
   - [ServiceAccount](#17-serviceaccount)
   - [Role](#18-role)
   - [RoleBinding](#19-rolebinding)
   - [ClusterRole](#20-clusterrole)
   - [ClusterRoleBinding](#21-clusterrolebinding)
8. [Cross-Cutting Reference](#-cross-cutting-reference)
   - [Naming Conventions](#naming-conventions-all-objects)
   - [Indentation & YAML Syntax Rules](#indentation--yaml-syntax-rules)
   - [Validation Rules](#important-validation-rules)
   - [kubectl Validation Commands](#common-kubectl-validation-commands)
   - [Common Errors & Troubleshooting](#common-errors--troubleshooting)
   - [Best Practices](#best-practices)
   - [Interview Quick-Fire](#interview-quick-fire)

---

## 🚀 How to Use This Guide

- **Learning?** Read top to bottom. Each object builds on the previous.
- **Revising for an interview?** Jump to [Interview Quick-Fire](#interview-quick-fire) and the per-object "Interview Hotspots".
- **Debugging?** Go straight to [Common Errors & Troubleshooting](#common-errors--troubleshooting).
- **Mandatory fields** are marked 🔴. **Interview-favorite fields** are marked ⭐.

---

## 🧬 Universal YAML Anatomy (Every Object)

Every Kubernetes manifest shares the same four top-level keys.

| YAML Field | Required/Optional | Description | Important Rules | Example Value |
|---|---|---|---|---|
| `apiVersion` 🔴 | Required | API group + version that defines the object schema | Must match the `kind`; check with `kubectl api-versions`. Wrong pairing = "no matches for kind" | `apps/v1` |
| `kind` 🔴 | Required | The object type | Case-sensitive, PascalCase | `Deployment` |
| `metadata` 🔴 | Required | Identity: name, namespace, labels, annotations | `metadata.name` is mandatory; must be DNS-compliant | `{name: web, namespace: prod}` |
| `metadata.name` 🔴 | Required | Unique name within (namespace + kind) | ≤253 chars, lowercase, RFC 1123 | `web-frontend` |
| `metadata.namespace` | Optional | Namespace placement (namespaced objects only) | Defaults to `default`; ignored for cluster-scoped objects | `production` |
| `metadata.labels` ⭐ | Optional | Key/value pairs for selection & grouping | Used by selectors, very important for Services/Deployments | `app: web` |
| `metadata.annotations` | Optional | Non-identifying metadata for tools/humans | Not used for selection; can hold large values | `kubernetes.io/change-cause: "v2"` |
| `spec` 🔴* | Required* | Desired state — the heart of the object | Schema depends on `kind`. *Not used by ConfigMap/Secret/Namespace | `{replicas: 3, ...}` |
| `status` | Auto | Actual state, **managed by Kubernetes** | Never set this by hand; it is read-only output | `{availableReplicas: 3}` |

**`labels` vs `annotations` — a classic interview question:**

| Aspect | `labels` | `annotations` |
|---|---|---|
| Purpose | Identify & **select** objects | Attach arbitrary metadata |
| Used by selectors? | ✅ Yes | ❌ No |
| Size limit | 63 chars per value | Large values allowed (up to 256 KB total metadata) |
| Example | `env: prod`, `app: api` | `prometheus.io/scrape: "true"` |

---

## 🛠 Core Workloads

### 1. Pod

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| Pod | Smallest deployable unit — one or more co-located containers sharing network & storage | Debugging, one-off tasks, building block for controllers | `v1` | Namespaced |

> **Reality check:** You rarely create bare Pods in production. Controllers (Deployment, StatefulSet, etc.) create them for you so they self-heal.

#### Production-Grade YAML

```yaml
apiVersion: v1                      # 🔴 core API group ("" → just v1)
kind: Pod                           # 🔴 object type
metadata:
  name: web-pod                     # 🔴 must be DNS-1123 compliant
  namespace: production             # optional, defaults to "default"
  labels:                           # ⭐ used by Services/selectors
    app: web
    tier: frontend
spec:
  containers:                       # 🔴 at least one container required
    - name: web                     # 🔴 unique within the pod, DNS-1123 label
      image: nginx:1.27.0           # 🔴 PIN the version, never use :latest
      imagePullPolicy: IfNotPresent # IfNotPresent | Always | Never
      ports:
        - containerPort: 80         # informational; does not "open" anything
          name: http
      env:                          # ⭐ environment variables
        - name: LOG_LEVEL
          value: "info"             # quote values to keep them strings
        - name: DB_PASSWORD
          valueFrom:                # pull from a Secret (best practice)
            secretKeyRef:
              name: db-secret
              key: password
      resources:                    # ⭐ requests = scheduling, limits = cap
        requests:
          cpu: "100m"               # 0.1 CPU core
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      livenessProbe:                # ⭐ restart container if this fails
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 10
      readinessProbe:               # ⭐ remove from Service endpoints if failing
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 5
  nodeSelector:                     # simple node targeting
    disktype: ssd
  tolerations:                      # allow scheduling onto tainted nodes
    - key: "dedicated"
      operator: "Equal"
      value: "web"
      effect: "NoSchedule"
  restartPolicy: Always             # Always (default) | OnFailure | Never
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `spec.containers` 🔴 | Required | List of containers in the pod | Must have ≥1; it's a **list** (`-`) | `- name: web` |
| `containers[].name` 🔴 | Required | Container identifier | DNS-1123 label, unique per pod | `web` |
| `containers[].image` 🔴 | Required | Container image reference | Pin a tag/digest; avoid `latest` | `nginx:1.27.0` |
| `containers[].ports` | Optional | Documents exposed ports | Purely informational; Services route by selector | `containerPort: 80` |
| `containers[].env` ⭐ | Optional | Environment variables | Use `valueFrom` for Secrets/ConfigMaps | `LOG_LEVEL=info` |
| `containers[].resources` ⭐ | Optional (best practice: required) | CPU/memory requests & limits | Requests ≤ limits; no limit = can starve node | see YAML |
| `livenessProbe` ⭐ | Optional | Health check → restarts container on failure | Tune `initialDelaySeconds` to avoid restart loops | httpGet `/healthz` |
| `readinessProbe` ⭐ | Optional | Gate for receiving traffic | Failing readiness removes pod from Service | httpGet `/ready` |
| `nodeSelector` | Optional | Hard node placement by label | Node must have matching label or pod stays Pending | `disktype: ssd` |
| `affinity` ⭐ | Optional | Advanced node/pod (anti-)affinity rules | More expressive than `nodeSelector` | see notes |
| `tolerations` ⭐ | Optional | Lets pod schedule onto tainted nodes | Must match a node taint key/effect | `NoSchedule` |
| `restartPolicy` | Optional | Restart behavior | `Always` for Pods/Deployments; `OnFailure`/`Never` for Jobs | `Always` |

**Interview Hotspots:** difference between `requests` and `limits`; liveness vs readiness vs startup probes; what `imagePullPolicy: IfNotPresent` does; why bare pods don't self-heal; taints/tolerations vs node affinity.

---

### 2. ReplicaSet

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| ReplicaSet | Maintains a stable set of replica Pods | Backing controller for Deployments (rarely used directly) | `apps/v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: apps/v1                 # 🔴
kind: ReplicaSet                    # 🔴
metadata:
  name: web-rs
  labels:
    app: web
spec:
  replicas: 3                       # ⭐ desired pod count
  selector:                         # 🔴 MUST match template labels
    matchLabels:
      app: web
  template:                         # 🔴 pod template
    metadata:
      labels:
        app: web                    # 🔴 must satisfy the selector above
    spec:
      containers:
        - name: web
          image: nginx:1.27.0
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `spec.replicas` ⭐ | Optional (default 1) | Desired number of pods | Integer ≥ 0 | `3` |
| `spec.selector` 🔴 | Required | How the RS finds its pods | **Must match** `template.metadata.labels`; immutable | `matchLabels: {app: web}` |
| `spec.template` 🔴 | Required | Pod blueprint | Labels must satisfy the selector | see YAML |

**Interview Hotspots:** why you use Deployment instead of ReplicaSet directly (rolling updates/rollbacks); the immutable selector rule; "selector does not match template labels" error.

---

### 3. Deployment

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| Deployment | Declarative updates for ReplicaSets/Pods with rollouts & rollbacks | Stateless apps (web servers, APIs) — the default workload | `apps/v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: apps/v1                       # 🔴
kind: Deployment                          # 🔴
metadata:
  name: web-deploy
  namespace: production
  labels:
    app: web
  annotations:
    kubernetes.io/change-cause: "Deploy nginx 1.27.0"  # shows in rollout history
spec:
  replicas: 3                             # ⭐
  revisionHistoryLimit: 5                 # keep 5 old ReplicaSets for rollback
  selector:                               # 🔴 immutable, must match template
    matchLabels:
      app: web
  strategy:                               # ⭐ how updates roll out
    type: RollingUpdate                   # RollingUpdate (default) | Recreate
    rollingUpdate:
      maxUnavailable: 1                    # how many pods can be down during update
      maxSurge: 1                          # how many extra pods allowed during update
  template:
    metadata:
      labels:
        app: web                          # 🔴 must match spec.selector
    spec:
      containers:
        - name: web
          image: nginx:1.27.0             # 🔴 pin the version
          ports:
            - containerPort: 80
          resources:
            requests: {cpu: "100m", memory: "128Mi"}
            limits:   {cpu: "500m", memory: "256Mi"}
          readinessProbe:
            httpGet: {path: /, port: 80}
            initialDelaySeconds: 5
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `spec.replicas` ⭐ | Optional (default 1) | Desired pod count | Overridden by HPA if one targets it | `3` |
| `spec.selector` 🔴 | Required | Selects managed pods | **Immutable** after creation; must match template labels | `matchLabels` |
| `spec.strategy` ⭐ | Optional | Update strategy | `RollingUpdate` = zero-downtime; `Recreate` = kill all then recreate | see YAML |
| `maxUnavailable` / `maxSurge` ⭐ | Optional | Rollout tuning | Can be counts or percentages; both 0 is invalid | `1`, `25%` |
| `revisionHistoryLimit` | Optional (default 10) | Old ReplicaSets kept | Needed for `kubectl rollout undo` | `5` |
| `spec.template` 🔴 | Required | Pod spec to deploy | Any change here triggers a new rollout | see YAML |

**Rollout commands:**

```bash
kubectl rollout status deployment/web-deploy
kubectl rollout history deployment/web-deploy
kubectl rollout undo deployment/web-deploy --to-revision=2
kubectl set image deployment/web-deploy web=nginx:1.27.1
```

**Interview Hotspots:** RollingUpdate vs Recreate; what `maxSurge`/`maxUnavailable` mean; how rollback works; why changing the selector is forbidden; relationship Deployment → ReplicaSet → Pod.

---

### 4. StatefulSet

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| StatefulSet | Manages stateful apps with stable identities & storage | Databases, Kafka, Zookeeper, anything needing stable network IDs/disks | `apps/v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: apps/v1                       # 🔴
kind: StatefulSet                         # 🔴
metadata:
  name: db
spec:
  serviceName: db-headless                # 🔴 must point to a headless Service
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: postgres:16.3
          ports:
            - containerPort: 5432
              name: pg
          volumeMounts:
            - name: data                  # matches volumeClaimTemplates name
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:                   # ⭐ each pod gets its OWN PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `spec.serviceName` 🔴 | Required | Governing **headless** Service for stable DNS | Service must exist with `clusterIP: None` | `db-headless` |
| `volumeClaimTemplates` ⭐ | Optional | PVC template — one PVC per pod | PVCs are **not deleted** when StatefulSet is deleted | see YAML |
| Pod identity | Auto | Pods named `<name>-0`, `<name>-1`… | Ordered, stable, predictable | `db-0` |

**Interview Hotspots:** StatefulSet vs Deployment; why a headless Service is required; stable pod names & DNS (`db-0.db-headless`); ordered scaling; PVCs persist after deletion.

---

### 5. DaemonSet

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| DaemonSet | Runs exactly one pod copy on every (matching) node | Log collectors (Fluentd), node monitoring, CNI agents | `apps/v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: apps/v1                       # 🔴
kind: DaemonSet                           # 🔴
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      tolerations:                         # ⭐ run on control-plane nodes too
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: fluentd
          image: fluent/fluentd:v1.17
          resources:
            limits: {memory: "200Mi"}
            requests: {cpu: "100m", memory: "200Mi"}
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `spec.selector` 🔴 | Required | Selects DaemonSet pods | Must match template labels; no `replicas` field exists | `matchLabels` |
| `tolerations` ⭐ | Optional | Allows running on tainted (e.g. control-plane) nodes | Common gotcha if agents "skip" master nodes | `Exists` |
| `hostPath` volume | Optional | Mounts a node directory | Security-sensitive; restrict in prod | `/var/log` |

**Interview Hotspots:** no `replicas` field (one per node); how to run on control-plane nodes (tolerations); use cases vs Deployment.

---

### 6. Job

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| Job | Runs pods to **successful completion** then stops | Batch processing, migrations, one-off tasks | `batch/v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: batch/v1                       # 🔴
kind: Job                                  # 🔴
metadata:
  name: db-migration
spec:
  completions: 1                           # how many successful pods needed
  parallelism: 1                           # how many run at once
  backoffLimit: 4                          # ⭐ retries before marking Failed
  activeDeadlineSeconds: 600               # hard timeout for the whole job
  ttlSecondsAfterFinished: 3600            # auto-clean 1h after finishing
  template:
    spec:
      restartPolicy: OnFailure             # 🔴 must be OnFailure or Never (not Always)
      containers:
        - name: migrate
          image: myapp/migrator:1.4.2
          command: ["python", "migrate.py"]
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `completions` | Optional (default 1) | Successful pods required | Integer ≥ 1 | `1` |
| `parallelism` | Optional (default 1) | Concurrent pods | Integer ≥ 0 | `5` |
| `backoffLimit` ⭐ | Optional (default 6) | Retry count before failing | Exceeding it marks Job `Failed` | `4` |
| `restartPolicy` 🔴 | Required | Pod restart behavior | **Must be** `OnFailure` or `Never` | `OnFailure` |
| `ttlSecondsAfterFinished` | Optional | Auto-delete finished job | Great for cleanup hygiene | `3600` |

**Interview Hotspots:** why `restartPolicy: Always` is invalid for Jobs; `backoffLimit` vs `activeDeadlineSeconds`; `completions` vs `parallelism`.

---

### 7. CronJob

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| CronJob | Creates Jobs on a time schedule | Backups, reports, periodic cleanup | `batch/v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: batch/v1                       # 🔴 (batch/v1beta1 removed in 1.25)
kind: CronJob                              # 🔴
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"                    # 🔴 cron format → 02:00 daily
  timeZone: "UTC"                          # supported from v1.27 stable
  concurrencyPolicy: Forbid                # Allow | Forbid | Replace
  startingDeadlineSeconds: 120
  successfulJobsHistoryLimit: 3            # keep last 3 successful jobs
  failedJobsHistoryLimit: 1
  jobTemplate:                             # 🔴 a Job spec
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: myapp/backup:2.1.0
              args: ["--target", "s3://bucket/backups"]
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `schedule` 🔴 | Required | Cron expression | 5 fields: min hour day month weekday | `0 2 * * *` |
| `concurrencyPolicy` ⭐ | Optional | Overlap handling | `Forbid` skips, `Replace` kills old, `Allow` overlaps | `Forbid` |
| `jobTemplate` 🔴 | Required | The Job to create | Same rules as Job (`restartPolicy` etc.) | see YAML |
| `timeZone` | Optional | IANA timezone | Stable since v1.27; older clusters use controller TZ | `UTC` |

**Interview Hotspots:** cron syntax; `concurrencyPolicy` options; `batch/v1beta1` removal in **v1.25**; history limits.

---

## 🌐 Networking

### 8. Service

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| Service | Stable network endpoint + load balancing for a set of pods | Exposing apps internally or externally | `v1` | Namespaced |

#### Service Types Comparison ⭐ (very common interview table)

| Type | What it does | Reachable from | Use Case |
|---|---|---|---|
| `ClusterIP` (default) | Internal virtual IP | Inside cluster only | Service-to-service |
| `NodePort` | Opens a port (30000–32767) on every node | Outside via `NodeIP:NodePort` | Quick external access / dev |
| `LoadBalancer` | Provisions a cloud LB | Public internet | Production external apps (cloud) |
| `ExternalName` | DNS CNAME alias | DNS resolution | Mapping to external service |
| Headless (`clusterIP: None`) | No virtual IP; returns pod IPs directly | Inside cluster | StatefulSets, custom discovery |

#### Production-Grade YAML

```yaml
apiVersion: v1                             # 🔴
kind: Service                              # 🔴
metadata:
  name: web-svc
spec:
  type: ClusterIP                          # ⭐ see comparison table
  selector:                                # 🔴 routes to pods with these labels
    app: web
  ports:
    - name: http                           # required if >1 port
      protocol: TCP                        # TCP (default) | UDP | SCTP
      port: 80                             # 🔴 the Service's port
      targetPort: 8080                     # pod's container port (name or number)
  sessionAffinity: None                    # None | ClientIP
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `spec.selector` 🔴 | Required (except ExternalName/manual) | Pod labels to route to | No selector = no automatic endpoints | `app: web` |
| `spec.ports[].port` 🔴 | Required | Port the Service listens on | Distinct from `targetPort` | `80` |
| `spec.ports[].targetPort` | Optional (defaults to `port`) | Container port to forward to | Can be a named port | `8080` |
| `spec.type` ⭐ | Optional (default ClusterIP) | Exposure type | See comparison table | `LoadBalancer` |
| `clusterIP: None` | Optional | Makes Service headless | Required for StatefulSet DNS | `None` |

**Interview Hotspots:** the 4 service types + headless; `port` vs `targetPort` vs `nodePort`; how a Service finds pods (label selector → Endpoints); why a Service has no endpoints (selector mismatch).

---

### 9. Ingress

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| Ingress | HTTP/HTTPS routing (host/path) into the cluster | Single entry point, TLS termination, virtual hosting | `networking.k8s.io/v1` | Namespaced |

> Requires an **Ingress Controller** (nginx, Traefik, AWS ALB, etc.) running in the cluster. The Ingress object alone does nothing.

#### Production-Grade YAML

```yaml
apiVersion: networking.k8s.io/v1           # 🔴 (extensions/v1beta1 removed in 1.22)
kind: Ingress                              # 🔴
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx                  # ⭐ which controller handles this
  tls:
    - hosts:
        - app.example.com
      secretName: web-tls                  # TLS cert stored in a Secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /                        # 🔴
            pathType: Prefix               # 🔴 Prefix | Exact | ImplementationSpecific
            backend:
              service:
                name: web-svc              # 🔴 target Service
                port:
                  number: 80
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `ingressClassName` ⭐ | Optional (but recommended) | Selects the Ingress Controller | Must match an installed IngressClass | `nginx` |
| `rules[].host` | Optional | Hostname matching | Omit for catch-all | `app.example.com` |
| `pathType` 🔴 | Required | Path matching mode | `Prefix` is most common | `Prefix` |
| `backend.service` 🔴 | Required | Target Service + port | Service must exist in same namespace | `web-svc:80` |
| `tls` | Optional | TLS termination config | `secretName` must hold a TLS-type Secret | see YAML |

**Interview Hotspots:** Ingress vs Service (L7 vs L4); needs a controller; `pathType` values; API moved to `networking.k8s.io/v1` in **v1.19**, old APIs removed in **v1.22**.

---

### 10. NetworkPolicy

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| NetworkPolicy | Firewall rules for pod-to-pod traffic | Zero-trust networking, isolating namespaces/tiers | `networking.k8s.io/v1` | Namespaced |

> Requires a CNI that enforces policies (Calico, Cilium, Weave). On a non-enforcing CNI it is silently ignored.

#### Production-Grade YAML

```yaml
apiVersion: networking.k8s.io/v1           # 🔴
kind: NetworkPolicy                        # 🔴
metadata:
  name: api-allow-frontend
  namespace: production
spec:
  podSelector:                             # 🔴 which pods this applies to
    matchLabels:
      app: api
  policyTypes:                             # ⭐ Ingress and/or Egress
    - Ingress
  ingress:
    - from:
        - podSelector:                      # only pods labeled tier=frontend
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 8080
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `podSelector` 🔴 | Required | Pods the policy targets | Empty `{}` = all pods in namespace | `app: api` |
| `policyTypes` ⭐ | Optional | `Ingress`, `Egress`, or both | If omitted, inferred from rules present | `[Ingress]` |
| `ingress.from` | Optional | Allowed sources | Empty `from` = allow all; absent = deny all | podSelector |

> **Default behavior:** With no NetworkPolicy, all traffic is allowed. The moment a pod is selected by **any** policy, it becomes "default-deny" for that direction.

**Interview Hotspots:** default allow-all vs default-deny once selected; needs supporting CNI; `podSelector: {}` meaning; difference between empty `from` and missing `from`.

---

## ⚙️ Configuration & Storage

### 11. ConfigMap

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| ConfigMap | Stores **non-sensitive** config as key/value | App settings, config files, env vars | `v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: v1                             # 🔴
kind: ConfigMap                            # 🔴
metadata:
  name: app-config
data:                                      # ⭐ UTF-8 string key/values
  LOG_LEVEL: "info"
  app.properties: |                        # multi-line file content
    server.port=8080
    cache.enabled=true
immutable: true                            # optional, perf + safety (v1.21+)
binaryData: {}                             # base64 for binary blobs
```

**Consume in a pod:**

```yaml
envFrom:
  - configMapRef:
      name: app-config        # inject ALL keys as env vars
# or as a mounted file:
volumes:
  - name: cfg
    configMap:
      name: app-config
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `data` ⭐ | Optional | String key/value pairs | UTF-8 only; ≤1 MiB total per ConfigMap | `LOG_LEVEL: info` |
| `binaryData` | Optional | Base64 binary values | Keys must not clash with `data` | `{}` |
| `immutable` | Optional | Locks the object | Faster + prevents accidental edits; must recreate to change | `true` |

> Note: ConfigMap has **no `spec`** — it uses `data`. Common beginner mistake.

**Interview Hotspots:** ConfigMap vs Secret; no `spec` field; immutable ConfigMaps (v1.21+); env vars don't auto-refresh, mounted files do (eventually).

---

### 12. Secret

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| Secret | Stores **sensitive** data (base64-encoded) | Passwords, tokens, TLS certs, registry creds | `v1` | Namespaced |

> ⚠️ Base64 is **encoding, not encryption**. Enable **encryption at rest** + RBAC for real security.

#### Production-Grade YAML

```yaml
apiVersion: v1                             # 🔴
kind: Secret                               # 🔴
metadata:
  name: db-secret
type: Opaque                               # ⭐ Opaque | kubernetes.io/tls | dockerconfigjson | ...
data:                                      # values MUST be base64-encoded
  username: YWRtaW4=                       # echo -n 'admin' | base64
stringData:                                # ⭐ plain text; auto-encoded by k8s
  password: "S3cr3tP@ss"
```

#### Common Secret Types

| `type` | Use Case | Required keys |
|---|---|---|
| `Opaque` | Generic key/value | any |
| `kubernetes.io/tls` | TLS certs (Ingress) | `tls.crt`, `tls.key` |
| `kubernetes.io/dockerconfigjson` | Private registry auth | `.dockerconfigjson` |
| `kubernetes.io/service-account-token` | SA token | managed |

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `type` ⭐ | Optional (default Opaque) | Secret category | Typed secrets require specific keys | `Opaque` |
| `data` | Optional | Base64-encoded values | Invalid base64 = create error | `YWRtaW4=` |
| `stringData` ⭐ | Optional | Plain text (write-only) | k8s encodes it; merged into `data` | `password: x` |

**Interview Hotspots:** base64 ≠ encryption; `data` vs `stringData`; encryption at rest; Secret types; how pods consume Secrets (env vs volume).

---

### 13. PersistentVolume (PV)

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| PersistentVolume | A piece of cluster storage (admin-provisioned or dynamic) | Backing storage for PVCs | `v1` | **Cluster-scoped** |

#### Production-Grade YAML

```yaml
apiVersion: v1                             # 🔴
kind: PersistentVolume                     # 🔴
metadata:
  name: pv-data                            # no namespace (cluster-scoped)
spec:
  capacity:
    storage: 10Gi                          # 🔴
  accessModes:
    - ReadWriteOnce                        # ⭐ RWO | ROX | RWX | RWOP
  persistentVolumeReclaimPolicy: Retain    # Retain | Delete | (Recycle deprecated)
  storageClassName: standard
  nfs:                                     # example backend
    server: 10.0.0.10
    path: /exports/data
```

#### Access Modes ⭐

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Mounted read-write by a single **node** |
| `ReadOnlyMany` | ROX | Read-only by many nodes |
| `ReadWriteMany` | RWX | Read-write by many nodes |
| `ReadWriteOncePod` | RWOP | Read-write by a single **pod** (v1.22+) |

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `capacity.storage` 🔴 | Required | Size of the volume | Uses Ki/Mi/Gi units | `10Gi` |
| `accessModes` 🔴 | Required | How it can be mounted | Must be compatible with the PVC | `ReadWriteOnce` |
| `persistentVolumeReclaimPolicy` ⭐ | Optional | What happens after PVC release | `Retain` keeps data; `Delete` removes it | `Retain` |

**Interview Hotspots:** PV vs PVC; access modes (esp. RWO vs RWX); reclaim policies; cluster-scoped (no namespace); static vs dynamic provisioning.

---

### 14. PersistentVolumeClaim (PVC)

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| PersistentVolumeClaim | A user's request for storage; binds to a PV | Attaching durable storage to pods | `v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: v1                             # 🔴
kind: PersistentVolumeClaim                # 🔴
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce                        # 🔴 must be satisfiable by a PV
  resources:
    requests:
      storage: 5Gi                         # 🔴
  storageClassName: standard               # "" = no dynamic provisioning
```

**Use in a pod:**

```yaml
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `accessModes` 🔴 | Required | Requested access mode | Must match an available PV | `ReadWriteOnce` |
| `resources.requests.storage` 🔴 | Required | Requested size | A PV ≥ this size must exist or be provisioned | `5Gi` |
| `storageClassName` ⭐ | Optional | Dynamic provisioning class | `""` disables dynamic provisioning | `standard` |

**Interview Hotspots:** binding lifecycle (Pending → Bound); how StorageClass enables dynamic provisioning; why a PVC stays Pending; PVC is namespaced but PV is not.

---

## 📈 Scaling

### 15. HorizontalPodAutoscaler (HPA)

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| HPA | Auto-scales replicas based on metrics | Handle variable traffic on Deployments | `autoscaling/v2` | Namespaced |

> Requires the **metrics-server** (for CPU/memory) installed in the cluster.

#### Production-Grade YAML

```yaml
apiVersion: autoscaling/v2                  # 🔴 v2 = multi-metric (v1 = CPU only)
kind: HorizontalPodAutoscaler               # 🔴
metadata:
  name: web-hpa
spec:
  scaleTargetRef:                           # 🔴 what to scale
    apiVersion: apps/v1
    kind: Deployment
    name: web-deploy
  minReplicas: 2                            # ⭐
  maxReplicas: 10                           # 🔴
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70            # scale to keep avg CPU ~70%
  behavior:                                 # fine-tune scale up/down (v1.23+)
    scaleDown:
      stabilizationWindowSeconds: 300
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `scaleTargetRef` 🔴 | Required | The workload to scale | Target must expose `scale` subresource | Deployment `web-deploy` |
| `minReplicas` ⭐ | Optional (default 1) | Lower bound | ≥1 (0 needs feature gate) | `2` |
| `maxReplicas` 🔴 | Required | Upper bound | Must be ≥ `minReplicas` | `10` |
| `metrics` ⭐ | Optional | Scaling signals | Needs metrics-server / custom metrics API | CPU 70% |

> ⚠️ Don't set a static `spec.replicas` on a Deployment managed by an HPA — they fight each other.

**Interview Hotspots:** `autoscaling/v2` vs `v1`; needs metrics-server; HPA vs VPA vs Cluster Autoscaler; the replicas conflict gotcha.

---

## 🗂 Organization & RBAC

### 16. Namespace

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| Namespace | Virtual cluster partition for isolation & quotas | Separate envs/teams (dev/stage/prod) | `v1` | **Cluster-scoped** |

#### Production-Grade YAML

```yaml
apiVersion: v1                             # 🔴
kind: Namespace                            # 🔴
metadata:
  name: production                         # 🔴 DNS-1123 label, ≤63 chars
  labels:
    team: platform
    environment: prod
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `metadata.name` 🔴 | Required | Namespace name | DNS-1123 **label** (≤63 chars, no dots) | `production` |
| (no `spec` needed) | — | Namespace has minimal spec | `kube-system`, `default`, `kube-public` are reserved | — |

**Interview Hotspots:** what is/ isn't namespaced (PV, Node, ClusterRole are not); resource isolation; default namespaces; deleting a namespace deletes everything in it.

---

### 17. ServiceAccount

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| ServiceAccount | Identity for processes running **in pods** | Pod → API server auth, workload identity | `v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: v1                             # 🔴
kind: ServiceAccount                       # 🔴
metadata:
  name: app-sa
  namespace: production
automountServiceAccountToken: false        # ⭐ disable if pod doesn't call the API
```

**Use in a pod:** `spec.serviceAccountName: app-sa`

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `metadata.name` 🔴 | Required | SA identity | One `default` SA per namespace auto-created | `app-sa` |
| `automountServiceAccountToken` ⭐ | Optional | Auto-mount API token | Set `false` for least privilege | `false` |

**Interview Hotspots:** ServiceAccount (machine) vs User (human, external); the default SA; token automounting; pairing SA with RoleBinding for RBAC.

---

### 18. Role

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| Role | Namespaced set of permissions | Grant access within one namespace | `rbac.authorization.k8s.io/v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1   # 🔴
kind: Role                                 # 🔴
metadata:
  name: pod-reader
  namespace: production
rules:                                     # 🔴 list of permission rules
  - apiGroups: [""]                        # "" = core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]        # ⭐ get/list/watch/create/update/delete
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `rules` 🔴 | Required | Permission rules | RBAC is **additive only** — no deny rules | see YAML |
| `rules[].apiGroups` 🔴 | Required | API groups | `""` for core (pods, services) | `["apps"]` |
| `rules[].resources` 🔴 | Required | Resource types | Plural, lowercase | `["pods"]` |
| `rules[].verbs` 🔴 | Required | Allowed actions | get/list/watch/create/update/patch/delete | `["get","list"]` |

**Interview Hotspots:** Role vs ClusterRole; RBAC is additive (no deny); core group is `""`; verbs list.

---

### 19. RoleBinding

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| RoleBinding | Grants a Role to subjects in a namespace | Bind users/groups/SAs to a Role | `rbac.authorization.k8s.io/v1` | Namespaced |

#### Production-Grade YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1   # 🔴
kind: RoleBinding                          # 🔴
metadata:
  name: read-pods
  namespace: production
subjects:                                  # 🔴 who gets access
  - kind: ServiceAccount
    name: app-sa
    namespace: production
roleRef:                                   # 🔴 which Role (immutable!)
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `subjects` 🔴 | Required | Users/Groups/ServiceAccounts | `kind` must be one of those three | ServiceAccount |
| `roleRef` 🔴 | Required | The Role/ClusterRole granted | **Immutable** after creation | `Role/pod-reader` |

> A RoleBinding can reference a **ClusterRole** to reuse it within a single namespace.

**Interview Hotspots:** `roleRef` is immutable; RoleBinding can bind a ClusterRole (namespace-scoped reuse); subject kinds.

---

### 20. ClusterRole

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| ClusterRole | Cluster-wide permissions (or reusable across namespaces) | Cluster admins, nodes, non-namespaced resources | `rbac.authorization.k8s.io/v1` | **Cluster-scoped** |

#### Production-Grade YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1   # 🔴
kind: ClusterRole                          # 🔴
metadata:
  name: node-reader                        # no namespace
rules:
  - apiGroups: [""]
    resources: ["nodes"]                   # cluster-scoped resource
    verbs: ["get", "list", "watch"]
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `rules` 🔴 | Required | Cluster-wide permission rules | Can grant access to non-namespaced resources | nodes/get |
| (no namespace) | — | ClusterRole is cluster-scoped | Reusable by many RoleBindings | — |

**Interview Hotspots:** ClusterRole vs Role; used for nodes/PVs/namespaces; can be bound cluster-wide or per-namespace.

---

### 21. ClusterRoleBinding

#### Basic Overview

| Object | Purpose | Common Use Case | API Version | Scope |
|---|---|---|---|---|
| ClusterRoleBinding | Grants a ClusterRole cluster-wide | Cluster admin access, system components | `rbac.authorization.k8s.io/v1` | **Cluster-scoped** |

#### Production-Grade YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1   # 🔴
kind: ClusterRoleBinding                   # 🔴
metadata:
  name: read-nodes-global
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
roleRef:                                   # 🔴 immutable
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

#### Field-by-Field Explanation

| YAML Field | Required/Optional | Description | Important Rules | Example |
|---|---|---|---|---|
| `subjects` 🔴 | Required | Who gets cluster-wide access | SAs need explicit `namespace` | ServiceAccount |
| `roleRef` 🔴 | Required | ClusterRole to bind | Immutable; `kind` must be ClusterRole | `node-reader` |

**Interview Hotspots:** scope of access (entire cluster, all namespaces); danger of binding `cluster-admin`; immutability of `roleRef`.

---

## 🔧 Cross-Cutting Reference

### Naming Conventions (All Objects)

| Item | Naming Rules | Allowed Characters | Best Practice | Example |
|---|---|---|---|---|
| **Object names** | RFC 1123 subdomain, ≤253 chars, lowercase | `a-z 0-9 - .` (must start/end alphanumeric) | Use kebab-case, include app + role | `web-frontend-prod` |
| **Namespace** | RFC 1123 **label**, ≤63 chars, no dots | `a-z 0-9 -` | Map to env/team | `payments-prod` |
| **Label keys** | Optional DNS prefix `/` + ≤63-char name | `a-z A-Z 0-9 - _ .` | Use a domain prefix for custom labels | `app.kubernetes.io/name` |
| **Label values** | ≤63 chars, alphanumeric ends | `a-z A-Z 0-9 - _ .` | Stable, low-cardinality values | `frontend` |
| **Annotation keys** | Same as label keys; values not constrained | any UTF-8 in value | Prefix with owning domain | `prometheus.io/scrape` |
| **Container names** | RFC 1123 label, ≤63 chars | `a-z 0-9 -` | Describe the process | `api-server` |
| **Service names** | RFC 1123 label (becomes DNS) | `a-z 0-9 -` | Keep short — used in DNS | `web-svc` |
| **Image names** | `registry/repo:tag` or `@digest` | per OCI spec | **Pin a version or digest**, never `latest` | `ghcr.io/org/api:1.4.2` |

**Recommended standard labels** (from Kubernetes docs):

```yaml
labels:
  app.kubernetes.io/name: web
  app.kubernetes.io/instance: web-prod
  app.kubernetes.io/version: "1.27.0"
  app.kubernetes.io/component: frontend
  app.kubernetes.io/part-of: shop
  app.kubernetes.io/managed-by: helm
```

---

### Indentation & YAML Syntax Rules

| Rule | Correct Example | Wrong Example | Why It Fails |
|---|---|---|---|
| **Spaces only, never tabs** | `··name: web` (spaces) | `→name: web` (tab) | YAML forbids tabs for indentation → parse error |
| **2-space indentation** | nested keys indented by 2 | mixing 2 and 4 randomly | Inconsistent indent breaks the mapping tree |
| **List items use `-`** | `containers:`<br>`- name: web` | `containers:`<br>`name: web` | Without `-` it's a map, not a list → "expected list" |
| **Space after colon** | `name: web` | `name:web` | No space → key not parsed as mapping |
| **Quote special strings** | `version: "1.0"` | `version: 1.0` | `1.0` becomes a float; `"1.0"` stays a string |
| **Quote booleans-as-strings** | `value: "true"` | `value: true` | env values must be strings; bare `true` is a boolean |
| **Quote `yes/no/on/off`** | `region: "no"` | `region: no` | YAML 1.1 reads `no` as boolean `false` (Norway problem) |
| **List vs object** | `args: ["a","b"]` | `args: a, b` | Comma string ≠ a YAML list |
| **Case sensitivity** | `kind: Deployment` | `kind: deployment` | Kinds are case-sensitive → "no matches for kind" |
| **Consistent list indent** | all items same indent | items at different indents | Breaks sequence structure |
| **One doc or `---` separator** | `---` between manifests | two docs no separator | Second manifest ignored / parse error |

> 🧠 **The "Norway Problem":** `country: NO` parses as `country: false`. Always quote ambiguous scalars: `"NO"`, `"true"`, `"1.0"`, `"08"` (octal-looking).

---

### Important Validation Rules

| Rule | Why Important | Common Error | Fix |
|---|---|---|---|
| **Selector must match template labels** | Controller can't find/own its pods | `selector does not match template labels` | Make `spec.selector.matchLabels` ⊆ `template.metadata.labels` |
| **Ports format** | Wrong shape rejects manifest | `cannot unmarshal number into ports` | `ports:` is a list of objects with `containerPort` |
| **imagePullPolicy values** | Typos rejected by API | `unsupported value` | Use exactly `Always`, `IfNotPresent`, or `Never` |
| **Immutable fields** | Some fields can't change | `field is immutable` | Recreate object (e.g. Deployment `selector`, `roleRef`) |
| **Resource limits ≥ requests** | Scheduler/runtime sanity | `limits.cpu < requests.cpu` | Ensure every limit ≥ its request |
| **Required metadata.name** | Identity is mandatory | `metadata.name: Required value` | Always set a valid name |
| **API version compatibility** | APIs get promoted/removed | `no matches for kind "Ingress" in version "extensions/v1beta1"` | Use current API (`networking.k8s.io/v1`) |
| **Unique names** | Name unique per (ns + kind) | `AlreadyExists` | Rename or apply to different namespace |
| **Namespace must exist** | Objects need a home | `namespaces "x" not found` | Create namespace first |
| **restartPolicy for Jobs** | Jobs can't use `Always` | `unsupported value: "Always"` | Use `OnFailure` or `Never` |

---

### Common kubectl Validation Commands

| Command | Purpose | Example |
|---|---|---|
| `kubectl apply -f` | Create/update from manifest (declarative) | `kubectl apply -f deploy.yaml` |
| `kubectl apply --dry-run=server -f` | Validate against the API **without** applying | `kubectl apply --dry-run=server -f deploy.yaml` |
| `kubectl apply --dry-run=client -f` | Local syntax/schema check only | `kubectl apply --dry-run=client -f deploy.yaml` |
| `kubectl diff -f` | Show what would change vs live cluster | `kubectl diff -f deploy.yaml` |
| `kubectl explain` | Inline schema/docs for any field | `kubectl explain deployment.spec.strategy` |
| `kubectl explain --recursive` | Full field tree of a resource | `kubectl explain pod.spec --recursive` |
| `kubectl describe` | Detailed object state + events | `kubectl describe pod web-pod` |
| `kubectl get -o yaml` | Inspect live object as YAML | `kubectl get deploy web -o yaml` |
| `kubectl api-resources` | List resource kinds + scope + apiVersion | `kubectl api-resources` |
| `kubectl api-versions` | List served API group/versions | `kubectl api-versions` |
| `kubectl logs` | Container logs (debugging) | `kubectl logs web-pod -c web --previous` |
| `kubectl get events` | Cluster events, newest issues | `kubectl get events --sort-by=.lastTimestamp` |

> 💡 **Pre-merge tip:** `kubectl apply --dry-run=server -f` is the strongest local validation — it runs admission controllers without persisting.

---

### Common Errors & Troubleshooting

| Error Message | Root Cause | Fix | Prevention Tip |
|---|---|---|---|
| `ImagePullBackOff` / `ErrImagePull` | Bad image name/tag, or missing registry creds | Fix image, add `imagePullSecrets` | Pin tested tags; pre-pull/mirror images |
| `CrashLoopBackOff` | Container exits repeatedly (bad cmd, missing config, failing probe) | Check `kubectl logs --previous`; fix app/probe | Sane probes; validate config before deploy |
| `CreateContainerConfigError` | Referenced ConfigMap/Secret/key missing | Create the ConfigMap/Secret or fix the key name | Apply config objects before workloads |
| `Pod stuck in Pending` | No node fits (resources, nodeSelector, taints, unbound PVC) | Check `kubectl describe pod` events | Right-size requests; verify selectors/PVCs |
| `0/3 nodes available: insufficient cpu` | Requests exceed free capacity | Lower requests or add nodes | Capacity planning; HPA + cluster autoscaler |
| `selector does not match template labels` | Deployment selector ≠ pod template labels | Align labels; selector ⊆ template labels | Copy labels into both blocks |
| `Service has no endpoints` | Service selector matches no ready pods | Fix selector or pod labels; check readiness | Keep selector == pod labels |
| `no matches for kind "X" in version "Y"` | Removed/incorrect apiVersion | Use current API version | Check `kubectl api-versions`; lint manifests |
| `field is immutable` | Editing an immutable field | Delete + recreate the object | Know immutable fields (selector, roleRef) |
| `error validating data: unknown field` | Typo / wrong indentation level | Fix field name/indent; use `kubectl explain` | Schema-validate in CI (`--dry-run=server`) |
| `forbidden: User cannot ... (RBAC)` | Missing Role/Binding permissions | Grant needed verbs via Role + RoleBinding | Least-privilege RBAC, test with `auth can-i` |
| `OOMKilled` (exit 137) | Container exceeded memory limit | Raise memory limit or fix the leak | Load test; set realistic limits |
| `Unbound PersistentVolumeClaim` | No matching PV / no StorageClass | Provide PV or default StorageClass | Use dynamic provisioning |

**Fast triage flow:**

```bash
kubectl get pods -o wide
kubectl describe pod <pod>          # read the Events section first
kubectl logs <pod> --previous       # last crashed container
kubectl get events --sort-by=.lastTimestamp
kubectl auth can-i <verb> <resource>  # RBAC checks
```

---

### Best Practices

| Practice | Why Recommended | Example |
|---|---|---|
| **Standard label strategy** | Consistent selection, tooling, cost attribution | Use `app.kubernetes.io/*` labels everywhere |
| **One namespace per env/team** | Isolation, quotas, RBAC boundaries | `dev`, `staging`, `prod` namespaces |
| **Always set requests & limits** | Stable scheduling, no noisy neighbors | `requests: {cpu:100m,memory:128Mi}` |
| **Liveness + readiness probes** | Self-healing + safe rollouts | httpGet `/healthz` and `/ready` |
| **Immutable infrastructure** | Reproducible, no in-place drift | Rebuild image, never `kubectl edit` pods |
| **Externalize config** | 12-factor; same image across envs | ConfigMap/Secret + `envFrom` |
| **Never bake secrets into images** | Leak prevention | Mount Secrets; enable encryption at rest |
| **Pin image versions/digests** | Reproducible, no surprise upgrades | `api:1.4.2` or `api@sha256:...` |
| **GitOps repo structure** | Auditable, declarative, reviewable | `base/` + `overlays/{dev,prod}` (Kustomize) |
| **Resource quotas + LimitRanges** | Prevent namespace resource abuse | `ResourceQuota` per namespace |
| **PodDisruptionBudget** | Maintain availability during drains | `minAvailable: 2` |
| **Least-privilege RBAC** | Reduce blast radius | Namespaced Role over `cluster-admin` |
| **Validate in CI** | Catch errors before cluster | `kubeconform` / `--dry-run=server` |

**Suggested GitOps layout:**

```text
k8s/
├── base/                 # shared manifests + kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/              # env-specific patches
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

---

### Interview Quick-Fire

| Question | Crisp Answer |
|---|---|
| Pod vs Deployment? | Pod = single unit, no self-heal. Deployment manages ReplicaSets → rolling updates, rollbacks, self-healing. |
| ReplicaSet vs Deployment? | Deployment wraps ReplicaSets to add versioned rollouts/rollbacks. You manage Deployments, not RS directly. |
| Deployment vs StatefulSet? | Deployment = interchangeable stateless pods. StatefulSet = stable identity, ordered, per-pod storage. |
| StatefulSet vs DaemonSet? | StatefulSet = N ordered stateful pods. DaemonSet = one pod per node. |
| Job vs CronJob? | Job runs to completion once. CronJob schedules Jobs on a cron expression. |
| ConfigMap vs Secret? | Both key/value. Secret is base64 + (optionally) encrypted at rest + RBAC-restricted; for sensitive data. |
| Service types? | ClusterIP (internal), NodePort (node port), LoadBalancer (cloud LB), ExternalName (DNS alias), Headless (no clusterIP). |
| `port` vs `targetPort` vs `nodePort`? | port = Service port; targetPort = container port; nodePort = node-level external port (NodePort type). |
| Service vs Ingress? | Service = L4 stable endpoint. Ingress = L7 host/path HTTP routing + TLS (needs a controller). |
| PV vs PVC? | PV = actual storage (cluster-scoped). PVC = namespaced request that binds to a PV. |
| Liveness vs readiness vs startup probe? | Liveness restarts a hung container; readiness gates traffic; startup protects slow-starting apps. |
| requests vs limits? | requests = guaranteed/used for scheduling; limits = hard cap (CPU throttled, memory → OOMKilled). |
| Role vs ClusterRole? | Role = namespaced perms; ClusterRole = cluster-wide / non-namespaced / reusable. |
| RoleBinding vs ClusterRoleBinding? | RoleBinding grants within a namespace; ClusterRoleBinding grants cluster-wide. |
| What's cluster-scoped? | Nodes, PersistentVolumes, Namespaces, ClusterRole(Binding), StorageClass — no `metadata.namespace`. |
| Why is `:latest` bad? | Non-reproducible, breaks rollbacks, cache ambiguity. Always pin a tag/digest. |
| Default NetworkPolicy behavior? | Allow-all until a policy selects a pod; then default-deny for that direction. |
| RollingUpdate vs Recreate? | RollingUpdate = gradual zero-downtime; Recreate = terminate all then create all (downtime). |
| HPA requirement? | metrics-server (CPU/memory) or custom-metrics API; don't pin static replicas alongside it. |
| Is base64 encryption? | No — it's encoding. Use encryption at rest + RBAC for real Secret protection. |

---

## ✅ Final Checklist Before `kubectl apply`

- [ ] `apiVersion` + `kind` correct and current (`kubectl api-versions`)
- [ ] `metadata.name` valid (RFC 1123) and unique in namespace
- [ ] Target namespace exists
- [ ] `spec.selector` ⊆ `template.metadata.labels`
- [ ] Image pinned to a specific tag/digest (no `:latest`)
- [ ] `requests` and `limits` set; limits ≥ requests
- [ ] Liveness + readiness probes defined
- [ ] Secrets referenced, not hardcoded
- [ ] `kubectl apply --dry-run=server -f` passes
- [ ] `kubectl diff -f` reviewed

---

> 📌 **Tip:** Keep this file in your repo as `README.md` or `docs/k8s-cheatsheet.md`. Pair it with `kubectl explain <resource>` for live, version-accurate field docs on your own cluster.

*End of handbook — happy shipping. 🚢*
