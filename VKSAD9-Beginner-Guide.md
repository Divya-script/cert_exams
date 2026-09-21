# VKSAD9 — The Complete Beginner's Guide (from zero)

*For someone who has never touched virtualization, servers, or Kubernetes. Read this top to bottom. Every new word is explained with an everyday analogy before we use it.*

This guide pairs with 4 diagrams (open the artifact panel):
1. From a Physical Server to a vSphere Supervisor
2. Networking & Load Balancer choices
3. Zones & Storage
4. The exam design-decision method

---

# Chapter 1 — The absolute basics (what is all this?)

Before any VMware words, let's build intuition.

### 1.1 What is a "server"?
A **server** is just a powerful computer that runs software for many people at once — like the computer behind a website or an app. It has **CPU** (brains), **RAM** (short-term memory), and **disk** (long-term storage).

**Problem:** One physical server is expensive, and most of the time it sits half-idle. Running one app per server wastes money.

### 1.2 What is "virtualization"? (the founding idea)
**Analogy:** Think of a large house. Instead of one family living in a huge empty house, you divide it into several **apartments**. Each apartment has its own door, kitchen, and rooms, and the tenants don't even know the others exist. You fit many families into one building.

**Virtualization** does this to a computer. It slices one physical server into many **Virtual Machines (VMs)**. Each VM behaves like its own complete computer — its own operating system, its own apps — but they all secretly share the one physical machine underneath.

- **Physical server** = the apartment building
- **VM** = one apartment
- The software that creates and manages the apartments = the **hypervisor**

### 1.3 What is ESX (ESXi)?
**ESX** is VMware's hypervisor. You install ESX directly onto a physical server, and from that moment the server can run many VMs. A server running ESX is called an **ESX host** (or just "host").

> Think: **host = the apartment building manager living in the basement**, handing out rooms to tenants (VMs).

### 1.4 What is vCenter?
If you have 3, 10, or 100 ESX hosts, you don't want to log into each one separately. **vCenter** is the single control tower that manages all your hosts and VMs from one screen.

> Think: **vCenter = the property-management company** that oversees every apartment building you own.

### 1.5 What is a "cluster"?
A **cluster** is a group of ESX hosts that vCenter treats as one big pool of resources. Two important cluster features:
- **vSphere HA (High Availability):** if one host dies, the VMs that were on it automatically restart on the surviving hosts. *(Like if one apartment building loses power, tenants are instantly moved to your other buildings.)*
- **vSphere DRS (Distributed Resource Scheduler):** automatically balances VMs across hosts so none is overloaded. *(Like a manager moving tenants around so no building is overcrowded.)*

**You have now met the base layer.** Diagram 1 shows this exact stack: physical servers → ESX hosts → cluster (managed by vCenter).

### 1.6 What is vSAN? (storage)
Normally storage is a separate expensive box (a SAN). **vSAN** takes the local disks *inside* each ESX host and magically pools them into one shared storage area that all hosts can use.

> Think: **vSAN = the tenants pooling their storage lockers into one shared warehouse** everyone can access.

### 1.7 What is NSX? (networking)
**NSX** is software-defined networking. Instead of buying physical switches, routers, and firewalls, NSX creates them in software. This lets you build isolated virtual networks, firewalls, and load balancers on demand.

> Think: **NSX = being able to instantly build new roads, gates, and security checkpoints between apartments, without calling a construction crew.**

### 1.8 What is VCF?
**VMware Cloud Foundation (VCF)** is the whole bundle: ESX + vCenter + vSAN + NSX + management tools, packaged as one private-cloud platform. Everything in this course sits inside VCF.

> Think: **VCF = the entire planned city** (buildings + roads + utilities + management office), not just one building.

---

# Chapter 2 — Containers and Kubernetes (the "modern app" world)

The course is about running **modern apps** next to traditional VMs. To understand that, you need containers.

### 2.1 What is a container?
A **VM** carries a whole operating system (heavy, boots in minutes). A **container** packages just an app and what it needs — no full OS — so it's tiny and starts in seconds.

