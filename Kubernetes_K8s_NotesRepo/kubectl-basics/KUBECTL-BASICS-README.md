# kubectl Basics — Complete Reference

> kubectl is the command-line tool to communicate with a Kubernetes cluster's API server.
> Every create, get, delete, and debug operation goes through kubectl.

---

## What is kubectl?

| Feature | Detail |
|---|---|
| **Full name** | Kubernetes Control (kube-control) |
| **Talks to** | Kubernetes API Server |
| **Auth via** | kubeconfig file (`~/.kube/config`) |
| **Format** | `kubectl <verb> <resource> [name] [flags]` |
| **Config override** | `--kubeconfig`, `KUBECONFIG` env var, or `--context` |
| **Installed separately?** | Yes — not bundled with the cluster |

---

## How kubectl Works

```
You type a command
       │
       ▼
  kubectl reads ~/.kube/config
       │  (finds cluster URL, user credentials, namespace)
       │
       ▼
  HTTPS request to API Server
       │  (authenticated via cert / token / OIDC)
       │
       ▼
  API Server validates + authorizes (RBAC)
       │
       ▼
  etcd (read/write cluster state)
       │
       ▼
  Response back to kubectl → printed to terminal
```

---

## Command Structure

```
kubectl  <verb>   <resource>   <name>      <flags>
   │        │         │           │            │
   │     get,      pod,        web-pod     -n prod
   │    apply,    deploy,                 -o yaml
   │   delete,   service,                --all
   │    logs,    node ...                -l app=web
   │    exec
   │
always first
```

---

## Core Verbs (What You Can Do)

| Verb | What it does | Example |
|---|---|---|
| `get` | List or show resources | `kubectl get pods` |
| `describe` | Detailed info + events | `kubectl describe pod web` |
| `apply` | Create or update from YAML (declarative) | `kubectl apply -f pod.yaml` |
| `create` | Create from YAML/flags (imperative, fails if exists) | `kubectl create -f pod.yaml` |
| `delete` | Remove a resource | `kubectl delete pod web` |
| `edit` | Open live resource in editor | `kubectl edit deployment web` |
| `patch` | Partial update via JSON/merge | `kubectl patch pod web -p '{"spec":...}'` |
| `replace` | Full replace from file | `kubectl replace -f pod.yaml` |
| `logs` | Container stdout/stderr | `kubectl logs web-pod` |
| `exec` | Run command inside container | `kubectl exec -it web -- bash` |
| `port-forward` | Forward local port to pod/service | `kubectl port-forward pod/web 8080:80` |
| `cp` | Copy files to/from container | `kubectl cp web:/app/log.txt ./log.txt` |
| `rollout` | Manage deployment rollouts | `kubectl rollout status deploy/web` |
| `scale` | Change replica count | `kubectl scale deploy/web --replicas=5` |
| `top` | CPU/memory usage | `kubectl top pods` |
| `diff` | Show what would change | `kubectl diff -f deploy.yaml` |
| `run` | Create a pod quickly (imperative) | `kubectl run nginx --image=nginx:1.27` |
| `expose` | Create a Service for a resource | `kubectl expose pod web --port=80` |

---

## kubeconfig — Context & Cluster Management ⭐

```
kubeconfig
 ├── clusters[]     ← API server URL + CA cert
 ├── users[]        ← credentials (cert, token, OIDC)
 └── contexts[]     ← cluster + user + namespace combo
         │
         └── current-context  ← the active one
```

| Command | What it does |
|---|---|
| `kubectl config view` | Show full kubeconfig |
| `kubectl config current-context` | Show active context |
| `kubectl config get-contexts` | List all contexts |
| `kubectl config use-context <name>` | Switch active context |
| `kubectl config set-context --current --namespace=prod` | Change default namespace |
| `kubectl cluster-info` | Show API server + CoreDNS address |

---

## Namespace Flag ⭐

| Command | Scope |
|---|---|
| `kubectl get pods` | Current namespace (from context) |
| `kubectl get pods -n kube-system` | Specific namespace |
| `kubectl get pods --all-namespaces` | All namespaces |
| `kubectl get pods -A` | Shorthand for `--all-namespaces` |

> Set default namespace for current context:
> `kubectl config set-context --current --namespace=production`

---

## Output Formats ⭐

| Flag | Output | Use Case |
|---|---|---|
| *(default)* | Clean table | Quick overview |
| `-o wide` | Extra columns (node, IP) | Node placement checks |
| `-o yaml` | Full YAML of live object | Inspect actual state |
| `-o json` | Full JSON | Scripting / jq piping |
| `-o name` | Just `kind/name` | Scripting loops |
| `-o jsonpath='{.status.phase}'` | Extract specific field | Automation |
| `-o custom-columns=NAME:.metadata.name` | Custom table | Reports |

