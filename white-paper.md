---

copyright:
  years: 2026

lastupdated: "2026-10-01"

keywords: OpenShift Virtualization, VMware, day-2 operations, live migration, virtual machine, KVM, vSphere

subcollection: vm-ops-openshift-virtualization-d2

---

{{site.data.keyword.attribute-definition-list}}

# Day-2 VM operations on OpenShift Virtualization Service
{: #white-paper}

A practical guide for VMware operations teams making the transition to Red Hat OpenShift Virtualization Service.
{: shortdesc}

## Introduction
{: #introduction}

This white paper covers common virtual machine operational tasks in [Red Hat OpenShift Virtualization Service](/docs/openshift?topic=openshift-rovs-overview), with each use case presented alongside the equivalent approach in VMware. It is intended as a practical reference for day-to-day VM operations, not a migration guide or platform overview.

The similar capabilities you depend on today — live migration, DRS-style placement, backup and recovery, resource oversubscription, logging and monitoring — all exist in OpenShift Virtualization Service. The activity may look different. You may run a YAML file instead of a script. You may use a pull request instead of a change ticket. But you will continue to do your job, and this paper gives you the quick guideposts to get there.

This paper covers seven use cases that come up in every client conversation about OpenShift Virtualization Service: live migration and placement, backup and disaster recovery, resource management, observability, VM lifecycle (provisioning, power operations, and decommission), networking, and storage. Each section follows the same pattern: the typical approach in VMware, the approach in an OpenShift Virtualization Service cluster, and then distinct differences if there are any.

## The paradigm shift: From imperative to declarative
{: #paradigm-shift}

BBefore getting into the use cases, one conceptual shift is worth naming. VMware operations are largely imperative and component focussed: you log into vCenter, you click, and the platform executes your command on a specific VM – including DRS rebalances, vMotion moves, vSphere Data Protection backs it up. You flick a switch and the platform does the work.

[OpenShift Virtualization](/docs/virtualization-solutions) introduces a declarative model, inherited from Kubernetes. Instead of executing commands on individual components (such as virtual machines), you declare the desired state of the product, service or environment. You describe what the final outcome should look like – how many CPUs the service, product or feature needs, the memory required, its protection policies- and the platform’s control plane continuously works to achieve and maintain the aggregate (desired) state, automatically handling the virtual machines.

While Kubernetes orchestrates the platform, the underlying hypervisor is KVM – the same cloud-native technology used by IBM Cloud, AWS, and Google Cloud. KVM is responsible for running the virtual machines above the hardware.

## How OpenShift Virtualization architecture actually works
{: #os-v-architecture}

Before getting into the use cases, it is worth spending a moment on why OpenShift VMs are architecturally different from both OpenShift containers and VMware VMs — because this distinction directly explains many of the operational patterns described in this paper, including why backup and DR work the way they do.

In a standard Kubernetes deployment, a containerised workload is stateless and image-driven. The container image is a small, layered bundle (typically tens to hundreds of megabytes) stored in an image registry. If a node fails, Kubernetes simply pulls the image onto another node and restarts the container. No disk state needs to ffollow it — the image is the definition and the runtime state.

A VMware VM is fundamentally different: it is a large, stateful disk image. A VMDK file typically ranges from gigabytes to hundreds of gigabytes and captures both the OS installation and all accumulated disk state. vSphere manages placement, migration, and recovery of these disk images through a centralised management plane (vCenter), shared storage (datastores), and purpose-built protocols (vMotion for live migration, vSphere Replication for DR).

An OpenShift Virtualization VM sits between these two worlds. Like a container, it runs inside Kubernetes as a pod (the virt-launcher pod) and is governed by Kubernetes scheduling and declarative policies. Unlike a container, it carries real stateful disk data: the VM's disk is a PersistentVolumeClaim (PVC) — a Kubernetes storage object that holds the full OS and data volumes, ranging from gigabytes to terabytes. This is why:

- A VM cannot simply be "pulled from a registry" like a container image. The disk data must be present on storage accessible to the node where the VM runs.
- Live migration requires ReadWriteMany (RWX) storage so the PVC can be accessed from the source and destination node simultaneously — exactly the same constraint as vMotion requiring shared storage.
- Backup requires both the VM definition (the Kubernetes object — lightweight YAML) and the disk data (the PVC — potentially very large). OADP/Velero handles both, but the PVC backup is the operationally significant part, not the definition.
- DR and recovery require that disk data be either replicated to the target site or restorable from backup — re-creating only the VM definition (the YAML) is not sufficient.

This architecture also explains why simple Image Registry approaches alone are insufficient for VM workloads: an image registry holds small, stateless images suitable for container restarts. It does not replicate or manage large, stateful block volumes. The disk state of a VM requires dedicated storage replication, backup tooling (such as OADP), or migration tooling (such as MTV) — not a container image registry.

With this foundation in place, the use cases below will make more sense — particularly the choices made in backup, DR, and storage.

## Use case 1: Live migration and VM placement
{: #live-migration-placement}

### In VMware
{: #uc1-vmware}

The purpose of this use case is to balance resource utilisation — compute and memory — across the underlying infrastructure nodes. This same mechanism also serves a second important function: moving VMs off a node so that maintenance can be performed on it safely, without interrupting running workloads.

vMotion moves a running VM from one host to another with no service interruption. DRS automates placement and rebalancing across the cluster — admins turn it on and it handles the rest. Maintenance mode drains a host by migrating its VMs before work begins.

### In OpenShift Virtualization Service
{: #uc1-ocpv}

Live migration works the same way. A running VM is moved from one node to another with no downtime. In OpenShift Virtualization, this is a `VirtualMachineInstanceMigration` — you can trigger it manually from the web console (**Actions** > **Migrate**) or let the platform trigger it automatically during node maintenance. The `evictionStrategy` field in the VM definition controls whether a VM migrates or shuts down when its node is drained.

Automated rebalancing is provided by the [OpenShift Descheduler Operator](https://developers.redhat.com/blog/2024/12/19/load-aware-rebalancing-openshift-virtualization){: external}. Install it from OperatorHub, configure the `KubeVirtRelieveAndMigrate` or `LongLifecycle` profile, and the platform periodically evaluates VM placement and migrates workloads to better-suited nodes — exactly the DRS rebalancing behaviour VMware admins rely on. In OpenShift Virtualization Service you can define custom policies that may differ from what might be provided in other environments out of the box.

### What is different
{: #uc1-differences}

Live migration requires storage that supports simultaneous access from multiple nodes (ReadWriteMany access mode). VMs on ReadWriteOnce storage cannot be live-migrated and must be shut down for node maintenance. Confirm storage class capabilities before relying on live migration for planned maintenance.

CPU and memory resizes are also handled through live migration rather than a power cycle. Changing a VM's instance type (equivalent to a VM size profile) triggers a migration to a new node where the new allocation takes effect — the VM keeps running throughout. This requires free capacity on another node, so capacity headroom planning remains just as important as it was in vSphere.

Placement rules — what VMware expressed as DRS VM-Host Affinity Rules — are expressed in OpenShift Virtualization Service through node labels, node affinity, and pod anti-affinity in the VM specification. The equivalent of dedicated host groups and anti-affinity rules is fully supported. Hard constraints (`required`) should be used sparingly: an over-constrained VM cannot be scheduled or migrated when its target nodes are unavailable.

### Architectural consideration: VM availability and node failure behavior
{: #uc-1-arch-considerations}

One architectural consideration when running virtual machines on OpenShift Virtualization is that VM availability and recovery behaviour are governed by Kubernetes constructs and policies rather than the hypervisor-centric mechanisms commonly found in traditional virtualization platforms. By default, a VM is not automatically live-migrated to another worker node following a node failure. Organizations requiring this level of workload mobility and resilience can enable additional OpenShift Virtualization capabilities, operators, and scheduling policies to support advanced VM lifecycle and availability requirements. This approach aligns with Kubernetes design principles, where workload placement, recovery, and availability are managed through declarative policies and platform services rather than being inherently provided by the underlying infrastructure.

The evictionStrategy field is the primary control point:

- evictionStrategy: LiveMigrate — the VM is live-migrated away when its node is cordoned or drained (planned maintenance). This is the recommended setting for production VMs on RWX storage.
- evictionStrategy: LiveMigrateIfPossible — live migration is attempted first; if not possible (for example, RWO storage), the VM is shut down and restarted on another node.
- evictionStrategy: None — the VM is not automatically moved; it must be manually restarted after a node failure.

For unplanned node failures (hardware crash, kernel panic), the platform's response depends on which additional capabilities are enabled:

1. RunStrategy: Always — if a node goes down and the VM's pod terminates, Kubernetes will attempt to reschedule the virt-launcher pod on a healthy node. For this to succeed, the node must be confirmed as unreachable (typically after the node-monitor-grace-period, default 40 seconds, plus taint-manager eviction timeout). On RWX storage the VM restarts on another node automatically; on RWO storage the PVC must be released first.

2. Node Health Check Operator with Self Node Remediation (SNR) — this operator combination actively monitors node health and performs automated remediation (fencing or reboot) of unhealthy nodes, enabling faster and more reliable VM restart on a healthy node after an unplanned failure. This is the recommended approach for environments that require HA behaviour equivalent to VMware HA. See the [documentation and installation steps](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/nodes/remediating-nodes-with-remediation-operators){: external}.

3. OpenShift Virtualization High Availability configuration guide — for the definitive reference on configuring live migration policies, eviction strategies, and node health remediation together see [live-migration documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/live-migration){: external}.

Summary: to configure OpenShift Virtualization to live-migrate VMs on restart (planned or unplanned), set evictionStrategy: LiveMigrate in each VM definition, ensure RWX storage, and deploy the Node Health Check Operator with Self Node Remediation for unplanned failure coverage. This combination provides HA behaviour that is directly comparable to VMware HA plus vMotion.


## Use case 2: Backup and disaster recovery
{: #backup-dr}

### In VMware
{: #uc2-vmware}

VM snapshots provide point-in-time recovery. Enterprise backup uses vSphere Data Protection or a third-party agent (Veeam, Commvault, and others) to [back up](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/backup-and-restore){: external} VMs to an external target. DR relies on replication, backup restoration, or vSphere Replication to a secondary site, with manual or scripted failover and failback.

### In OpenShift Virtualization Service
{: #uc2-ocpv}

Snapshots work in the same way conceptually. A `VirtualMachineSnapshot` captures the VM definition and disk state at a point in time and can be restored from the web console (**Virtualization** > **VirtualMachines** > **Snapshots** tab). Snapshot support requires the storage class to support CSI snapshots — confirm this with the storage team before relying on snapshots in production.

Enterprise backup uses OpenShift API for Data Protection (OADP), the Red Hat-supported backup framework based on Velero. OADP backs up Kubernetes resources (including the complete VM definition) and persistent disk data to an S3-compatible external store. It replaces vSphere Data Protection as the platform-native backup mechanism.

For disaster recovery, it is important to separate two distinct concerns: (1) recovering the platform configuration — namespaces, network definitions, RBAC, VM specifications and (2) recovering the VM disk data.

The platform configuration (the Kubernetes objects that define your VMs and their surrounding infrastructure) is entirely text-based YAML. These objects are typically small and can be stored in a Git repository at no meaningful cost. If a cluster or namespace is lost, these definitions can be reapplied to a new or recovery cluster directly — with oc apply, with a GitOps tool such as ArgoCD, or even manually. ArgoCD is not a requirement for this; it is mentioned here because it automates continuous reconciliation (it keeps the cluster matching what is in Git), which is valuable in large or regulated environments. If your team is not already using ArgoCD, you do not need to build that infrastructure to benefit from Git-stored VM definitions — a simple oc apply from a Git-cloned directory achieves the same recovery outcome.

The VM disk data (the PVCs) is the operationally significant part of a DR strategy. This is not text — it is block storage data ranging from gigabytes to terabytes. OADP backs up both the VM definition and the PVC data to an S3-compatible external store. On recovery, OADP restores the PVC data first, then reattaches the VM definition. This is the equivalent of vSphere Data Protection backing up VMDKs to an external target and restoring them after a DR event.

For teams that are not yet using Git-based workflows, the minimum viable DR posture for OpenShift Virtualization is: OADP configured to back up VMs (definitions + disks) to an external S3 store, with tested restore runbooks. Git storage of VM definitions is a valuable addition — it reduces recovery time for platform configuration and supports auditability — but it is a separate concern from disk data recovery and can be adopted incrementally.

For continuous replication of VM disk data — the equivalent of vSphere Replication — OpenShift Virtualization uses the [Migration Toolkit for Virtualization](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/){: external} (MTV).

### Advanced Cluster Management (ACM) and multi-cluster DR
{: #acm-dr}

Red Hat Advanced Cluster Management ([ACM](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.17){: external}) for Kubernetes is the platform-level capability that sits above individual clusters and enables coordinated disaster recovery across sites. It is worth addressing directly, because it answers a question that OADP alone cannot: "Can I do live, policy-driven DR failover without manually restoring backups?"

The short answer is: OADP alone is a backup-and-restore tool — it is not live replication. The distinction matters:

- OADP (backup and restore): Creates scheduled backups of VMs (definition + PVC data) to an S3-compatible store. Recovery is a restore operation: you initiate an OADP restore, the PVC data is pulled from S3 to the recovery cluster, and the VM is restarted. Recovery time depends on the size of the PVC data and the speed of the S3 restore. This is the equivalent of vSphere Data Protection with restore to a secondary site — operational, reliable, but not zero-RPO or automated failover.

- ACM + OpenShift DR (live replication and orchestrated failover): Red Hat Advanced Cluster Management, combined with the OpenShift DR (ODF DR / Regional DR) capabilities of OpenShift Data Foundation (ODF), provides synchronous or asynchronous storage replication between clusters at different sites, plus ACM-orchestrated failover policies. This is the equivalent of vSphere Replication plus Site Recovery Manager — live replication with automated or semi-automated failover. ACM manages the workload placement policies across clusters and can relocate or failover VM workloads to a secondary cluster with a single policy action rather than a manual restore sequence.

For organizations running OpenShift Virtualization on IBM Cloud, the minimum viable DR posture is OADP to an IBM Cloud Object Storage (COS) bucket. For workloads requiring near-zero RPO and automated failover, ACM with [ODF Regional DR](https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.18/html/configuring_openshift_data_foundation_disaster_recovery_for_openshift_workloads/index){: external} is the correct solution — it requires OpenShift Data Foundation on both clusters and an ACM hub, which represents additional platform investment.


### What is different
{: #uc2-differences}

A snapshot in OpenShift Virtualization Service, like a vSphere snapshot, is not a backup. It lives in the same failure domain as the VM. For application-consistent snapshots of running VMs, the QEMU guest agent must be installed in the guest OS — without it, snapshots are crash-consistent only. This mirrors the vSphere requirement for VMware Tools for quiesced snapshots. Install the QEMU guest agent in every production VM as a Day 1 task.

### Snapshot and application consistency
{: #snapshot-consistency}

A [VM snapshot](https://redhatquickcourses.github.io/ocp-virt-cookbook/ocp-virt-cookbook/1/vm-lifecycle/vm-snapshots.html){: external} should not be treated as a substitute for a backup. Snapshots are intended primarily as point-in-time restore or rollback mechanisms and may depend on the same underlying storage as the VM. A separate backup strategy should therefore be used where protection from storage failure or loss is required.
{: important}

For a running VM in OpenShift Virtualization, snapshot consistency depends on guest OS integration:

- **Without the [QEMU guest agent](https://redhatquickcourses.github.io/ocp-virt-cookbook/ocp-virt-cookbook/1/agentic-vm-management/ai-vm-snapshot-management.html){: external}:** the guest filesystem cannot be quiesced and OpenShift Virtualization takes a best-effort snapshot. This is commonly described as crash-consistent, similar to recovering the VM after an unexpected shutdown.
- **With the QEMU guest agent installed and running:** OpenShift Virtualization attempts to quiesce the guest filesystem by freezing it before the snapshot and thawing it afterwards, allowing in-flight I/O to be written to disk and providing a more [consistent snapshot](https://docs.redhat.com/en/documentation/openshift_container_platform/4.14/html/virtualization/backup-and-restore){: external}.

Install and enable the [QEMU guest agent](https://docs.redhat.com/en/documentation/openshift_container_platform/4.14/html/virtualization/backup-and-restore){: external} on production VMs as a Day 1 operational requirement, particularly where online snapshots are required. For applications requiring true application-level consistency, validate the application's own quiescing and backup requirements rather than relying solely on filesystem consistency. The QEMU guest agent alone provides filesystem-consistent (quiesced) snapshots; application-level consistency requires additional application-specific coordination.
{: note}

## Use case 3: Resource management — oversubscription, quotas, and capacity
{: #resource-management}

### In VMware
{: #uc3-vmware}

Resource pools define CPU and memory reservations, limits, and shares for groups of VMs. CPU oversubscription is common — typical ratios of 2:1 to 4:1 vCPU-to-pCPU depending on the workload mix. Memory oversubscription is used cautiously; most experienced VMware admins follow the rule: do not oversubscribe memory on production workloads unless you want to kill performance.

### In OpenShift Virtualization Service
{: #uc3-ocpv}

The same rules apply, and the same instincts are correct. CPU oversubscription is supported and works in the same way — vCPUs are time-sliced against physical CPUs. The CPU Allocation Ratio in OpenShift Virtualization Service maps directly to what VMware admins configure in resource pool shares. A 4:1 vCPU-to-pCPU ratio works fine for mixed, bursty workloads. A 10:1 ratio works for dev and test. Production compute-bound workloads should use guaranteed (pinned) CPUs. For the full configuration details on setting CPU and memory allocation ratios, dedicated resource pinning, and overcommit policies, refer to the official [Red Hat documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/virtualization/virtual-machines#virt-dedicated-resources-vm){: external}.

Memory is not oversubscribed by default in OpenShift Virtualization Service — every VM's memory request is fully reserved on the node, giving guaranteed QoS and predictable scheduling. This is the safest default and mirrors best practice in VMware environments. Memory oversubscription can be enabled using the `memoryOvercommitPercentage` setting in the `HyperConverged` resource, backed by node swap managed by the `wasp-agent`. The rule of thumb remains unchanged: oversubscribe memory in non-production if you need density, but expect performance degradation at high utilisation. Don't do it in production unless you have thoroughly tested the workload behaviour.

Namespace `ResourceQuota` and `LimitRange` replace VMware resource pools as the mechanism for controlling consumption. Quotas are applied at the namespace (project) level and control the total CPU, memory, and storage that a group of VMs can consume. These are version-controlled alongside VM definitions in Git.

### What is different
{: #uc3-differences}

OpenShift Virtualization Service introduces instance types as a standardised approach to VM sizing. Instead of defining CPU and memory independently for every VM, administrators can select from predefined, reusable sizing profiles appropriate to the workload. Instance types define CPU and memory and can include additional VM characteristics, helping simplify and standardise repeated [VM creation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/creating-a-virtual-machine){: external}.

Use the predefined instance-type catalogue as the default approach for standard workloads, providing consistent and repeatable VM sizing. Where workload requirements cannot be met by the available profiles, custom instance types can be created rather than forcing every VM into a standard size.
{: note}

CPU and memory defined by an instance type cannot be overridden directly for an individual VM using that instance type. Select an alternative instance type or create an appropriate custom instance type when different resource requirements are needed.
{: important}

## Use case 4: Logging, monitoring, and troubleshooting
{: #observability}

### In VMware
{: #uc4-vmware}

vCenter Tasks and Events shows the history of operations and alerts. Performance charts provide CPU, memory, disk, and network graphs per VM. The vSphere VM console gives direct guest OS access. Logs are available from the host and guest, typically shipped to an external SIEM or log aggregator.

### In OpenShift Virtualization Service
{: #uc4-ocpv}

The OpenShift Web Console is the primary operational interface — think of it as your new vCenter. The Virtualization section provides a VM-centric view with status, power controls, events, metrics, snapshots, and direct console access in one place. It provides the full set of capabilities VMware admins expect.

Prometheus-based monitoring is built in with no additional setup. Navigate to **Monitoring** > **Dashboards** in the console and virtualization dashboards are ready to use — CPU, memory, disk I/O, network throughput, migration status, all pre-built. Alerting rules for common virtualization failures (VM stuck in pending, live migration failures, guest agent missing, storage capacity) are included by default and route through Alertmanager. This capability is included in OpenShift Virtualization Service at no extra cost and requires no additional setup.

For guest console access: use the OpenShift web console (**Virtualization** > **VirtualMachines** > select VM > **Console** tab) or the `virtctl console` command. This is equivalent to the vCenter VMRC console — direct serial console access to the guest OS regardless of network state, useful for boot failures and network misconfigurations.

### What is different
{: #uc4-differences}

Troubleshooting follows a layered model. When a VM is not behaving as expected, check: (1) VM phase and events in the console, equivalent to vCenter Tasks & Events; (2) the serial console for guest OS state; (3) node conditions and resource utilisation; (4) PVC and storage health; (5) `virt-launcher` pod logs for hypervisor-level diagnostics. The same information is there — it is accessed through different paths. Most issues VMware admins encounter day-to-day are diagnosed through the first two steps in the console.

## Use case 5: VM lifecycle — provisioning, power operations, and decommission
{: #vm-lifecycle}

The purpose of this use case is to cover the full operational lifecycle of a virtual machine: creating it from a known-good template or image, controlling its power state during normal operations, and safely decommissioning it when it is no longer required. Getting each of these steps right — particularly decommission — is essential for maintaining a clean, auditable, and cost-controlled environment.

### In VMware
{: #uc5-vmware}

VM provisioning is template-driven. Admins deploy from VM templates stored in the content library, selecting guest OS, hardware profile, and network/storage placement through the deploy wizard. Customisation specifications (sysprep for Windows, cloud-init for Linux) handle post-deploy guest configuration. Cloning an existing VM produces a direct copy with all its disks.

Power operations — power on, power off, reset, suspend — are right-click actions in vCenter or driven by scripts via PowerCLI. VMware HA automatically restarts VMs on a healthy host if their running host fails. Guest shutdown is issued through VMware Tools; without Tools, a power-off is a hard cut.

Decommission is a guided workflow in vCenter: remove the VM from inventory or delete it from disk, with an explicit prompt to delete associated virtual disks. IPAM and CMDB cleanup are typically handled by separate runbook steps outside vCenter.

### In OpenShift Virtualization Service
{: #uc5-ocpv}

Provisioning is catalog-based, directly analogous to VMware content libraries and VM templates. Instance types define compute sizing. Preferences define OS-specific configuration — firmware, drivers, clock. Boot sources are managed OS images (RHEL 9, Windows Server, and others) that are auto-updated by the platform. To deploy a VM: **Virtualization** > **VirtualMachines** > **Create VirtualMachine** > **From template**. Select instance type and boot source, fill in the name and namespace, and deploy. Guest customisation uses cloud-init, specified declaratively in the VM definition — equivalent to a vSphere customisation specification.

Power operations — start, stop, restart, pause — are available from the **Actions** menu in the console or via the `virtctl` CLI (`virtctl start`, `virtctl stop`, `virtctl restart`). The `runStrategy: Always` field in the VM definition provides automatic restart on failure, equivalent to VMware HA. Cloning is supported through **Actions** > **Clone Virtual Machine**; confirm storage class CSI clone support before relying on this at scale.

Decommission follows the same intent as vSphere but requires more explicit steps: delete the `VirtualMachine` object via the console (**Actions** > **Delete**) or `oc delete vm <name> -n <namespace>`, then separately delete the associated PersistentVolumeClaims. Release the IP address from the IPAM pool, remove DNS records, close the CMDB record, and confirm backup retention policy. These steps should be codified in a decommission runbook — none of them happen automatically.

### What is different
{: #uc5-differences}

The most operationally significant difference is decommission: deleting a `VirtualMachine` in OpenShift Virtualization Service does not automatically delete its storage disks (PersistentVolumeClaims). vSphere prompts for disk deletion at removal time; OpenShift Virtualization Service does not. Orphaned PVCs consume namespace quota silently and will not surface as a problem until capacity is exhausted. Add an explicit PVC cleanup step to every decommission runbook.

Guest customisation at provisioning time is handled by cloud-init rather than vSphere customisation specifications. The outcome is the same — hostname, network config, user accounts — but the configuration is expressed in a cloud-init `user-data` block within the VM YAML definition, making it version-controllable alongside the VM specification itself. For Windows VMs, Sysprep is supported via the `sysprepRef` field in the VM definition.

Power-off behaviour differs: without the QEMU guest agent installed, a stop operation in OpenShift Virtualization Service is equivalent to pulling the power — the guest OS does not receive a graceful shutdown signal. Install the QEMU guest agent in every production VM to enable clean shutdowns and application-consistent snapshots.

## Use case 6: Networking
{: #networking}

### In VMware
{: #uc6-vmware}

VM networking in vSphere revolves around port groups on vSwitches or distributed vSwitches. Each VM has one or more network adapters connected to a port group, which maps to a VLAN. Network policies, traffic shaping, security rules, and teaming are configured on the switch and inherited by connected VMs. Micro-segmentation and NSX provide advanced isolation, though in practice most environments rely on VLAN-backed port groups for segmentation.

### In OpenShift Virtualization Service
{: #uc6-ocpv}

Every VM in OpenShift Virtualization Service gets a default network interface on the pod network — a fully routed cluster network with built-in DNS and service discovery. For VMs that need attachment to a specific VLAN, an external network, or a flat Layer 2 segment (the equivalent of a vSphere port group), OpenShift Virtualization Service uses [NetworkAttachmentDefinitions](https://developers.redhat.com/articles/2024/12/19/how-configure-network-attachment-definitions){: external} (NADs). A NAD defines the network type, bridge, VLAN, and IP address management (IPAM) configuration. VMs reference NADs in their definition to attach a secondary interface — exactly the same outcome as connecting a VM to a vSphere port group, expressed declaratively.

[Network policies](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/virtualization/networking){: external} in OpenShift Virtualization Service control traffic between VMs and pods at the namespace level — equivalent to micro-segmentation. For VLAN-backed secondary networks, Multus CNI manages multi-network attachment, and Whereabouts or DHCP handles IP address management. On IBM Cloud OpenShift Virtualization Service specifically, the underlying infrastructure networking (VLANs, bonds, uplinks) is pre-configured; the operational focus is on defining NADs for workload-specific attachments and confirming IP allocation is managed correctly.

### What is different
{: #uc6-differences}

The key operational difference is that network attachments are Kubernetes resources, not infrastructure objects configured in a separate management plane. A NAD is a YAML definition — it can be version-controlled in Git, reviewed in a pull request, and applied consistently across environments. There is no separate distributed switch console; network configuration lives alongside VM configuration. When a VM is decommissioned, the IP address must be explicitly released — OpenShift Virtualization Service does not automatically return IPs to an IPAM pool the way vCenter integrates with IPAM on port-group removal.

## Use case 7: Storage
{: #storage}

### In VMware
{: #uc7-vmware}

VM storage in vSphere lives on datastores — NFS, VMFS on block storage, or vSAN. VMs have virtual disks (VMDK files) associated with a datastore. Storage policies control performance tiers, replication, and encryption. Admins add, expand, or migrate disks through vCenter, and datastore capacity is managed at the host cluster level.

### In OpenShift Virtualization Service
{: #uc7-ocpv}

VM disks in OpenShift Virtualization Service are PersistentVolumeClaims (PVCs) — Kubernetes storage objects backed by whatever storage class the cluster administrator has provisioned. On IBM Cloud OpenShift Virtualization Service, storage classes map to IBM Cloud block storage tiers (10 IOPs/GB, 5 IOPs/GB, and so on), directly analogous to vSphere storage policies. Adding a disk to a VM means adding a new PVC and attaching it to the VM definition — from the web console, this is done through **Virtualization** > **VM** > **Disks** > **Add Disk**, the same flow as adding a virtual disk in vCenter.

Disk expansion is supported on storage classes that allow volume expansion — equivalent to expanding a VMDK. DataVolumes manage image imports and cloning: importing a custom OS image from an HTTP URL or an existing PVC clone is a single YAML definition (or a console action), directly analogous to deploying from a vSphere datastore ISO or cloning a template disk. Golden image management — maintaining a controlled, patched base image for VM provisioning — uses DataSources, which auto-update from Red Hat's image registry and serve as the boot source for catalog-based VM deployment.

### What is different
{: #uc7-differences}

The storage access mode is the most operationally significant difference. ReadWriteMany (RWX) access mode allows a PVC to be mounted by multiple nodes simultaneously and is required for live migration to work without copying the disk. ReadWriteOnce (RWO) is the default for most block storage classes and limits a PVC to a single node — VMs on RWO storage cannot be live-migrated and must be shut down for node maintenance. Confirm RWX support with the IBM Cloud storage team before deploying production VMs that require live migration. The second difference: as noted in use case 2, deleting a VM does not delete its PVCs — storage cleanup is always a manual, explicit step.

## Quick reference: VMware to OpenShift Virtualization Service
{: #quick-reference}

The following table summarizes the most common Day-2 tasks and their OpenShift Virtualization Service equivalents. The intent is not exhaustive documentation — it is a fast lookup for VMware admins encountering a task for the first time on the new platform.

| Day-2 task | VMware (vSphere) | OpenShift Virtualization Service |
| ---------- | ---------------- | -------------------------------- |
| Provision VM | Deploy from template / content library | **Virtualization** > **Create VM** > **From template**. Select instance type + boot source. |
| Start / Stop / Restart | Right-click > Power On/Off in vCenter | Console **Actions** menu or: `virtctl start/stop/restart <vm> -n <ns>` |
| Resize CPU / Memory | Edit Settings > Hardware, then power cycle | Change instance type binding. Live migration applies the change — no power cycle needed. |
| VM console access | vCenter VMRC console | **Console** tab in web UI, or: `virtctl console <vm> -n <ns>` |
| Live migration | vMotion (manual or DRS) | Console: **Actions** > **Migrate**. CLI: `virtctl migrate <vm> -n <ns>`. Requires RWX storage. |
| DRS / automated rebalancing | Enable DRS on cluster — runs automatically | Install Descheduler Operator, configure `KubeVirtRelieveAndMigrate` profile. Policy-driven. |
| Node maintenance | Enter Maintenance Mode | `oc adm drain <node>`. VMs with `evictionStrategy: LiveMigrate` auto-migrate. |
| Take snapshot | VM > Snapshots > Take Snapshot | **Virtualization** > **VM** > **Snapshots** tab > **Take Snapshot**. Requires CSI snapshot + QEMU guest agent. |
| Enterprise backup | vSphere Data Protection / Veeam / Commvault | OADP (Velero-based). Backs up VM definition + disks to S3. Third-party agents also supported. |
| View events / alerts | vCenter Tasks & Events tab | **Virtualization** > **VM** > **Events** tab. **Monitoring** > **Alerting** for platform-wide alerts. |
| Performance metrics | vCenter Performance charts | **Monitoring** > **Dashboards**. Prometheus built-in, no setup required. |
| Decommission VM | Remove from Inventory / Delete (prompts for disk delete) | Delete VirtualMachine via console or `oc delete vm`. PVCs NOT auto-deleted — clean up manually. |
| Resource pools / quotas | Resource Pools with reservations, limits, shares | Namespace `ResourceQuota` + `LimitRange`. Instance types enforce per-VM sizing. |
| Attach VM to VLAN / network | Connect NIC to vSphere port group (VLAN-backed) | Create NetworkAttachmentDefinition (NAD) for the VLAN; reference it in the VM definition. |
| IP address management | DHCP or static via vCenter / IPAM integration | Whereabouts IPAM or DHCP via NAD config. Explicit IP release required on VM decommission. |
| Add a disk to a VM | Edit Settings > Add Hard Disk in vCenter | **Virtualization** > **VM** > **Disks** > **Add Disk**. Creates a new PVC and attaches it. |
| Expand a disk | Increase VMDK size in Edit Settings | Edit PVC size on storage classes that support volume expansion. |
| Storage tiers / performance class | vSphere storage policies (SPBM) | StorageClass selection (for example, IBM Cloud 10 IOPS/GB block). Set at PVC creation time. |
| Golden image / base disk | VM template disk on datastore | DataSource (boot source) — auto-updated from Red Hat image registry or custom import. |
{: caption="Day-2 task mapping: VMware to OpenShift Virtualization Service" caption-side="bottom"}
{: summary="This table maps common Day-2 virtual machine operations to their OpenShift Virtualization Service equivalents. The first column lists the task, the second column describes the VMware approach, and the third column describes the OpenShift Virtualization Service approach."}

## The forward view: Where this is all heading
{: #forward-view}

The seven use cases in this paper describe how VMware admins operate OpenShift Virtualization Service using interfaces and workflows that are recognizably similar to what they know today. That is the immediate goal: get the job done, keep the estate running, and build confidence on the new platform.

The longer-term direction is worth naming, even if it is not the focus today. In a mature OpenShift Virtualization Service environment, VM definitions, network profiles, placement policies, backup schedules, and monitoring rules are all stored in a Git repository. Changes go through a pull request — reviewed, approved, and automatically applied. ArgoCD continuously ensures the cluster matches the declared state and flags anything that drifts. Ansible Automation Platform handles the operations that are inherently procedural: guest OS patching, DR failover sequencing, compliance evidence collection.

This is not a requirement on day one — and it is worth being explicit about what adopting these tools involves. Git is widely available and storing VM YAML definitions in a repository requires no new infrastructure. ArgoCD is an operator deployed inside OpenShift itself; it is included in OpenShift GitOps and does not require a separate platform to run. Ansible Automation Platform is a separately licensed Red Hat product. None of these are required for Day 2 operations as described in this paper — they represent a maturity path, not a prerequisite.

Most VMware admins will operate through the console for months before adopting GitOps workflows, and that is a perfectly valid operating model. The value of the GitOps direction is most evident in environments with multiple clusters, strict change control requirements, or the need for automated compliance evidence — contexts where the effort of building the tooling is quickly offset by the operational leverage it provides.

For teams that want to explore this direction now, Red Hat's published materials on [OpenShift GitOps](https://developers.redhat.com/blog/2025/03/05/openshift-gitops-recommended-practices){: external} provide the definitive guidance. The same Git repository and ArgoCD instance that manage VM definitions can be extended to cover the full platform — containers, policies, and multi-cluster governance — without starting over.

## Conclusion
{: #conclusion}

The capabilities VMware administrators depend on are present in OpenShift Virtualization Service. Live migration, automated placement rebalancing, resource oversubscription, snapshots, enterprise backup, monitoring, alerting, console access, networking constructs, and storage management all exist and all work. The tooling underneath is KVM, the same hypervisor that powers IBM Cloud, AWS, Azure, and Google Cloud — and it has been in production use for over a decade.

What changes is not the capability set, but how you interact with it. Some tasks that were automatic in VMware require explicit declaration in OpenShift Virtualization Service. Some tasks that required clicking through multiple GUI screens are now a YAML edit and a pull request. The activity is different. The job is the same.

This paper is a starting point. Use the quick reference table to locate the task you need. The platform provides the same operational capabilities, expressed in a different language.

For a deeper treatment of this paradigm shift and the [GitOps](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.14/html-single/understanding_openshift_gitops/index){: external} principles behind it, Red Hat's published materials are the authoritative reference. This paper focuses on the practical day-to-day implications.