> **Analogy:**
> - A **VM** is a **house** — full foundation, plumbing, walls. Solid but slow and heavy to build.
> - A **container** is a **shipping container** — standardized, stackable, you can move and start thousands of them quickly.

### 2.2 What is Kubernetes? (often written "K8s")
When you have hundreds of containers, something must decide where each runs, restart the crashed ones, scale them up when busy, and network them together. **Kubernetes** is that automatic orchestrator.

> **Analogy:** Kubernetes is the **port authority / crane operator** at a shipping yard. It decides which container goes on which ship, replaces damaged ones, and adds more when demand rises — all automatically.

### 2.3 What does "declarative" mean? (very important word)
Kubernetes is **declarative**: you write down the **desired end state** ("I want 3 copies of this app running"), and the system continuously makes reality match that. You don't give step-by-step commands.

> **Analogy:** Declarative = telling a smart thermostat *"keep the room at 22°C."* You don't manually turn the heater on and off — you declare the goal and it maintains it. The opposite (**imperative**) would be "turn heater on, wait, turn it off, wait…"

Everything in this course is declarative. You'll see it described as *"declaring the desired state in YAML"* (YAML is just a simple text file format for writing that desired state).

---

# Chapter 3 — The star of the course: the vSphere Supervisor

### 3.1 The core idea
Normally, VMware (vSphere) runs **VMs**, and Kubernetes runs **containers** — two separate worlds, two separate skill sets, two separate tools.

The **vSphere Supervisor** merges them. You take an ordinary vSphere cluster and **switch on Kubernetes**. Now the hypervisor *itself* understands the Kubernetes API. From one platform you can create Kubernetes clusters, containers, AND regular VMs — all using the same declarative style.

> **Analogy:** Imagine your apartment building manager suddenly learns to *also* run a shipping port. Now, from the same front desk, tenants can rent apartments (VMs) OR ship containers (Kubernetes) — one manager, one desk, both services.

**Definition to memorize:** *A vSphere cluster that has Kubernetes workloads enabled is called a Supervisor.*

### 3.2 What runs the Supervisor? Its key parts
When you enable the Supervisor, several components appear. Here's each one in plain English:

| Component | What it is | Analogy |
|---|---|---|
| **Control Plane VM(s)** | The VM(s) that run the Kubernetes "brain"/API for the Supervisor. **1 VM = Simple**, **3 VMs = High Availability**. | The **front desk** where all requests come in. One desk, or three for redundancy. |
| **Spherelet** | A special version of the Kubernetes worker-agent (`kubelet`) built to run directly on ESX, so an ESX host can act as a Kubernetes worker node. | A **translator** that lets the building manager understand shipping-port instructions. |
| **CRX (Container Runtime Executive)** | A super-light VM that runs containers almost as fast as normal containers, but with the strong isolation of a VM. | A **shipping container that's secretly armored** — fast like a container, safe like a vault. |
| **wcpsvc (Workload Control Plane Service)** | The vCenter service that turns the Supervisor on and manages it. | The **city permit office** that authorizes turning a building into a port. |
| **Cluster API (CAPI)** | The Kubernetes standard for creating/managing whole clusters declaratively. | The **blueprint system** for building new ports on demand. |

### 3.3 What is a vSphere Pod?
A **vSphere Pod** is a tiny VM that runs one or more containers. Unlike normal Kubernetes (where containers on a host share one kernel), each vSphere Pod gets its **own** small Linux kernel — so it's more isolated and secure.