**jsonpath examples:**
```bash
# Get all pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Get image of first container in a pod
kubectl get pod web -o jsonpath='{.spec.containers[0].image}'

# Get all node IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[0].address}'
```

---

## Filtering & Selecting Resources

### By Label (`-l` / `--selector`)
```bash
kubectl get pods -l app=web                  # exact match
kubectl get pods -l app=web,env=prod         # multiple labels (AND)
kubectl get pods -l 'env in (dev,staging)'   # set-based
kubectl get pods -l '!canary'                # label does NOT exist
```

### By Field (`--field-selector`)
```bash
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector spec.nodeName=node-1
kubectl get events --field-selector type=Warning
```

### Sort results
```bash
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.phase
```

---

## Applying Changes — apply vs create vs replace

| Command | Behaviour | When to use |
|---|---|---|
| `kubectl apply -f` | Create if missing, update if exists | **Default — always prefer this** |
| `kubectl create -f` | Create only — fails if exists | First-time object creation |
| `kubectl replace -f` | Full replace — must already exist | Force full spec rewrite |
| `kubectl apply --server-side -f` | Server-side apply (better conflict handling) | GitOps / large teams |

---

## Dry-run & Diff ⭐

```bash
# Client-side: validates YAML syntax only (no API call)
kubectl apply --dry-run=client -f pod.yaml

# Server-side: runs admission webhooks WITHOUT creating (strongest validation)
kubectl apply --dry-run=server -f pod.yaml

# Show diff between file and live cluster state
kubectl diff -f pod.yaml
```

| Method | Validates | API call? | Admission webhooks? |
|---|---|---|---|
| `--dry-run=client` | Syntax + schema | No | No |
| `--dry-run=server` | Full server validation | Yes (not persisted) | Yes |
| `diff` | Shows delta vs live | Yes | No |

---

## Imperative Commands (Quick / One-Liners)

```bash
# Create a pod
kubectl run web --image=nginx:1.27 --port=80

# Generate YAML without creating (useful starting point)
kubectl run web --image=nginx:1.27 --dry-run=client -o yaml > pod.yaml

# Create a deployment
kubectl create deployment web --image=nginx:1.27 --replicas=3

# Expose a deployment as a service
kubectl expose deployment web --port=80 --target-port=8080 --type=ClusterIP

# Create a configmap from literal
kubectl create configmap app-config --from-literal=LOG_LEVEL=info

# Create a secret from literal
kubectl create secret generic db-secret --from-literal=password=S3cr3t

# Scale
kubectl scale deployment web --replicas=5

# Set a new image
kubectl set image deployment/web web=nginx:1.28
```

---

## Logs ⭐

```bash
kubectl logs <pod>                         # current container logs
kubectl logs <pod> -c <container>          # specific container
kubectl logs <pod> --previous              # last crashed container
kubectl logs <pod> -f                      # follow (stream live)
kubectl logs <pod> --tail=50               # last 50 lines
kubectl logs <pod> --since=1h             # last 1 hour
kubectl logs -l app=web                    # logs from all pods with label
```

---

## Exec & Debug ⭐

```bash
# Shell into running container
kubectl exec -it <pod> -- bash
kubectl exec -it <pod> -- sh              # if bash not available

# Run a single command
kubectl exec <pod> -- cat /etc/config

# Specific container in multi-container pod
kubectl exec -it <pod> -c <container> -- bash

# Add ephemeral debug container (non-distroless)
kubectl debug -it <pod> --image=busybox --target=<container>

# Debug a node
kubectl debug node/<node-name> -it --image=ubuntu
```

---

## Port Forwarding

```bash
# Forward local 8080 to pod port 80
kubectl port-forward pod/web 8080:80

# Forward to a service
kubectl port-forward svc/web-svc 8080:80

# Forward to a deployment (picks a pod)
kubectl port-forward deployment/web 8080:80
```

> Useful for accessing internal services without a LoadBalancer or Ingress.

---

## Rollout Management ⭐

```bash
kubectl rollout status deployment/web           # watch rollout progress
kubectl rollout history deployment/web          # list revisions
kubectl rollout undo deployment/web             # roll back one revision
kubectl rollout undo deployment/web --to-revision=2  # roll back to specific
kubectl rollout pause deployment/web            # pause mid-rollout
kubectl rollout resume deployment/web           # resume
```

| Command | Use |
|---|---|
| `status` | Is the rollout done or stuck? |
| `history` | What versions exist? |
| `undo` | Revert last change |
| `pause/resume` | Canary-style controlled rollout |

