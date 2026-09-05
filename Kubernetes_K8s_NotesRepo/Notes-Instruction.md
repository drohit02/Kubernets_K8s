# Kubernetes Docs — Notes & Instructions

> This file explains exactly how the documentation for each Kubernetes topic is structured.
> Follow this every time you add a new subfolder (Deployments, Services, ConfigMaps, etc.).

---

## Folder Structure (Target)

```
Kubernetes/
├── README.md                  ← master cheatsheet (all objects, YAML, commands)
├── Notes-Instruction.md       ← this file
│
├── Pods/
│   ├── simple-pod.yml         ← working YAML example
│   ├── POD-README.md          ← topic deep-dive reference
│   └── Que&Ans[Pod].docx      ← interview Q&A document
│
├── Deployments/
│   ├── simple-deployment.yml
│   ├── DEPLOYMENT-README.md
│   └── Que&Ans[Deployment].docx
│
├── Services/
│   ├── simple-service.yml
│   ├── SERVICE-README.md
│   └── Que&Ans[Service].docx
│
├── ConfigMaps/
│   ├── simple-configmap.yml
│   ├── CONFIGMAP-README.md
│   └── Que&Ans[ConfigMap].docx
│
├── Secrets/
│   ├── simple-secret.yml
│   ├── SECRET-README.md
│   └── Que&Ans[Secret].docx
│
└── ...more topics
```

---

## File Naming Convention

| File | Pattern | Example |
|---|---|---|
| YAML example | `simple-<topic>.yml` (lowercase) | `simple-deployment.yml` |
| Reference doc | `<TOPIC>-README.md` (uppercase topic) | `DEPLOYMENT-README.md` |
| Interview Q&A | `Que&Ans[<Topic>].docx` (PascalCase topic) | `Que&Ans[Deployment].docx` |

---

## What Each File Must Contain

### 1. `simple-<topic>.yml`

A clean, working YAML manifest for the topic. Rules:
- Include all **required fields** only — no bloat
- Add **inline comments** on every important line explaining what it does and why
- Pin image versions (no `:latest`)
- Use realistic names (`simple-pod-webapp`, not `my-pod`)

**Template comment style:**
```yaml
apiVersion: v1       # core API group — check with kubectl api-versions
kind: Pod            # case-sensitive PascalCase
metadata:
  name: simple-pod   # DNS-1123: lowercase, hyphens only, ≤253 chars
```

---

### 2. `<TOPIC>-README.md`

Deep-dive reference for that specific Kubernetes object. Keep it **visual, scannable, no long paragraphs**.

#### Required Sections (in order):

| # | Section | Content |
|---|---|---|
| 1 | **What is X?** | Overview table: apiVersion, kind, scope, self-healing, key feature |
| 2 | **Lifecycle / Flow** | ASCII diagram showing states or flow |
| 3 | **YAML Structure** | Full annotated YAML with comments |
| 4 | **Field Reference** | Tables split by area (Metadata, Spec, sub-sections) |
| 5 | **Key Concepts** | Comparisons — tables like `requests vs limits`, `types`, `modes` |
| 6 | **kubectl Commands** | Table: command + what it does |
| 7 | **Common Errors & Fixes** | Table: error → cause → fix |
| 8 | **Best Practices** | Bullet list, one line each |

#### Rules for the README:
- Use **tables** for anything that has 2+ comparable properties
- Use **ASCII diagrams** for flows, lifecycles, relationships
- Use **code blocks** for YAML, bash commands
- Use **bullet points** for lists — never long paragraphs
- Mark required fields with `🔴`, interview favorites with `⭐` (match style of main README.md)
- Keep each section tight — if a point needs more than one line, it's probably two bullet points

---

### 3. `Que&Ans[<Topic>].docx`

Interview question-and-answer document. **Not a wall of text** — structured, scannable, practical.

#### Document Structure:

```
Title: Kubernetes <Topic> — Interview Q&A
Subtitle: Simple to Medium Level | CKAD / CKA / DevOps Interviews

Section 1: Basic Concepts        (5 questions)
Section 2: Core Fields & Config  (6–8 questions)
Section 3: Lifecycle & Behavior  (4–5 questions)
Section 4: Intermediate/Tricky   (5–7 questions)

Quick-Fire Reference Table       (15–20 rows)
Essential kubectl Commands Table
```

#### Question Format:

```
Q<N>. <Question text>  [bold]

Answer: <One sentence core answer>  [indented]
  • Detail / rule / gotcha 1
  • Detail / rule / gotcha 2
  • Detail / rule / gotcha 3
```

