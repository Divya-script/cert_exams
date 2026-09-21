# VKSAD9 Study Guide — vSphere Kubernetes Service: Advanced Design [V9.0]

A beginner-friendly guide to understanding and passing the **VKSAD9** course/exam. This is built directly from your course materials (the Lecture and Lab manuals in `EDU-EN-VKSAD9-Coursematerials`).

> **What is this course?** VKSAD9 teaches you how to *design* (not just deploy) a modern private-cloud platform where you run **Kubernetes clusters, VMs, and containers side-by-side** on VMware infrastructure. The heart of it is a thing called the **vSphere Supervisor**. The exam-style assessment is a **design workshop**: you read a customer scenario and make justified design decisions.

---

## Part 0 — Read this first: the mental model

If you're new to this, anchor everything to these five ideas:

1. **VCF (VMware Cloud Foundation)** is the whole private-cloud platform (compute + storage + networking + management), built on ESX hosts, vCenter, vSAN, and NSX.
2. **vSphere Supervisor** = you take an ordinary vSphere cluster and "switch on Kubernetes." Now the hypervisor itself speaks the Kubernetes API. You can create Kubernetes clusters, VMs, and pods using the same declarative style.
3. **vSphere Namespace** = a walled-off slice of resources (CPU/memory/storage + permissions) where a team's workloads run.
4. **VKS (vSphere Kubernetes Service)** = the service that spins up full, conformant Kubernetes clusters on top of the Supervisor. (Formerly called TKGS / Tanzu Kubernetes Grid Service.)
5. **Design** = choosing between options (networking, storage, zones, load balancers) and *justifying* each choice against the customer's Requirements, Constraints, Assumptions, and Risks (the "RCAR" list).

Keep asking: *"What problem does this solve, and when would I choose it over the alternative?"* That is exactly what the exam tests.

---

## Part 1 — Key vocabulary (learn these cold)

| Term | Plain-English meaning |
|---|---|
| **ESX / ESXi** | The hypervisor software that runs on a physical server ("host"). |
| **vCenter** | The central management server for all your ESX hosts. |
| **vSphere cluster** | A group of ESX hosts managed together (with HA and DRS). |
| **vSAN** | VMware's software that pools the local disks of hosts into one shared datastore. |
| **NSX** | VMware's software-defined networking (virtual switches, routers, firewalls, load balancers). |
| **VDS (vSphere Distributed Switch)** | A virtual switch that spans all hosts in a cluster; the "plain vSphere" networking option (no NSX). |
| **vSphere Supervisor** | A vSphere cluster with Kubernetes enabled. Exposes a Kubernetes API from the hypervisor. |
| **Supervisor Control Plane VM** | The VM(s) that run the Kubernetes API for the Supervisor (like Kubernetes control-plane nodes). 1 VM = simple; 3 VMs = HA. |
| **Spherelet** | A version of the Kubernetes `kubelet` ported to run natively on ESX, so an ESX host can act as a Kubernetes worker node. |
| **CRX (Container Runtime Executive)** | A lightweight VM-based container runtime that lets vSphere Pods boot almost as fast as containers but with VM-level isolation. |
| **vSphere Pod** | A tiny VM that runs one or more Linux containers, each with its own Photon-based kernel (strong isolation). |
| **vSphere Namespace** | A resource + access boundary on the Supervisor. Backed by a vSphere resource pool. |
| **vSphere Zone** | A logical grouping that maps 1:1 to a vSphere cluster; used for fault tolerance. You deploy on **1 or 3 zones**. |
| **VKS (vSphere Kubernetes Service)** | Service that provisions full Kubernetes clusters (formerly TKGS). |
| **VM Service** | Lets you create/manage regular VMs using Kubernetes-style declarative YAML. |
| **CAPI (Cluster API) / CAPV** | Kubernetes project (and its vSphere provider) used to declaratively create/manage clusters. |
| **VM Operator** | The Supervisor component that actually provisions the VMs behind VKS clusters and VM Service. |
| **Velero** | The integrated backup/restore tool for clusters and workloads. |
| **CNS / CSI** | Cloud Native Storage (in vCenter) and the Container Storage Interface driver — how Kubernetes persistent volumes map to vSphere storage. |
| **Avi Load Balancer** | An advanced (L4/L7) load balancer. Components: Controller + Service Engines + AKO. |
| **Foundation Load Balancer (FLB)** | A basic **L4-only** load balancer bundled with vCenter; **only** works with VDS networking. |

---

## Part 2 — The six modules, explained

The lecture guide has 6 modules. Here's what each one is really about.

