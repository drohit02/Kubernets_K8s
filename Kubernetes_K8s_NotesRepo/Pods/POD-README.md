# Kubernetes Pod — Complete Reference

> Smallest deployable unit in Kubernetes. One or more containers sharing the same network and storage.

---

## What is a Pod?

| Feature | Detail |
|---|---|
| **API Version** | `v1` |
| **Kind** | `Pod` |
| **Scope** | Namespaced |
| **Self-healing?** | ❌ Bare pods don't restart. Use a Deployment. |
| **Shared across containers** | Network (same IP), Volumes, `localhost` communication |

---

## Pod Lifecycle

```
Pending ──► Running ──► Succeeded
              │
              ▼
           Failed  ──► (CrashLoopBackOff if restartPolicy=Always)
```

| Phase | Meaning |
|---|---|
| `Pending` | Scheduled, but container not started yet (image pull, node assignment) |
| `Running` | At least one container is running |
| `Succeeded` | All containers exited with code 0 |
| `Failed` | At least one container exited with non-zero code |
| `Unknown` | Node lost contact — status unknown |

---

## YAML Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod               # required — must be DNS-1123 compliant
  namespace: default          # optional, defaults to "default"
  labels:
    app: web                  # used by Services to route traffic
spec:
  containers:
    - name: web               # required — unique per pod
      image: nginx:1.27.0     # required — ALWAYS pin a version, never :latest
      ports:
        - containerPort: 80   # informational only — doesn't open firewall
      env:
        - name: LOG_LEVEL
          value: "info"
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 5
  restartPolicy: Always       # Always | OnFailure | Never
```

---

## Field Reference

### Metadata Fields

| Field | Required | Rule | Example |
|---|---|---|---|
| `metadata.name` | Yes | ≤253 chars, lowercase, RFC 1123 | `web-pod` |
| `metadata.namespace` | No | Defaults to `default` | `production` |
| `metadata.labels` | No | Key/value pairs for selectors | `app: web` |
| `metadata.annotations` | No | Non-selector metadata | `owner: team-a` |

### Spec Fields

| Field | Required | Description | Key Rule |
|---|---|---|---|
| `spec.containers` | Yes | List of containers (min 1) | Must be a list (`-`) |
| `containers[].name` | Yes | Container identifier | Unique per pod |
| `containers[].image` | Yes | Docker image | Pin version, avoid `:latest` |
| `containers[].ports` | No | Port documentation | Informational only |
| `containers[].env` | No | Environment variables | Use `valueFrom` for Secrets |
| `containers[].resources` | No* | CPU/memory bounds | *Best practice to always set |
| `containers[].livenessProbe` | No | Restart trigger | Tune `initialDelaySeconds` |
| `containers[].readinessProbe` | No | Traffic gate | Failing removes pod from Service |
| `containers[].startupProbe` | No | Protects slow-starting apps | Disables liveness until passes |
| `restartPolicy` | No | Restart on exit | `Always`/`OnFailure`/`Never` |
| `nodeSelector` | No | Hard node targeting | Node must have matching label |
| `tolerations` | No | Schedule on tainted nodes | Must match taint key + effect |
| `serviceAccountName` | No | Pod identity for API access | Defaults to `default` SA |

---

## Resources — requests vs limits

```
requests = what Kubernetes schedules on (guaranteed)
limits   = hard cap (CPU throttled, Memory → OOMKilled)
```

| | `requests` | `limits` |
|---|---|---|
| **Used for** | Node scheduling decision | Runtime enforcement |
| **CPU over limit** | — | Throttled (not killed) |
| **Memory over limit** | — | OOMKilled (exit 137) |
| **Rule** | Must be ≤ limits | Must be ≥ requests |

---

## Health Probes

| Probe | Trigger | Action | Use Case |
|---|---|---|---|
| `livenessProbe` | Container is stuck/hung | Restart container | Deadlock recovery |
| `readinessProbe` | App not ready for traffic | Remove from Service endpoints | Startup / overloaded |
| `startupProbe` | App takes long to boot | Delays liveness check | Legacy apps, slow JVM |

**Probe methods:**

| Method | How | Example |
|---|---|---|
| `httpGet` | HTTP GET → expects 2xx/3xx | `path: /healthz, port: 80` |
| `tcpSocket` | TCP connection check | `port: 5432` |
| `exec` | Runs a command, exit 0 = healthy | `command: ["cat", "/ready"]` |

---

## Environment Variables

```yaml
env:
  # Static value
  - name: ENV
    value: "production"

  # From Secret
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password

  # From ConfigMap
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL

# Inject ALL keys from a ConfigMap/Secret
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: db-secret
```

---

## Volumes in a Pod

```
Pod
 ├── Container A ──► volumeMount (mountPath: /data)
 ├── Container B ──► volumeMount (mountPath: /shared)
 └── Volume (shared between both containers)
