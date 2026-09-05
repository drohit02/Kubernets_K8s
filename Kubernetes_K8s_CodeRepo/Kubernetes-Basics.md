# Kubernetes Architecture and Components

# 1. What is Kubernetes Architecture?

Kubernetes Architecture defines how a Kubernetes Cluster is structured and how different components work together to manage applications and infrastructure.

A Kubernetes Cluster is mainly divided into two major parts:

```text
Kubernetes Cluster
│
├── Control Plane
│
└── Worker Nodes
```

The Control Plane manages and controls the entire Kubernetes Cluster.

Worker Nodes run the actual application workloads.

---

# 2. High-Level Kubernetes Architecture

```text
                         Kubernetes Cluster
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
              ▼                                   ▼
        Control Plane                        Worker Nodes
      (Cluster Management)                (Run Applications)
              │                                   │
              │                       ┌───────────┼───────────┐
              │                       │           │           │
              │                       ▼           ▼           ▼
              │                    Node 1      Node 2      Node 3
              │                       │           │           │
              │                     Pods        Pods        Pods
              │                       │           │           │
              │                  Containers  Containers  Containers
              │
              ├── API Server
              ├── Scheduler
              ├── Controller Manager
              └── etcd
```

The architecture can be understood simply as:

```text
Control Plane = Manages the Cluster

Worker Nodes  = Run the Applications
```

---

# 3. Kubernetes Cluster

A Kubernetes Cluster is a group of machines working together under Kubernetes management.

These machines can be:

- Virtual Machines
- Physical Servers
- Cloud Instances

A Kubernetes Cluster contains:

```text
Kubernetes Cluster
│
├── Control Plane
│   ├── API Server
│   ├── Scheduler
│   ├── Controller Manager
│   └── etcd
│
└── Worker Nodes
    ├── kubelet
    ├── kube-proxy
    └── Container Runtime
```

The Control Plane manages the state of the cluster.

Worker Nodes execute the workloads instructed by the Control Plane.

---

# 4. Control Plane

The Control Plane is the brain of the Kubernetes Cluster.

It is responsible for managing and controlling the overall Kubernetes environment.

The Control Plane makes important decisions such as:

- Where a Pod should run
- Monitoring the cluster state
- Maintaining the desired state
- Managing cluster resources
- Detecting failures
- Scheduling workloads
- Storing cluster configuration and state

The main Control Plane components are:

```text
Control Plane
│
├── kube-apiserver
│
├── etcd
│
├── kube-scheduler
│
└── kube-controller-manager
```

In some environments, the Control Plane can run on a single machine.

In production environments, multiple Control Plane nodes are commonly used for High Availability.

---

# 5. kube-apiserver

The kube-apiserver is the main entry point of the Kubernetes Control Plane.

Almost all communication with the Kubernetes Cluster goes through the API Server.

For example:

```text
Developer
    │
    │ kubectl commands
    ▼
API Server
    │
    ▼
Kubernetes Cluster
```

When you run:

```bash
kubectl get pods
```

The request does not directly communicate with the Worker Node.

The communication flow is:

```text
kubectl
   │
   ▼
kube-apiserver
   │
   ▼
Kubernetes Components
```

The API Server handles requests from:

- kubectl
- Kubernetes Dashboard
- External API clients
- Internal Kubernetes components

---

## API Server Responsibilities

The API Server is responsible for:

- Receiving API requests
- Authenticating requests
- Authorizing requests
- Validating requests
- Processing Kubernetes resource operations
- Communicating with etcd

Simplified flow:

```text
User
 │
 │ kubectl apply
 ▼
API Server
 │
 ├── Authentication
 │
 ├── Authorization
 │
 ├── Validation
 │
 ▼
etcd
```

The API Server acts as the central communication hub of Kubernetes.

Most Kubernetes components communicate through the API Server rather than directly communicating with each other.

---

# 6. etcd