### Module 1 — Course Introduction
Sets the scene. The whole course is about designing vSphere Supervisor + VKS on VCF 9.0: concepts, deployment models, networking, storage, zones, services, and VM Service.

### Module 2 — Supervisor Deployment Models (the biggest module)

**2a. What the Supervisor is and its parts**
- Enabling Kubernetes on a vSphere cluster turns it into a **Supervisor**. It runs on an SDDC layer: **ESX (compute) + VDS (networking) + vSAN or shared storage**.
- Key constructs: **vSphere Zones, Namespaces, vSphere Pods, Services** (VKS, VM Service, Velero).
- Core components: **Control Plane VM(s)**, **Cluster API (CAPI)**, **Spherelet**, **CRX**.
- In vCenter, the **Workload Control Plane Service (wcpsvc)** orchestrates enabling/managing the Supervisor.

**2b. Networking options** — pick ONE networking stack per Supervisor:

| Networking stack | Load balancer choices | When to use |
|---|---|---|
| **NSX VPC** (Virtual Private Clouds) | NSX LB or Avi | Full modern NSX; best isolation/multi-tenancy across zones. |
| **NSX Segment** | NSX LB or Avi | NSX overlay segments; supports vSphere Pods. |
| **VDS** (no NSX) | **Foundation LB** or Avi | Simplest; plain vSphere networking. FLB only works here. |

Remember: **A Supervisor uses either the vSphere (VDS) networking stack OR NSX** — not both. VDS-based Supervisors need an external load balancer on the management network.

**2c. Load balancers**
- **Avi**: `Controller` (control plane, no data traffic) + `Service Engines` (data plane VMs, up to 1,000 virtual services each; vnic0 = management, vnic1-8 = data) + `AKO` (the Kubernetes operator pod). Only **one** Avi instance per Supervisor, chosen at deploy time, and it can't be mixed with other LBs.
- **Foundation LB**: basic **L4 only**, bundled with vCenter, **VDS-only**. Two topologies:
  - **Two-Arm**: separate Virtual Server + Transit + Management interfaces (3 NICs).
  - **One-Arm**: combined Virtual Server/Transit + Management (2 NICs). One-Arm is only supported in the **Simplified Supervisor** deployment.

**2d. Zone deployment models** — how many zones for management vs. workloads:

| Model | Mgmt zones | Workload zones |
|---|---|---|
| **Simplified Supervisor** | 1 (combined) | 1 control-plane VM, single network, **VM Service only** — no vSphere Pods/Services. Scalable later. |
| **Single Mgmt Zone, Combined Workloads** | 1 | Same zone as mgmt. HA via vSphere HA. |
| **Single Mgmt Zone, Isolated Workloads** | 1 | Separate zone(s) for workloads. |
| **Three Mgmt Zones, Combined Workloads** | 3 | Same 3 zones; tolerates a full cluster failure. |
| **Three Mgmt Zones, Isolated Workloads** | 3 | Dedicated 3 mgmt zones + separate workload zones. |

**2e. Control plane availability**
- **Simple**: 1 control-plane VM (can scale up to 3 later).
- **High Availability**: 3 control-plane VMs.
- Control-plane VM sizes: **Tiny** (2 CPU/8 GB), **Small** (4/16), **Medium** (8/16), **Large** (16/32). Bigger size = more workloads supported.

### Module 3 — Storage Topologies

- **VCF storage categories**: **Principal storage** (chosen when creating a domain — vSAN recommended, or NFS/VMFS-FC) and **Supplemental storage** (added capacity — vSAN storage clusters, NFS, VMFS iSCSI, NVMe/TCP, etc.).
- **Cloud Native Storage (CNS)** + **vSphere CNS-CSI** connect Kubernetes persistent volumes to vSphere storage. **First Class Disk (FCD)** backs ReadWriteOnce volumes. Placement is driven by **SPBM (Storage Policy-Based Management)**.
- Supported: dynamic **ReadWriteOnce** block volumes, volume expansion, snapshots, topology/zones, stretched clusters. **Not** supported by CNS-CSI: dynamic ReadWriteMany (file) volumes, encryption, Windows, Storage DRS, Storage vMotion of attached volumes.
- **Zones and storage**:
  - **Zonal datastore** = local to one zone (dies if the zone dies).
  - **Cross-zonal datastore** = spans zones.
  - Multi-zone Supervisor needs a **common Zonal Storage Policy** (topology-aware, tag-based) applied across all 3 zones, all managed by the **same vCenter** (no cross-vCenter). A Zonal Storage Policy has a **1:1 relationship** with its vCenter/Supervisor.
  - **No infrastructure-level replication across zones** — so multi-zone apps must have **built-in replication** (i.e., cloud-native apps).