> **Exam facts:** vSphere Pods use CRX, are placed by DRS, are visible as vSphere objects, and are **NOT compatible with vMotion** (you can't live-migrate them).

### 3.4 The three "Services" you'll hear about constantly
On top of the Supervisor, you enable **Services** (certified Kubernetes operators). The three core ones:

1. **VKS (vSphere Kubernetes Service):** creates full, standard Kubernetes clusters for your dev teams. *(Formerly called TKGS / Tanzu Kubernetes Grid Service — old exam material may use that name.)*
2. **VM Service:** lets you create and manage **ordinary VMs** using Kubernetes-style YAML. This is for teams that want the Kubernetes workflow but still have apps that can't be containerized.
3. **Velero:** backup and restore for your clusters and workloads.

There are also **extensible services** (Harbor, Contour, ArgoCD…) covered in Chapter 8.

---

# Chapter 4 — vSphere Namespaces (fencing off resources for teams)

### 4.1 The idea
A **vSphere Namespace** is a fenced-off slice of the Supervisor where a team's workloads live. The admin decides how much CPU, memory, and storage that slice gets (a **quota**), who can access it (**permissions/RBAC**), and which storage and VM types are allowed.

> **Analogy:** A namespace is a **rented floor in an office building.** The landlord (vSphere admin) says: "Team A gets floor 3, with this much electricity and this many desks, and only these keycards open the door." Team A does whatever they want *inside* that floor, but can't exceed their limits or enter other floors.

### 4.2 The confusing part: vSphere Namespace ≠ Kubernetes namespace
This trips up beginners and shows up on the exam:
- A **vSphere Namespace** is a VMware concept — it's built on a **resource pool** (a way to carve out CPU/RAM) and adds permissions, storage policies, quotas.
- It **maps to** a Kubernetes namespace underneath (to enforce quotas in Kubernetes terms).
- **Key difference:** with a vSphere Namespace, the **vSphere admin controls user access** — that isn't how a plain Kubernetes namespace works.

> **Memory hook:** *vSphere Namespace = a resource pool with superpowers (access control + storage policies + libraries).* 

### 4.3 What "backed by a resource pool" means in multi-zone
If your Supervisor spans 3 zones (see Chapter 6), the namespace gets a resource pool in **each** cluster, and resources are drawn **equally**. Example from the course: *if you assign 300 MHz of CPU, 100 MHz comes from each of the 3 clusters.*

### 4.4 Single-tenant vs multi-tenant (a design choice)
- **Single-tenant:** each team/customer gets their own isolated namespace/instance. **Secure and customizable, but costs more.** *(Like a single-family house.)*
- **Multi-tenant:** many teams share one instance with isolation at the app level. **Cheaper and easier to maintain, but less isolated.** *(Like an apartment building.)*

### 4.5 RBAC & identity (who is allowed to do what)
**RBAC = Role-Based Access Control.** You grant permissions based on a person's role, not per-person.

- **Who are you? (Identity providers):**
  - **vCenter Single Sign-On (SSO)** — the default; can connect to your company's Active Directory/LDAP.
  - **External OIDC provider** — a modern login standard; the Supervisor uses a helper called **Pinniped** to log you into VKS clusters via the VCF command line.
- **What can you do? (Kubernetes RBAC pieces):**
  - **Role** = permissions **inside one namespace**; linked to a user with a **RoleBinding**.
  - **ClusterRole** = permissions **across the whole cluster**; linked with a **ClusterRoleBinding**.

> **Analogy:** A **Role/RoleBinding** is a keycard that works on **one floor**. A **ClusterRole/ClusterRoleBinding** is a master key for the **whole building**.

---

# Chapter 5 — Networking & Load Balancers (see Diagram 2)

This is the biggest source of exam questions, so go slow.

### 5.1 First, what is a "load balancer"?
When an app gets popular, one copy can't handle all the traffic, so you run several copies. A **load balancer** is the traffic cop standing in front, spreading incoming requests evenly across the copies, so no single one is overwhelmed.

> **Analogy:** A **load balancer = the host at a busy restaurant** who sends each new group to whichever waiter has the fewest tables.

- **Layer 4 (L4)** load balancing = routes by IP address/port (basic, fast).
- **Layer 7 (L7)** load balancing = routes by the *content* of the request, e.g. the URL path (smarter).

### 5.2 The three networking "stacks" — pick exactly ONE per Supervisor
When you enable a Supervisor, you choose how workloads connect to the network:

| Stack | Plain English | Load balancer options |
|---|---|---|
| **NSX VPC** (Virtual Private Clouds) | Full modern NSX, with isolated private-cloud networks per tenant. Best isolation & multi-tenancy. | NSX LB **or** Avi |
| **NSX Segment** | NSX overlay network using "segments" (virtual switches). Needed for vSphere Pods. | NSX LB **or** Avi |
| **VDS** (vSphere Distributed Switch) | The plain-vSphere option — **no NSX**. Simplest. | **Foundation LB** or Avi |

**Golden rule:** a Supervisor uses **either** the plain vSphere (VDS) stack **or** NSX — never both. A VDS Supervisor needs an external load balancer on the management network.

> **Analogy for the stacks:**
> - **VDS** = normal public roads with basic traffic lights. Simple, works, no fancy features.
> - **NSX Segment** = you build your own private road network with software.
> - **NSX VPC** = you build fully gated, separate neighborhoods, each with its own private road system and security — great when many different tenants must be kept apart.

### 5.3 The two load balancers — Avi vs Foundation
**Avi Load Balancer** (advanced, L4 **and** L7):
- **Controller** = the brain. Manages everything but carries **no** actual traffic.
- **Service Engines (SEs)** = the worker VMs that actually move the traffic. Each can host up to 1,000 virtual services. `vnic0` = management network, `vnic1–8` = data network.
- **AKO (Avi Kubernetes Operator)** = a pod inside the Supervisor that tells Avi what to do.
- Only **one Avi instance per Supervisor**, chosen at deployment; **cannot be mixed** with another load balancer.

**Foundation Load Balancer (FLB)** (basic):
- **Layer 4 only**, comes **bundled with vCenter**, and works **only with VDS networking**.
- Two wiring styles:
  - **Two-Arm** = 3 network interfaces (Virtual Server + Transit + Management). The normal choice.
  - **One-Arm** = 2 interfaces (combined Virtual Server/Transit + Management). Only supported in the **Simplified Supervisor**.

> **Memory hook:** **Foundation = "Free, basic, VDS-only, L4-only."** **Avi = "Advanced, L4+L7, has Controller + Service Engines + AKO."**

---

# Chapter 6 — Zones & Control Plane availability (see Diagram 3, top half)

### 6.1 What is a vSphere Zone?
A **vSphere Zone** is a **fault domain** — a boundary that can fail on its own without taking others down. **One zone = one vSphere cluster.** You may deploy a Supervisor across **only 1 zone or exactly 3 zones** (never 2, never 4).

> **Analogy:** Zones are like **putting copies of your business in 3 different cities.** If one city has a blackout, the other two keep running.

**Critical exam point:** there is **no automatic data replication across zones** at the infrastructure level. So if you spread an app across 3 zones, **the app itself must know how to replicate** (i.e., it must be a cloud-native app). VMware won't copy the data for you between zones.

### 6.2 The zone deployment models
"Management" means the Supervisor's own control-plane VMs. "Workloads" means your apps.

| Model | Mgmt zones | Workloads | Use it when |
|---|---|---|---|
| **Simplified Supervisor** | 1 (shared) | Same zone; **1 control-plane VM, single network, VM Service ONLY** (no vSphere Pods, no Services) | Quick start; grow later |
| **Single Mgmt Zone, Combined Workloads** | 1 | Same zone as mgmt | Small, simple |
| **Single Mgmt Zone, Isolated Workloads** | 1 | Separate workload zone(s) | Keep workloads away from mgmt |
| **Three Mgmt Zones, Combined Workloads** | 3 | Same 3 zones | Survive a full cluster failure |
| **Three Mgmt Zones, Isolated Workloads** | 3 | Dedicated 3 mgmt + separate workload zones | Highest isolation + availability |

### 6.3 Control Plane availability (how many "front desks")
- **Simple Control Plane** = **1** control-plane VM (can grow to 3 later).
- **High Availability Control Plane** = **3** control-plane VMs (survives one failing).
- Control-plane VM **sizes**: **Tiny** (2 CPU / 8 GB) → **Small** (4/16) → **Medium** (8/16) → **Large** (16/32). Bigger = supports more workloads.

---

# Chapter 7 — Storage (see Diagram 3, bottom half)

### 7.1 Two categories in VCF
- **Principal storage** = the main storage you pick when creating a domain. **vSAN is recommended** (or NFS, or VMFS-on-Fibre-Channel).
- **Supplemental storage** = extra capacity you bolt on later (vSAN storage clusters, NFS, iSCSI, NVMe/TCP, etc.).

### 7.2 How containers get persistent storage
Containers are disposable, but some apps need data that survives (a database, for example). That's a **Persistent Volume (PV)**. The pieces that connect Kubernetes storage to VMware storage:
- **CNS (Cloud Native Storage)** in vCenter — tracks the volumes.
- **CSI driver** — the standard connector Kubernetes uses to ask for storage.
- **FCD (First Class Disk)** — an improved virtual disk that backs the volumes.
- **SPBM (Storage Policy-Based Management)** — a **policy** that says "put this data on storage that meets these rules" (speed, redundancy, etc.).

> **Exam facts:** CNS-CSI supports **ReadWriteOnce** (one node writes at a time) block volumes, volume expansion, snapshots, and zones/topology. It does **NOT** support dynamic **ReadWriteMany** (shared-write file) volumes, encryption, Windows, Storage DRS, or Storage vMotion of attached volumes.

**Persistent Volume workflow (simple version):** a developer asks for storage (a **PVC = Persistent Volume Claim**) → the request mirrors to the Supervisor → CNS-CSI creates the volume on policy-compliant storage → both sides show it as **Bound** (ready to use).

### 7.3 Storage across zones
- **Zonal datastore** = lives in **one** zone. If that zone dies, the datastore dies with it.
- **Cross-zonal datastore** = spans zones.
- A multi-zone Supervisor needs a **common Zonal Storage Policy** that is **topology-aware** (knows about zones) and applied across all 3 zones — all managed by the **same vCenter** (no cross-vCenter). A Zonal Storage Policy has a strict **1:1 relationship** with its vCenter/Supervisor.

### 7.4 vSAN Stretched Cluster (a key availability design)
A **stretched cluster** takes one vSAN datastore and stretches it across **two sites (availability zones)**, and adds a small **witness** host in a **third** site to act as a tie-breaker (so the cluster can decide who's "alive" if the link breaks).

> **Analogy:** Two warehouses in two cities kept as perfect mirrors, plus a **referee in a third city** who breaks ties if the two ever disagree. If one city goes dark, the other keeps serving with no data loss.

Benefits: mirrored data, automatic failover, simplified disaster recovery.

### 7.5 vSAN storage clusters (formerly "vSAN MAX")
Dedicated storage-only clusters (built on vSAN **ESA**) that serve storage to other "client" clusters. Sizing guidance from the course: single-site **7+ hosts** (enables RAID-6); stretched **6 + 6 + 1**. Fast networking (25/100 GbE) between storage nodes; clients can use 10 GbE.

---

# Chapter 8 — Services on the Supervisor

### 8.1 VKS cluster deep-dive (Module 5)
- **VKS** provisions full Kubernetes clusters using **Cluster API** + **VM Service** under the hood.
- **Networking inside a VKS cluster (CNI = Container Network Interface):** choose **Antrea (default, uses Open vSwitch)** or **Calico (Linux bridge + BGP)**.
- **Service types:** ClusterIP (internal only), NodePort (a port on each node), **LoadBalancer** (via NSX/Avi/Foundation). **Ingress** (URL routing) via a controller like **Contour**.
- **VKS + zones:** single-zone namespace → HA at the ESX-host level. Multi-zone namespace → control-plane nodes auto-spread across zones; **you** control worker placement by defining a **NodePool** that maps each zone to a **FailureDomain**.

### 8.2 Velero (backup/restore) — the rules that get tested
- Backs up VKS clusters and vSphere Pods to **any S3-compatible** storage.
- **Supervisor backup/restore = done in the vCenter UI.** **VKS-cluster Velero = installed and run via the CLI.**
- Supervisor control-plane backup piggybacks on vCenter file-based backups; it captures **etcd, infra container images, the Kubernetes CA cert/key, and all namespaces/resources**. **Restoring vCenter does NOT restore the Supervisor control plane** (separate workflows).
- **Back up ONLY when** the Supervisor is enabled/Running or upgraded-and-Running. **Do NOT back up** during an error state, during/after a failed upgrade, or while a service is upgrading.
- **Do NOT restore** if: Kubernetes versions mismatch; you restored vCenter out of sync; you created VKS clusters or upgraded VKS after the backup; you created vSphere Pods after the backup.
- **Three ways to protect workload data (know the trade-offs):**
  - **CSI Snapshot** — best for CNS block volumes; crash-consistent; supports incremental, dedup, compression, encryption, parallel.
  - **File System Backup** — for non-CNS/NFS volumes; **not** crash-consistent (reads live files); default engine is **Kopia** (Restic deprecated).
  - **vSphere Plug-in Snapshot** — block-level, crash-consistent, direct snapshot access; **no changed-block-tracking, so every backup is FULL**; no dedup/parallel.

### 8.3 Extensible services (what each is FOR — one line each)
- **LCI (Local Consumption Interface):** a vSphere UI to deploy VMs/clusters/LBs/PVCs; auto-writes the YAML for you.
- **Harbor:** a private container image **registry** (with security scanning, signing, RBAC, OIDC/LDAP). **Needs Contour installed first.**
- **Contour:** an **Ingress controller** (URL routing) using Envoy. Often a prerequisite for Harbor.
- **External-DNS:** auto-creates DNS records for your services.
- **MinIO (on vDPP):** S3-style **object storage** as a stateful service.
- **Secret Store (backed by OpenBao):** central, encrypted **secret management**; injects secrets into Pods/VMs. (Windows not supported.)
- **ArgoCD:** **GitOps** — keeps your environment matching what's declared in a Git repository, automatically.
- **cert-manager:** automatic **certificate** management using **ClusterIssuers** (which represent certificate authorities).

---

# Chapter 9 — VM Service (Module 6)

### 9.1 The idea
**VM Service** lets developers create **normal VMs** with the same declarative Kubernetes YAML they use for containers. Great for apps that can't be containerized yet, so teams don't need two separate tools.

- Deploy VMs from **OVF or ISO** images.
- Configure them at boot with **cloud-init** (Linux) or **Sysprep** (Windows).
- Size them with a **VM Class**.

### 9.2 Components
- In vCenter: **VAPI Server** (API endpoint) + **wcpsvc**.
- In the Supervisor: **VM Operator** — the component that reconciles three custom resources: **`VirtualMachine`**, **`VirtualMachineClass`**, **`VirtualMachineImage`**.

### 9.3 VM Class
A **VM Class** is a reusable hardware template (how many vCPUs, how much memory, reservations). There are **16 default classes**; types are **Guaranteed** (resources reserved) or **Best-effort** (shared). Admins can make custom classes in the vSphere Client or via **DCLI** (`namespacemanagement virtualmachineclasses …`).

### 9.4 The workflow (who does what)
**Admin prep:** create namespace → assign storage policies → assign content libraries (VM images) → associate VM classes.
**Developer (via kubectl):** list images / classes / networks / storage classes → write a cloud-init ConfigMap + a VM spec YAML → `kubectl apply` → verify with `kubectl get virtualmachine`.

### 9.5 vGPU / PCI passthrough (for AI/graphics VMs)
To give a VM a slice of a physical NVIDIA GPU: hosts need GPUs in **Shared Direct** mode, images must be **EFI** boot, and you install a **vGPU Manager** on the ESX host + a **guest driver** in the VM. In 3-zone setups, the VM and its persistent volume must be in the **same zone**.

---

# Chapter 10 — How the exam actually works (see Diagram 4)

The assessment is **not** "click these buttons." It's a **design workshop**: you read a customer story and make **justified design decisions**. The example customer is **Rainpole e-Commerce Services (RCS)**.

### 10.1 The RCAR conceptual model
Every decision must trace back to four things. Learn these letters:

| Letter | Meaning | Example |
|---|---|---|
| **R** — Requirements | What the business/tech **needs** (BR = business, TR = technical) | "Provide a modern app platform," "99% uptime," "PCI-DSS compliant" |
| **C** — Constraints | Hard **limits** you can't break | "Must reuse existing networking/storage" |
| **A** — Assumptions | Things taken as **true** | "Datacenters can add more racks/hosts" |
| **R** — Risks | What could **go wrong** | "Team lacks VKS skills," "can't add multi-site after single-site is built" |

> **Analogy:** It's like an architect designing a house. Requirements = "3 bedrooms, wheelchair access." Constraints = "budget $X, this plot of land." Assumptions = "the soil is stable." Risks = "the builder may be inexperienced." Your design must honor all four.

### 10.2 The shape of one design decision (memorize this template)
For each decision you write:
1. **Decision ID** (e.g., DD-202)
2. **The question / choices** offered
3. **Your Decision**
4. **Conceptual Model Reference** — which R/C/A items justify it
5. **Business Justification** — why it meets the need
6. **Design Qualities Affected** — availability, security, performance, manageability…
7. **Implications** — the downside/risk you accept

### 10.3 The one habit that passes the exam
For **every** question, silently ask:
> *"Which option satisfies the Requirements, respects the Constraints, and leaves acceptable Risks?"*

The correct answer is almost never "the most powerful option" — it's the one that **fits the customer's stated situation**. Be ready to explain **why**.

---

# Chapter 11 — Fast revision sheet

- **Virtualization** = slice 1 server into many VMs. **ESX** = the hypervisor; a server with ESX = a **host**. **vCenter** = manages all hosts. **Cluster** = pooled hosts (with **HA** = auto-restart, **DRS** = auto-balance). **vSAN** = pooled disks. **NSX** = software networking. **VCF** = the whole bundle.
- **Container** = lightweight app package; **Kubernetes** = orchestrates many containers; **declarative** = you state the desired end-result and the system maintains it.
- **Supervisor** = a vSphere cluster with Kubernetes turned on. Parts: **Control Plane VM(s)**, **Spherelet** (kubelet on ESX), **CRX** (fast container runtime), **wcpsvc**, **CAPI**.
- **vSphere Pod** = tiny VM per container, own kernel, **no vMotion**.
- **Namespace** = fenced resource+access slice (a resource pool with superpowers); **≠** a Kubernetes namespace; **admin controls access**.
- **Networking:** pick one of **NSX VPC / NSX Segment / VDS**. **Foundation LB = VDS-only, L4-only, bundled.** **Avi = L4/L7, Controller+SEs+AKO, one per Supervisor.**
- **Zones:** only **1 or 3**; a zone = a cluster; **no cross-zone infra replication** (apps must self-replicate).
- **Control plane:** **1 (Simple)** or **3 (HA)**; sizes Tiny/Small/Medium/Large.
- **Storage:** Principal vs Supplemental; **CNS + CSI + FCD + SPBM**; **ReadWriteOnce yes, dynamic ReadWriteMany no**. Multi-zone needs a **topology-aware Zonal Storage Policy**. **Stretched cluster** = 2 sites + witness.
- **VKS CNI:** **Antrea (default)** or **Calico**; multi-zone workers = **NodePool → FailureDomain**.
- **Velero:** Supervisor backup via **UI**, VKS via **CLI**, target **S3-compatible**. Methods: **CSI / File-System (Kopia) / vSphere Plug-in (full every time, no CBT)**.
- **Extensible:** **Harbor (needs Contour)**, Contour, External-DNS, MinIO/vDPP, Secret Store (OpenBao), ArgoCD (GitOps), cert-manager.
- **VM Service:** **VM Operator** reconciles `VirtualMachine`/`VirtualMachineClass`/`VirtualMachineImage`; deploy from **OVF/ISO** + **cloud-init**; **VM Class** = hardware template.
- **Exam method:** every choice traces to **RCAR** (Requirements, Constraints, Assumptions, Risks). Pick the option that **fits the customer**, and justify it.

---

### Source
Summarized and rephrased for study from your course materials: **EDU-EN-VKSAD9-LEC-IE.pdf** (Lecture) and **EDU-EN-VKSAD9-LAB-IE.pdf** (Workshop), *vSphere Kubernetes Service: Advanced Design [V9.0]*, © Broadcom.
