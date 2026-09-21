# VKSAD9 — Exam Method + Fully Worked Examples

This shows you *exactly* how to answer the design-workshop questions. Study the method once, then work the practice questions at the end. Pair this with **Diagram 4**.

---

## Part 1 — The method in one paragraph

You get a **customer story**. You extract its **RCAR** facts (Requirements, Constraints, Assumptions, Risks). For each question, you choose the option that **fits those facts** — not the fanciest option — and you **justify** it by pointing back to specific RCAR items. Then you note the **implication** (the downside you're accepting). That's the entire game.

---

## Part 2 — The customer (Rainpole e-Commerce Services / "RCS")

Here is the story your course uses, condensed:

- RCS is an e-commerce division modernizing its apps. They chose **VCF** as the platform. Their flagship modern app is **"opencart."**
- **Three datacenters:** San Jose (primary), Salt Lake City, Albany (NY).
- **Three environments:** Production, Development, DMZ. Development workloads recently **tripled**.
- Phase 1 built a **new Development environment** on VCF (done in a month). Later phases extend to Production and DMZ.
- Business wants **BCP/DR** (business continuity / disaster recovery) across all environments → **multi-site** capability.
- **Network:** MTU 9000; dark-fibre links; latency **25 ms San Jose↔Salt Lake**, **80 ms Albany↔others**.
- **Storage:** vSAN in San Jose and Salt Lake; 900+ TB.
- **Security:** L7 WAF, DMZ firewall, evaluating **distributed firewalling**; must be **PCI-DSS compliant**; must use **existing Active Directory**.
- **SLA:** 99% uptime.

### The RCAR facts (this is what you reason from)

**Requirements (needs):**
- BR: a modern, extensible, declarative platform (Kubernetes + VMs together); run cloud-native apps in Dev, later Production; high-throughput low-latency storage; expose some apps to the internet for field testing; multi-site HA later.
- TR: resilient during maintenance; centralized visibility/control; logically organized for delegation; predictable performance; **role-based access**; **backups** validated to SLA; logging/alerting; **99% SLA**; **PCI-DSS**; **reuse existing Active Directory**.

**Constraints (hard limits):**
- C01: must use **existing networking**.
- C02: must use **existing storage**.
- C03: only essential health/status KPIs on a dashboard for now.

**Assumptions (taken as true):**
- Datacenters can add racks/hosts, and have the resources/config ready for multi-site.

**Risks (what could go wrong):**
- R01: team may **lack VKS-on-VCF skills**.
- R02: storage reconfiguration needed to go multi-site.
- R03: **you can't bolt multi-site onto an already-built single-site** deployment.

Keep this list beside you — every answer cites it.

---

## Part 3 — Worked example #1 (the platform decision, DD-201)

**Question:** Should RCS keep its current architecture and use the vSphere Supervisor, or re-architect?

**How to think:** They already chose VCF (BR02: want a declarative hypervisor-level API). They must reuse existing storage (C02). They can add hosts (A01). Nothing suggests throwing it away. So the low-risk, requirement-satisfying answer is: **use the vSphere Supervisor.**

**Model answer:**

| Field | Answer |
|---|---|
| Decision ID | DD-201 |
| Decision | **Use the vSphere Supervisor** (keep the architecture) |
| Conceptual Model Reference | BR02, BR04, TR03, C02, A01 |
| Business Justification | Native integration with VCF; delivers the declarative Kubernetes+VM platform they asked for, reusing existing investment |
| Design Qualities Affected | Availability, Security, Performance |
| Implication (accepted downside) | The infra team **might lack VKS/VCF 9.x skills** (R01) — plan for training |

---

## Part 4 — Worked example #2 (networking model, DD-202)

**Question:** Which Supervisor deployment/networking model — NSX VPC, NSX Segment, or VDS?

**How to think:** RCS needs **isolation and multi-tenancy** across Dev/Prod/DMZ and is even evaluating **distributed firewalling** (that's an NSX strength). They want segmentation across zones and workloads. VDS is too basic for that; NSX **VPC** gives the strongest isolation and per-tenant private networks. So: **NSX VPC.**

**Model answer:**

| Field | Answer |
|---|---|
| Decision ID | DD-202 |
| Decision | **Use the deployment model with NSX VPC networking** |
| Conceptual Model Reference | BR02, BR04, TR03, C02, A01 |
| Business Justification | VPC networking supports isolation, multi-tenancy, and network/storage segmentation across zones and workloads — matches their multi-environment, PCI-driven needs |
| Design Qualities Affected | Manageability, Availability, Performance |
| Implication | Team may **lack NSX/VPC skills** on VCF 9.x (R01) — added complexity and training need |

---

## Part 5 — Worked example #3 (multi-site availability)

**Question:** RCS wants BCP/DR across sites. Which storage/availability design fits, given 25 ms San Jose↔Salt Lake and existing vSAN in both?

**How to think:**
- They need site-level protection with **automatic failover** and minimal data loss → **vSAN Stretched Cluster** (mirrors data across two sites + a witness).
- The **25 ms** link works for stretched vSAN between San Jose and Salt Lake; **Albany at 80 ms is too far** for the same stretched cluster, so it's not a stretched partner.
- Risk R03 matters: **multi-site can't be retrofitted onto a single-site build** — so this must be planned **before** locking in the single-site design.

**Model answer:**

| Field | Answer |
|---|---|
| Decision | **Use a vSAN Stretched Cluster between San Jose and Salt Lake, with a witness in a third location** |
| Conceptual Model Reference | BR07 (multi-site HA), TR01 (resilience), TR06 (backups), A02/A03 (sites ready) |
| Business Justification | Mirrors data across two availability zones with automatic failover → meets BCP/DR + 99% SLA |
| Design Qualities Affected | Availability, Recoverability, Performance |
| Implication | Requires storage reconfiguration (R02); Albany (80 ms) cannot join the stretched cluster; must be designed up front (R03) |

---

## Part 6 — Worked example #4 (access control)

**Question:** How should RCS control who can access namespaces, given they must reuse Active Directory and be PCI-compliant?

**How to think:** TR05 (role-based access), TR10 (reuse existing AD), TR09 (PCI). The Supervisor's default identity provider is **vCenter SSO**, which integrates with **AD/LDAP**. Use **RBAC** (Roles/RoleBindings per namespace) to enforce least privilege for PCI.

**Model answer:**

| Field | Answer |
|---|---|
| Decision | **Use vCenter Single Sign-On integrated with existing Active Directory + namespace RBAC (Roles/RoleBindings)** |
| Conceptual Model Reference | TR05, TR09, TR10 |
| Business Justification | Reuses existing AD, gives role-based least-privilege access required for PCI-DSS |
| Design Qualities Affected | Security, Manageability |
| Implication | Must carefully map AD groups to namespace roles; audit regularly for compliance |

---

## Part 7 — Practice questions (try before peeking)

Write your own DD table (Decision / References / Justification / Qualities / Implication) for each, then check the hint.

1. **RCS wants developers to self-service Kubernetes clusters in the Development environment.** Which service enables that, and what CNI is the default?
   *Hint: VKS provisions the clusters; default CNI is Antrea.*

2. **RCS has legacy apps that can't be containerized yet but wants one workflow.** What do you use?
   *Hint: VM Service — declarative VMs via the same Kubernetes-style API.*

3. **RCS needs a private, secured image registry with vulnerability scanning.** What service, and what must be installed first?
   *Hint: Harbor; install Contour (ingress) first.*

4. **RCS must back up workloads to their existing S3-compatible object store, with the most storage-efficient method for CNS block volumes.** Which Velero method?
   *Hint: CSI Snapshot (supports incremental + dedup + compression; crash-consistent).*

5. **RCS wants the quickest possible starting point on one cluster, only VMs for now, and to grow later.** Which deployment model?
   *Hint: Simplified Supervisor (1 control-plane VM, single network, VM Service only; scalable later).*

6. **RCS wants the Supervisor control plane to survive one control-plane failure.** What do you choose?
   *Hint: High Availability Control Plane = 3 control-plane VMs.*

7. **They spread a stateful app across 3 zones and expect VMware to replicate the data between zones.** What's wrong with that expectation?
   *Hint: No cross-zone infra replication — the app itself must replicate (cloud-native).*

8. **Which load balancer can RCS NOT use with NSX, and why?**
   *Hint: Foundation LB — it's VDS-only and L4-only.*

---

## Part 8 — The 6 sentences that win the exam

1. Always start from the **customer's RCAR facts**, never from "what's best in theory."
2. The right answer **satisfies Requirements, respects Constraints, and leaves acceptable Risks.**
3. **NSX** when you need isolation/multi-tenancy/firewalling; **VDS** when simple.
4. **Foundation LB = VDS-only, L4-only, free;** **Avi = advanced L4/L7.**
5. **1 or 3 zones only; no cross-zone replication** — cloud-native apps self-replicate; **stretched cluster** protects at the storage layer.
6. Every decision needs a **justification (why)** and an **implication (the trade-off you accept).**

---

### Source
Summarized and rephrased from your course materials: **EDU-EN-VKSAD9-LEC-IE.pdf** and **EDU-EN-VKSAD9-LAB-IE.pdf**, *vSphere Kubernetes Service: Advanced Design [V9.0]*, © Broadcom. Model answers follow the reasoning shown in the lab workshop; your instructor's answers may vary.