- **vSAN Stretched Cluster**: one vSAN datastore spans two availability zones + a **witness** host in a third site for quorum. Gives automatic failover and simplified DR.
- **vSAN storage clusters** (formerly vSAN MAX): ESA-only, disaggregated storage-only clusters serving other "client" clusters. Recommended sizing: single-site **7+ hosts** (for RAID-6); stretched **6+6+1**. Requires 25/100 GbE between server nodes; clients can use 10 GbE.
- **Persistent Volume workflow**: DevOps creates a PVC → mirrored PVC on the Supervisor → CNS-CSI calls CNS Create Volume → volume placed on a policy-compliant datastore → both VKS cluster and Supervisor show the PV/PVC as **Bound**.

### Module 4 — vSphere Namespaces and Zones

- A **vSphere Namespace** is a resource + access boundary. It is **not** the same as a Kubernetes namespace — a vSphere Namespace is an extension of a **resource pool** and maps *to* a Kubernetes namespace for quota enforcement.
- Each namespace gets its **own resource pool**. In a 3-zone Supervisor, a resource pool is created in **each** cluster and resources are drawn **equally** (e.g., 300 MHz total = 100 MHz per cluster).
- Admins set **quotas** (CPU, memory, storage, object counts), assign **storage policies**, **VM classes**, and **content libraries**, and control **access (RBAC)**.
- **Single-tenant vs. multi-tenant** namespace design: single-tenant = isolated, secure, costly; multi-tenant = shared, cost-efficient, less isolated.
- **RBAC & identity**:
  - Identity providers: **vCenter Single Sign-On** (default, integrates AD/LDAP) and **External OIDC provider** (uses **Pinniped** to connect to VKS clusters via the VCF CLI).
  - Kubernetes RBAC building blocks: **Role** (namespace-scoped) + **RoleBinding**; **ClusterRole** (cluster-wide) + **ClusterRoleBinding**.
  - Assigning **"Can edit"/"Can view"** in vSphere creates a Kubernetes RoleBinding mapping the user/group to a ClusterRole.
- **Folder hierarchy**: Supervisor = a folder; namespaces = child folders; each namespace has a restricted `vSpherePods` subfolder.

### Module 5 — Services in vSphere Supervisor

**5a. VKS cluster architecture**
- VKS uses three controller layers; built on **Cluster API** + **VM Service**.
- **CNI options**: **Antrea (default, uses Open vSwitch)** or **Calico (Linux bridge + BGP)**.
- Service types: ClusterIP, NodePort, and **LoadBalancer** (via NSX LB / Avi / Foundation LB). Ingress via a third-party controller like **Contour**.
- **Zones + VKS HA**: single-zone namespace → HA at ESX-host level (vSphere HA). Multi-zone namespace → control-plane nodes auto-spread across zones; **you** control worker placement using a **NodePool** mapping each zone to a **FailureDomain**.

**5b. Velero (backup/restore)**
- Core Supervisor service. Backs up VKS clusters and vSphere Pods to **any S3-compatible** storage.
- Supervisor backup/restore is done from the **vCenter UI**; VKS-cluster Velero is installed and run via **CLI**.
- **Supervisor control-plane backup** rides along with vCenter file-based backups; captures **etcd state, infra container images, the Kubernetes CA cert/key, and all namespaces/resources**. Restoring vCenter does **not** restore the Supervisor control plane (separate workflows).
- **When to back up**: only when the Supervisor is **enabled/Running** or **upgraded and Running**. **Do not** back up during an error state, during an upgrade, after a failed upgrade, or while a service (e.g. VKS) is upgrading.
- **When NOT to restore**: Kubernetes version mismatch; after upgrading/restoring vCenter out of sync; after creating VKS clusters or upgrading VKS post-backup; after creating vSphere Pods post-backup.
- **Workload protection methods** (know the trade-offs):
  - **CSI Snapshot** — recommended for CNS block volumes; crash-consistent; supports incremental, dedup, compression, encryption, parallelism.
  - **File System Backup** — for non-CNS/NFS volumes; not crash-consistent (reads live FS); default uploader is **Kopia** (Restic deprecated in 1.15).
  - **vSphere Plug-in Snapshot** — block-level, crash-consistent, direct snapshot access, network offload; **no CBT** so every backup is **full**; no dedup/parallelism.