etcd is the key-value database used by Kubernetes.

It stores the configuration and state of the Kubernetes Cluster.

Important information stored in etcd includes:

- Cluster configuration
- Node information
- Pod information
- Deployment configuration
- Service configuration
- Secrets
- ConfigMaps
- Desired state of Kubernetes resources

Example:

```text
etcd

├── Cluster Information
├── Node Information
├── Pod Information
├── Deployment Information
├── Service Information
├── ConfigMap Information
└── Secret Information
```

A simplified understanding:

```text
etcd = Database of Kubernetes Cluster State
```

---

## Example

Suppose you create a Deployment with:

```yaml
replicas: 3
```

The desired configuration is stored in the Kubernetes cluster state.

Simplified flow:

```text
kubectl apply
      │
      ▼
API Server
      │
      ▼
etcd
      │
      ▼
Cluster State Updated
```

etcd is one of the most critical components of Kubernetes.

If etcd data is lost, important Kubernetes cluster configuration and state information can be lost.

For this reason, etcd backups are very important in self-managed Kubernetes environments.

---

# 7. kube-scheduler

The kube-scheduler is responsible for deciding:

> Which Worker Node should run a newly created Pod?

When a new Pod is created, Kubernetes needs to decide where the Pod should run.

Example:

```text
New Pod Created
       │
       ▼
 kube-scheduler
       │
       ▼
Select Best Node
       │
       ├── Node 1 ❌
       ├── Node 2 ❌
       └── Node 3 ✅
              │
              ▼
          Run Pod
```

The Scheduler does not directly create the container.

It selects the most suitable Worker Node for the Pod.

---

## How Does Scheduler Select a Node?

The Scheduler considers multiple factors, such as:

- CPU availability
- Memory availability
- Resource requests
- Resource limits
- Node selectors
- Node affinity
- Pod affinity
- Pod anti-affinity
- Taints and tolerations

At the basic level, remember:

```text
kube-scheduler = Decides where a Pod should run
```

---

# 8. kube-controller-manager

The kube-controller-manager runs multiple controllers responsible for maintaining the desired state of the Kubernetes Cluster.

A Controller continuously monitors Kubernetes resources.

It compares:

```text
Desired State
      vs
Actual State
```

If there is a difference, the Controller attempts to correct it.

Example:

```text
Desired State = 3 Pods

Actual State = 2 Pods
```

The Controller detects the difference:

```text
Desired = 3
Actual  = 2
```

Then Kubernetes takes action to restore the desired state.

```text
Create New Pod
      │
      ▼
Actual State = 3 Pods
```

---

## Important Controllers

The kube-controller-manager manages several controllers.

Some important controllers include:

### Node Controller

Monitors the status of Nodes.

Example:

```text
Node 1 = Healthy
Node 2 = Healthy
Node 3 = Not Responding
```

The Node Controller monitors and manages node-related situations.

---

### ReplicaSet Controller

Maintains the required number of Pod replicas.

Example:

```text
Required Pods = 3

Current Pods = 2

Action:
Create 1 New Pod
```

---

### Deployment Controller

Manages Deployment-related operations.

For example:

- Creating ReplicaSets
- Managing application updates
- Rolling updates
- Rolling back Deployments

---

## Simple Controller Concept

```text
Controller
    │
    ▼
Check Desired State
    │
    ▼
Check Actual State
    │
    ▼
Are They Equal?
    │
 ┌──┴─────┐
 │        │
Yes       No
 │        │
 ▼        ▼
Wait     Correct State
```

---

# 9. Control Plane Complete Flow

The major Control Plane components work together like this:

```text
Developer
    │
    │ kubectl apply
    ▼
kube-apiserver
    │
    ├──────────────► etcd
    │                 │
    │                 ▼
    │            Store Cluster State
    │
    ├──────────────► Scheduler
    │                 │
    │                 ▼
    │            Select Worker Node
    │
    └──────────────► Controller Manager
                      │
                      ▼
                Maintain Desired State
```