#### Rules for the .docx:
- **Bold** the question — it should be scannable
- **One sentence** as the core answer — no rambling
- **3–5 bullets** for details, edge cases, or rules
- No unnecessary filler words ("It is important to note that...")
- Prefer concrete facts over vague descriptions
- End with a **Quick-Fire table** (question → crisp one-liner answer)
- End with an **essential kubectl commands table** for that topic

#### How to generate the .docx:

The .docx file is generated using a Python script that uses only the built-in `zipfile` module (no external libraries needed). The script:

1. Defines all Q&A content as Python data (list of tuples)
2. Builds Open XML (`word/document.xml`) with paragraphs, tables, bullets
3. Packages everything into a ZIP with the `.docx` extension

**To create the script for a new topic:**

```
Ask Claude:
"Create POD-README.md and Que&Ans[Pod].docx for the Pods folder,
same style as the existing Pods folder. Use tables, ASCII diagrams,
bullet points. No long paragraphs. Simple to medium interview questions."

Replace "Pod/Pods" with the new topic name.
```

The script will be generated, run with `python3 create_docx.py`, and then deleted — only the `.docx` output stays.

---

## Content Guidelines (Apply to All Files)

| Rule | Why |
|---|---|
| No long paragraphs | Hard to scan during revision |
| Tables over prose | Faster comparison at a glance |
| Bullet points over sentences | Easy to read, easy to memorize |
| Concrete examples over abstract | `nginx:1.27.0` beats "use a version tag" |
| Short answers first, details second | Interviewer wants the bottom line fast |
| No filler phrases | "It is important to note", "Please be aware" — cut these |
| Active voice | "Kubernetes restarts the container" not "the container is restarted by Kubernetes" |

---

## Topics to Cover (Checklist)

### Foundations
- [x] kubectl Basics

### Core Workloads
- [x] Pods
- [x] ReplicationController
- [ ] Deployments
- [x] ReplicaSets
- [ ] StatefulSets
- [ ] DaemonSets
- [ ] Jobs
- [ ] CronJobs

### Networking
- [ ] Services (ClusterIP, NodePort, LoadBalancer, Headless)
- [ ] Ingress
- [ ] NetworkPolicy

### Configuration & Storage
- [ ] ConfigMaps
- [ ] Secrets
- [ ] PersistentVolumes (PV)
- [ ] PersistentVolumeClaims (PVC)
- [ ] StorageClass

### Scaling & Scheduling
- [ ] HorizontalPodAutoscaler (HPA)
- [ ] VerticalPodAutoscaler (VPA)
- [ ] Resource Quotas
- [ ] LimitRange

### Organization & RBAC
- [ ] Namespaces
- [ ] ServiceAccounts
- [ ] Roles & RoleBindings
- [ ] ClusterRoles & ClusterRoleBindings

### Advanced
- [ ] Helm Charts
- [ ] Kustomize
- [ ] PodDisruptionBudget
- [ ] Taints & Tolerations
- [ ] Affinity & Anti-Affinity

---

## Quick Prompt Template (for Claude)

Copy this prompt when starting a new topic:

```
Topic: <TOPIC NAME>
Subfolder: <FOLDER NAME>

Create the following for the <FOLDER NAME> folder (same style as the Pods folder):
1. <TOPIC>-README.md — deep-dive reference with tables, ASCII diagrams,
   bullet points. Sections: overview, lifecycle, YAML structure, field reference,
   key concepts, kubectl commands, common errors, best practices.
   No long paragraphs. Mark required fields 🔴 and interview picks ⭐.

2. Que&Ans[<Topic>].docx — 20–25 interview questions, simple to medium.
   Format: bold question → one-sentence answer → 3–5 detail bullets.
   Sections: Basic, Core Config, Lifecycle/Behavior, Intermediate/Tricky.
   End with Quick-Fire table and kubectl commands table.

Keep it concise, meaningful, no filler words. Bullet points and tables always.
```

---

## Style Reference (Match the Main README.md)

| Element | Style |
|---|---|
| Required field marker | `🔴` |
| Interview-favorite marker | `⭐` |
| Headings | `##` for sections, `###` for sub-sections |
| Code blocks | Triple backtick with `yaml` or `bash` |
| Tables | GFM pipe tables, header row always |
| ASCII diagrams | Inside triple backtick (no language tag) |
| Inline code | Backtick — e.g. `kubectl apply` |
| Notes / warnings | `>` blockquote |