**5c. Extensible services (know what each one is for)**
- **LCI (Local Consumption Interface)**: vSphere-UI plug-in to deploy/manage VMs, VKS clusters, LBs, PVCs; auto-generates YAML.
- **Harbor**: OCI container image registry (RBAC, OIDC/LDAP, vulnerability scanning, signed images, replication). **Requires Contour** first; needs an FQDN + DNS record to the Envoy ingress IP; trust via appending Harbor CA to the `image-fetcher-ca-bundle`.
- **Contour**: Ingress controller (sets up Envoy). Often a prerequisite for Harbor.
- **External-DNS**: auto-publishes DNS records for services/ingresses.
- **vSAN Data Persistence Platform (vDPP)**: framework for stateful third-party services; **MinIO** (S3-style object storage) is the example service.
- **Secret Store** (backed by **OpenBao**): centralized secret management; injects secrets into Pods/VMs; encrypted by default; Windows not supported.
- **ArgoCD**: GitOps continuous delivery — one API target keeps clusters/VMs/LBs/PVCs in the desired state defined in Git.
- **cert-manager**: certificate management using **ClusterIssuers** (represent CAs).

### Module 6 — Virtual Machines with VM Service

- **VM Service** lets DevOps run regular VMs with a declarative Kubernetes-style API — deploy from **OVF or ISO**, configure via **cloud-init / Sysprep**, size with a **VM Class**.
- **Components**: in vCenter → VAPI Server + wcpsvc; in the Supervisor → **VM Operator** (reconciles `VirtualMachine`, `VirtualMachineClass`, `VirtualMachineImage` CRDs).
- **VM Class** = reusable hardware spec (vCPU, memory, reservations). 16 default classes; **Guaranteed** vs **Best-effort**; custom classes via vSphere Client or **DCLI** (`namespacemanagement virtualmachineclasses ...`).
- **Deployment prerequisites** (admin does these): create namespace → assign storage policies → assign content libraries (VM images) → associate VM classes.
- **Deploy workflow** (DevOps, via kubectl):
  1. `kubectl get virtualmachineimages -n <ns>`
  2. `kubectl get virtualmachineclass -n <ns>`
  3. `kubectl get network -n <ns>`
  4. `kubectl get storageclass -n <ns>`
  5. Create cloud-init ConfigMap + VM spec YAML → `kubectl apply` → verify with `kubectl get virtualmachine -n <ns> -o wide`.
- **vGPU / PCI passthrough**: hosts need NVIDIA GRID GPUs in **Shared Direct** mode; content-library images must be **EFI** boot; install **vGPU Manager** on the ESX host and the **guest driver** in the VM. vGPU VMs power off when a host enters maintenance mode; DRS spreads them breadth-first; in 3-zone, VM and its PV must be in the **same zone**.

---

## Part 3 — The design workshop (this is how you're assessed)

The lab manual is a **design workshop** around a fictional customer: **Rainpole e-Commerce Services (RCS)**. You don't click through a live lab — you **make and justify design decisions**.

**The RCAR conceptual model** — every decision traces back to these:
- **Requirements** — Business (BR) and Technical/Non-functional (TR) needs (e.g., "modern dev platform," "99% SLA," "PCI DSS compliant," "restrict access by role").
- **Constraints (C)** — hard limits (e.g., "must use existing networking/storage").
- **Assumptions (A)** — things taken as true (e.g., "datacenters can add racks/hosts").
- **Risks (R)** — what could go wrong (e.g., "team lacks VKS skills," "can't add multi-site to an already single-site deployment").

**How a design decision is structured** (memorize this shape):

| Field | Example |
|---|---|
| Design Decision ID | DD-202 |
| Question | Which Supervisor deployment model? |
| Choices | NSX VPC / NSX Segment / VDS |
| **Decision** | Use NSX VPC networking |
| **Conceptual Model Reference** | BR02, BR04, TR03, C02, A01 |
| **Business Justification** | Supports isolation, multi-tenancy, segmentation across zones |
| **Design Qualities Affected** | Manageability, Availability, Performance |
| **Implications** | Team may lack NSX/VPC skills |

The workshop walks through these decision areas (mirroring the lecture modules):
1. **Single-site Supervisor deployment** (DD-201 to DD-205)
2. **Multi-site Supervisor deployment** (DD-301+) — introduces vSAN Stretched Cluster
3. **Single & zone-based Namespaces** (DD-401+)
4. **Services in Supervisor** (DD-601+) — base + extensible services
5. **Specific VM-Service VMs** (DD-801+)

**Exam tip:** For any scenario question, the winning answer is the one that (a) satisfies the stated Requirements, (b) respects the Constraints, and (c) has acceptable Implications/Risks. Always be ready to say *why*.

---

## Part 4 — High-yield facts to memorize