---

# 10. Worker Node

Worker Nodes are the machines responsible for running application workloads.

A Worker Node can be:

- Physical Server
- Virtual Machine
- Cloud Instance

Worker Nodes contain the components required to run Pods and containers.

A Worker Node mainly contains:

```text
Worker Node
│
├── kubelet
│
├── kube-proxy
│
├── Container Runtime
│
└── Pods
```

Example:

```text
Worker Node
│
├── kubelet
├── kube-proxy
├── Container Runtime
│
├── Pod 1
│   └── Container
│
├── Pod 2
│   └── Container
│
└── Pod 3
    └── Container
```

---

# 11. kubelet

kubelet is an agent that runs on every Worker Node.

Its primary responsibility is to communicate with the Kubernetes Control Plane and ensure that the required Pods are running on the Node.

Simplified:

```text
Control Plane
      │
      │ Pod should run on Node
      ▼
    kubelet
      │
      ▼
Container Runtime
      │
      ▼
Create Container
```

The kubelet continuously checks the status of Pods running on its Node.

It reports the Node and Pod status back to the Control Plane.

---

## kubelet Responsibilities

kubelet is responsible for:

- Communicating with the API Server
- Receiving Pod specifications
- Ensuring required Pods are running
- Managing Pod lifecycle
- Monitoring container status
- Reporting Node status
- Reporting Pod status

Important:

```text
kubelet does NOT decide where the Pod should run.

kube-scheduler decides the Node.

kubelet ensures the Pod runs on the assigned Node.
```

---

# 12. Container Runtime

The Container Runtime is responsible for actually running containers.

Kubernetes itself does not directly execute containers.

The Container Runtime performs container-related operations.

Examples include:

- containerd
- CRI-O

The flow is:

```text
kubelet
   │
   ▼
Container Runtime
   │
   ▼
Create Container
   │
   ▼
Run Application
```

---

## Container Runtime Responsibilities

The Container Runtime is responsible for:

- Pulling container images
- Creating containers
- Starting containers
- Stopping containers
- Managing container lifecycle

Example:

```text
Container Image
      │
      ▼
Container Runtime
      │
      ▼
Pull Image
      │
      ▼
Create Container
      │
      ▼
Run Container
```

---

# 13. kube-proxy

kube-proxy is a networking component that runs on Worker Nodes.

It helps manage network communication for Kubernetes Services.

Simplified architecture:

```text
User Request
     │
     ▼
Service
     │
     ▼
kube-proxy
     │
     ├────► Pod 1
     │
     ├────► Pod 2
     │
     └────► Pod 3
```

kube-proxy helps implement Service networking and traffic forwarding rules.

At a basic level:

```text
kube-proxy = Helps manage network communication and Service traffic on Nodes
```

We will study Kubernetes networking and Services separately in detail.

---

# 14. Complete Worker Node Architecture

```text
                    Worker Node
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
     kubelet        kube-proxy    Container Runtime
        │                               │
        │                               ▼
        │                          Run Containers
        │                               │
        └───────────────┬───────────────┘
                        │
                        ▼
                       Pods
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
           Pod 1      Pod 2      Pod 3
             │          │          │
             ▼          ▼          ▼
        Container   Container   Container
```

---

# 15. Control Plane and Worker Node Communication

Let's understand how everything works together.

Suppose you deploy an application.

```bash
kubectl apply -f deployment.yaml
```

The flow is:

```text
1. Developer
      │
      ▼
2. kubectl
      │
      ▼
3. kube-apiserver
      │
      ├── Store Configuration in etcd
      │
      ├── Controller Manager Detects Required Resources
      │
      └── Scheduler Selects Worker Node
              │
              ▼
4. Worker Node
      │
      ▼
5. kubelet Receives Pod Requirement
      │
      ▼
6. Container Runtime
      │
      ▼
7. Container Starts
```