```

| Volume Type | Description | Persists after pod? |
|---|---|---|
| `emptyDir` | Temp scratch space, shared between containers | ❌ No |
| `hostPath` | Mounts a node directory | ✅ On same node only |
| `configMap` | Mounts ConfigMap as files | ❌ No |
| `secret` | Mounts Secret as files | ❌ No |
| `persistentVolumeClaim` | Attaches a PVC | ✅ Yes |

---

## restartPolicy

| Policy | When it restarts | Use with |
|---|---|---|
| `Always` | Always on exit (default) | Pods, Deployments |
| `OnFailure` | Only on non-zero exit code | Jobs |
| `Never` | Never | Jobs, one-off tasks |

---

## Node Scheduling

### nodeSelector (simple)
```yaml
nodeSelector:
  disktype: ssd     # node must have this label
```

### Tolerations (for tainted nodes)
```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "web"
    effect: "NoSchedule"
```

| Taint Effect | Meaning |
|---|---|
| `NoSchedule` | Pod won't be scheduled unless tolerated |
| `PreferNoSchedule` | Soft — tries to avoid, but not forced |
| `NoExecute` | Evicts existing pods that don't tolerate it |

---

## Multi-Container Patterns

```
Pod
 ├── Main Container      ← the app
 ├── Sidecar Container   ← helper (e.g., log shipper, proxy)
 └── Init Container      ← runs before main starts (setup tasks)
```

| Pattern | Container | Runs | Purpose |
|---|---|---|---|
| Sidecar | `containers[]` | Alongside main | Logging, monitoring, proxy |
| Init | `initContainers[]` | Before main starts | DB migration, config fetch |
| Ephemeral | `ephemeralContainers[]` | On-demand (debug) | Live debugging |

---

## kubectl — Essential Pod Commands

| Command | What it does |
|---|---|
| `kubectl run nginx --image=nginx:1.27` | Create a bare pod quickly |
| `kubectl apply -f pod.yaml` | Create/update pod from YAML |
| `kubectl get pods` | List pods in current namespace |
| `kubectl get pods -o wide` | List with node, IP info |
| `kubectl describe pod <name>` | Full details + events (first debug step) |
| `kubectl logs <pod>` | Container stdout/stderr |
| `kubectl logs <pod> -c <container>` | Logs from specific container |
| `kubectl logs <pod> --previous` | Logs from last crashed container |
| `kubectl exec -it <pod> -- bash` | Shell into running container |
| `kubectl delete pod <name>` | Delete a pod |
| `kubectl top pod <name>` | CPU/memory usage (needs metrics-server) |
| `kubectl get pod <name> -o yaml` | Inspect live YAML |

---

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` | Wrong image name/tag or missing auth | Fix image tag; add `imagePullSecrets` |
| `CrashLoopBackOff` | App exits repeatedly | Check `kubectl logs --previous`; fix app or probes |
| `Pending` (no node) | Insufficient CPU/memory or no matching node | Reduce requests; check `kubectl describe pod` events |
| `CreateContainerConfigError` | Missing ConfigMap/Secret | Create referenced resources first |
| `OOMKilled` (exit 137) | Memory limit exceeded | Increase memory limit or fix memory leak |
| `Error: failed to create containerd task` | Image not pullable | Check registry access and image path |

**Triage flow:**
```bash
kubectl describe pod <name>       # read Events section first
kubectl logs <name> --previous    # last crash output
kubectl get events --sort-by=.lastTimestamp
```

---

## Best Practices

- **Never use `:latest`** — pin a specific version or digest
- **Always set** `resources.requests` and `resources.limits`
- **Add both** `livenessProbe` and `readinessProbe`
- **Inject secrets** via `secretKeyRef` or mounted volume — never hardcode
- **Use labels** — Services and controllers depend on them
- **Bare pods don't self-heal** — use a Deployment for production workloads
- **One app per container** — don't run multiple apps in one container
- **Set `automountServiceAccountToken: false`** if pod doesn't call the Kubernetes API

---

## Labels vs Annotations

| | `labels` | `annotations` |
|---|---|---|
| **Used for selection?** | ✅ Yes (Services, Deployments) | ❌ No |
| **Size limit** | 63 chars per value | Up to 256 KB total |
| **Example** | `app: web`, `env: prod` | `prometheus.io/scrape: "true"` |

---

## Quick Reference — apiVersion + kind

| Object | apiVersion | kind |
|---|---|---|
| Pod | `v1` | `Pod` |
| Service | `v1` | `Service` |
| ConfigMap | `v1` | `ConfigMap` |
| Secret | `v1` | `Secret` |
| Deployment | `apps/v1` | `Deployment` |
| ReplicaSet | `apps/v1` | `ReplicaSet` |

> Run `kubectl api-versions` and `kubectl api-resources` on your cluster for the authoritative list.

---

*Pod is the foundation — understand it deeply and everything else in Kubernetes will make sense.*