- Supervisor = a vSphere cluster with Kubernetes enabled; exposes a declarative K8s API from the hypervisor.
- **Spherelet** = kubelet on ESX; **CRX** = fast VM-based container runtime; **vSphere Pods** have their own Photon kernel and are **not** vMotion-compatible.
- Control plane: **1 VM (Simple)** or **3 VMs (HA)**. Sizes Tiny/Small/Medium/Large.
- Zones: **only 1 or 3** are supported. A zone = one vSphere cluster. **No cross-zone infra replication.**
- Networking stacks: **NSX VPC**, **NSX Segment**, **VDS**. Pick one.
- **Foundation LB = VDS-only, L4-only, bundled with vCenter.** **Avi = L4/L7, one per Supervisor, Controller + SEs + AKO.**
- **Simplified Supervisor** = 1 CP VM, 1 network, **VM Service only** (no Pods/Services); supports FLB One-Arm.
- Storage: **Principal vs Supplemental**; CNS + CSI + FCD + SPBM. **ReadWriteOnce yes, ReadWriteMany (dynamic) no** via CNS-CSI.
- Multi-zone storage needs a **topology-aware Zonal Storage Policy**, same vCenter, apps must self-replicate.
- **vSAN Stretched Cluster** = 2 data sites + witness; **vSAN storage clusters** = ESA-only disaggregated storage.
- vSphere Namespace ≠ Kubernetes namespace; namespace = resource pool + quota + RBAC + policies + libraries.
- Identity: **SSO (default)** or **external OIDC (Pinniped)**. RBAC: Role/RoleBinding (namespace) vs ClusterRole/ClusterRoleBinding (cluster-wide).
- VKS CNI: **Antrea (default)** or **Calico**. Multi-zone worker placement = **NodePool → FailureDomain**.
- Velero: Supervisor backup via **vCenter UI**; VKS via **CLI**; **S3-compatible** target. Backup methods: **CSI snapshot / File System (Kopia) / vSphere Plug-in (full every time, no CBT)**.
- Extensible services: **Harbor (needs Contour), Contour, External-DNS, MinIO/vDPP, Secret Store (OpenBao), ArgoCD (GitOps), cert-manager**.
- VM Service: **VM Operator** reconciles `VirtualMachine`/`VirtualMachineClass`/`VirtualMachineImage`; deploy from **OVF/ISO** with **cloud-init**; **VM Class** = hardware spec (Guaranteed/Best-effort).

---

## Part 5 — A 10-day study plan

- **Day 1–2:** Part 0, Part 1 (vocabulary), Module 1–2 concepts. Be able to draw: ESX → cluster → Supervisor → Namespace → workloads.
- **Day 3:** Module 2 networking + load balancers. Make your own table of NSX VPC vs NSX Segment vs VDS and Avi vs FLB.
- **Day 4:** Module 2 zone models + control plane availability.
- **Day 5:** Module 3 storage (CNS/CSI/SPBM, zonal policies, stretched clusters).
- **Day 6:** Module 4 namespaces + RBAC + identity providers.
- **Day 7:** Module 5 VKS architecture + Velero + extensible services.
- **Day 8:** Module 6 VM Service + vGPU.
- **Day 9:** Work through the RCS design workshop. For each DD, write your own decision + justification, then compare with the reasoning in the lab manual.
- **Day 10:** Review Part 4 (high-yield facts) and self-quiz below.

---

## Part 6 — Self-quiz (answer from memory)

1. What is the difference between a vSphere Namespace and a Kubernetes namespace?
2. Name the three Supervisor networking stacks and which load balancer(s) each supports.
3. When would you choose Foundation LB over Avi?
4. How many zones can a Supervisor use, and why must multi-zone apps replicate themselves?
5. What does a Simplified Supervisor support and NOT support?
6. Which persistent-volume access mode is NOT supported dynamically by CNS-CSI?
7. Where do you back up the Supervisor vs. a VKS cluster (UI vs CLI)?
8. Which Velero backup method has no changed-block tracking (always full)?
9. What must you install before Harbor, and why?
10. In a 3-zone namespace, how do you control where VKS worker nodes land?
11. What are the three custom resources the VM Operator reconciles?
12. Name the four parts of the RCAR conceptual model used to justify design decisions.

*(Answers are all in Parts 1–3 above.)*

---

### Source
Built from your course materials: **EDU-EN-VKSAD9-LEC-IE.pdf** (Lecture Manual) and **EDU-EN-VKSAD9-LAB-IE.pdf** (Workshop Manual), *vSphere Kubernetes Service: Advanced Design [V9.0]*, © Broadcom. Content was summarized and rephrased for study purposes.