---

## Resource Discovery ⭐

```bash
kubectl api-resources                     # all resource types + short names + apiVersion
kubectl api-resources --namespaced=false  # cluster-scoped only
kubectl api-versions                      # all available API groups/versions
kubectl explain pod                       # schema docs for Pod
kubectl explain pod.spec.containers       # nested field docs
kubectl explain pod.spec --recursive      # full tree
```

---

## Events & Troubleshooting

```bash
kubectl get events                                      # all events in namespace
kubectl get events --sort-by=.lastTimestamp             # newest last
kubectl get events --field-selector type=Warning        # warnings only
kubectl get events -n kube-system                       # system events
```

**Fast triage checklist:**
```bash
kubectl get pods -o wide                 # status + node placement
kubectl describe pod <name>             # events section = root cause
kubectl logs <name> --previous          # last crash logs
kubectl get events --sort-by=.lastTimestamp
kubectl top pod <name>                  # CPU/memory (needs metrics-server)
```

---

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `connection refused` | API server unreachable or kubeconfig wrong | Check `kubectl cluster-info`; verify context |
| `Unauthorized` | Wrong credentials or expired token | Refresh token; check `kubectl config view` |
| `Forbidden` (RBAC) | Missing permission | Check `kubectl auth can-i <verb> <resource>`; add Role + Binding |
| `no such resource` | Wrong resource name or API version | Use `kubectl api-resources` to find correct name |
| `Error from server (NotFound)` | Resource doesn't exist in that namespace | Check namespace with `-n`; verify name |
| `context not found` | Wrong context name | `kubectl config get-contexts` to list available |
| `error: the server doesn't have a resource type "X"` | CRD not installed or typo | `kubectl api-resources \| grep X` |
| `ImagePullBackOff` | Bad image name/tag | Fix image; add `imagePullSecrets` |
| `unable to forward port` | Pod not running / port wrong | Check pod status; verify port number |

---

## RBAC — Check Permissions

```bash
kubectl auth can-i get pods                          # can I get pods?
kubectl auth can-i create deployment -n production   # in specific namespace
kubectl auth can-i '*' '*'                           # am I cluster-admin?
kubectl auth can-i get pods --as=dev-user            # check as another user
```

---

## Best Practices

- **Use `apply` not `create`** — idempotent, works in CI/CD pipelines
- **Always `--dry-run=server` before applying** to catch errors early
- **Use `-n <namespace>`** explicitly — don't rely on default context namespace
- **`kubectl get events`** is the fastest way to understand why something failed
- **`kubectl describe`** before `kubectl logs` — events tell you what Kubernetes did
- **Generate YAML from imperative commands** with `--dry-run=client -o yaml` instead of writing from scratch
- **Never `kubectl edit` in production** — use GitOps (apply from a file in version control)
- **Use `kubectl diff`** before every `kubectl apply` in production
- **Set meaningful context names** — `dev`, `staging`, `prod` not `kubernetes-admin@cluster.local`
- **Use short names** to save time: `po`=pods, `svc`=services, `deploy`=deployments, `ns`=namespaces, `cm`=configmaps

---

## Short Names Cheatsheet

| Short | Full Resource |
|---|---|
| `po` | pods |
| `svc` | services |
| `deploy` | deployments |
| `rs` | replicasets |
| `ds` | daemonsets |
| `sts` | statefulsets |
| `cm` | configmaps |
| `ns` | namespaces |
| `no` | nodes |
| `pv` | persistentvolumes |
| `pvc` | persistentvolumeclaims |
| `sa` | serviceaccounts |
| `ep` | endpoints |
| `ing` | ingresses |
| `hpa` | horizontalpodautoscalers |

```bash
# All equivalent:
kubectl get pods
kubectl get po

kubectl get deployments
kubectl get deploy
```

---

## Key Flags Reference ⭐

| Flag | Short | Meaning |
|---|---|---|
| `--namespace` | `-n` | Target namespace |
| `--all-namespaces` | `-A` | All namespaces |
| `--output` | `-o` | Output format |
| `--filename` | `-f` | File or directory |
| `--selector` | `-l` | Label selector |
| `--watch` | `-w` | Watch for changes |
| `--force` | — | Force delete (skips graceful) |
| `--grace-period=0` | — | Immediate termination |
| `--record` | — | Record command in rollout history (deprecated) |
| `--dry-run=client` | — | Client-side validation |
| `--dry-run=server` | — | Server-side validation |
| `--context` | — | Override active context |
| `--kubeconfig` | — | Override config file path |

---

*Master kubectl and you can manage any Kubernetes cluster — it's the single interface for everything.*