Complete architecture flow:

```text
Developer
    │
    │ kubectl apply
    ▼
┌─────────────────────────────┐
│        CONTROL PLANE        │
│                             │
│  ┌───────────────────────┐  │
│  │     API Server        │  │
│  └───────────┬───────────┘  │
│              │              │
│      ┌───────┼───────┐      │
│      ▼       ▼       ▼      │
│    etcd   Scheduler Controller│
│             │      Manager   │
└─────────────┼────────────────┘
              │
              │ Assign Pod
              ▼
┌─────────────────────────────┐
│        WORKER NODE          │
│                             │
│         kubelet             │
│            │                │
│            ▼                │
│    Container Runtime        │
│            │                │
│            ▼                │
│           Pod               │
│            │                │
│            ▼                │
│        Container            │
└─────────────────────────────┘
```

---

# 16. Control Plane vs Worker Node

| Control Plane | Worker Node |
|---|---|
| Manages the Kubernetes Cluster | Runs application workloads |
| Makes cluster-level decisions | Executes assigned workloads |
| Stores and manages cluster state | Runs Pods and Containers |
| Contains API Server | Contains kubelet |
| Contains Scheduler | Contains kube-proxy |
| Contains Controller Manager | Contains Container Runtime |
| Uses etcd for cluster state storage | Hosts application Pods |

---

# 17. Kubernetes Architecture Short Notes

## Control Plane

- Control Plane is the brain of Kubernetes.
- It manages the overall Kubernetes Cluster.
- It maintains the desired state.
- It makes scheduling decisions.
- It monitors cluster resources.
- It communicates with Worker Nodes.

### Control Plane Components

```text
API Server
    │
    ├── Main entry point of Kubernetes
    ├── Handles API requests
    └── Communicates with cluster components

etcd
    │
    ├── Key-value database
    └── Stores Kubernetes cluster state

Scheduler
    │
    ├── Selects Worker Node
    └── Schedules Pods

Controller Manager
    │
    ├── Runs Kubernetes controllers
    └── Maintains desired state
```

---

## Worker Node

- Worker Nodes run application workloads.
- Worker Nodes run Pods.
- Pods contain containers.
- Every Worker Node runs kubelet.
- kubelet communicates with the Control Plane.
- Container Runtime runs containers.
- kube-proxy manages Service networking rules.

### Worker Node Components

```text
kubelet
    │
    ├── Runs on every Worker Node
    ├── Communicates with API Server
    └── Ensures Pods are running

Container Runtime
    │
    ├── Pulls container images
    ├── Creates containers
    └── Runs containers

kube-proxy
    │
    └── Helps manage Kubernetes Service networking
```

---

# 18. Final Kubernetes Architecture Diagram

```text
                         KUBERNETES CLUSTER
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
              ▼                                   ▼
       CONTROL PLANE                         WORKER NODES
              │                                   │
   ┌──────────┼──────────┐              ┌─────────┼─────────┐
   │          │          │              │         │         │
   ▼          ▼          ▼              ▼         ▼         ▼
API Server   etcd    Scheduler      kubelet   kube-proxy  Container
                      │                            │       Runtime
              Controller Manager                   │         │
                                                   │         │
                                             ┌─────┴─────────┴─────┐
                                             │                     │
                                             ▼                     ▼
                                           Pods                  Pods
                                             │                     │
                                             ▼                     ▼
                                        Containers            Containers
```

---

# 19. Important Points to Remember

```text
Control Plane
    = Manages the Cluster

API Server
    = Main Communication Entry Point

etcd
    = Stores Cluster State

Scheduler
    = Decides Where Pods Run

Controller Manager
    = Maintains Desired State

Worker Node
    = Runs Application Workloads

kubelet
    = Ensures Pods Run on Assigned Node

Container Runtime
    = Actually Runs Containers

kube-proxy
    = Helps Manage Service Networking
```
