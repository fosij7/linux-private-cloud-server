# Linux private cloud server: how many servers, how much IP space, and which stack? (Harvester vs MicroCloud vs Proxmox, with bare metal plans)

A Linux private cloud costs nothing in licences. Harvester and Canonical's MicroCloud are both open source and both install on hardware you rent by the month. The money goes to the server, the IP block routed to it, and the hours you spend on the install. One machine is enough to start; high availability begins at three.

Below are the three stacks people actually deploy, the IP planning that decides whether the install succeeds, what a single box can carry, and where SpinServers' published plans fit into that picture. If you want the hardware question settled first, 👉 [compare SpinServers' bare metal plans](https://bit.ly/Spinservers) and come back to the stack choice.

> **Most readers need one server, not three.** A 4-core/8-thread machine with 32 GB of ECC memory and a 1 TB NVMe runs Harvester and a handful of VMs comfortably. SpinServers lists exactly that configuration in Dallas at $79/month, billed monthly. Move to three nodes when VMs have to survive a node failure. 👉 [check the current SpinServers plan list](https://bit.ly/Spinservers)

## SpinServers plans and prices

SpinServers sells bare metal dedicated servers from two US locations, Dallas and San Jose. Four plans carry published monthly rates:

| Plan | CPU | Memory / storage | Network | Price | Order |
| --- | --- | --- | --- | --- | --- |
| Intel Xeon E3-1280 v5 (Dallas) | 4 cores / 8 threads, 3.7 GHz turbo | 32 GB DDR4 ECC, 1 TB M.2 NVMe | 30 TB/month on 10 Gbps, 1 IPv4 | $79/mo | [ View plan](https://bit.ly/Spinservers) |
| Starter Package | not published | 32 GB, 1 TB NVMe | 10 TB/month, 10 Gbps, full root | $99/mo | [ View plan](https://bit.ly/Spinservers) |
| Growth Package | not published | 256 GB, 2 × 1.6 TB SSD | 20 TB/month, 10 Gbps, full root | $149/mo | [ View plan](https://bit.ly/Spinservers) |
| High Performance Package | not published | 768 GB, 8 × 2 TB SSD | 40 TB/month, 10 Gbps, full root | $679/mo | [ View plan](https://bit.ly/Spinservers) |

Those are monthly rates with no setup fee listed on the three packages. Stock rotates as inventory sells, so treat the table as the shape of the lineup rather than a fixed catalogue.

The San Jose listings add configurations the table does not cover: 16 cores/32 threads with 64 GB DDR4 and a 1.6 TB NVMe, a 24-core/48-thread variant with the same memory, and a 32-core/64-thread machine with 512 GB RAM and 2 × 3.84 TB SSD on a hardware RAID controller with battery-backed cache. All of them list dual 10 Gbps ports, 30 TB of monthly transfer, instant provisioning and one IPv4. Their prices appear on the order page rather than in the plan list.

Two things the spec sheet does not tell you. CPU models are unpublished for the Starter, Growth and High Performance packages, so the $99 and $149 tiers differ in memory and bandwidth but not in any stated way on compute. And according to third-party write-ups, the hardware is enterprise-refurbished rather than new, which is a reasonable explanation for a 256 GB machine at $149 and also a reason to test drives after delivery.

## Three ways to build a Linux private cloud server

### Harvester: one machine, Kubernetes-native

Harvester is an open-source hyper-converged stack that manages virtual machines and Kubernetes clusters from one control plane. It is the shortest path from a single server to something that behaves like a cloud: you install it, and you get a web UI, VM images and a Kubernetes API on the same box.

The hardware requirement is one server. Three are recommended for high availability and are not a condition of entry. What people miss is IP space: at least one /29 block for the cluster and its VMs, or two /29 blocks if management and VM traffic should stay apart.

Installation runs from an iPXE script pasted into the provider's reinstall dialog, plus a YAML config file that script downloads. Four fields break installs. The config file URL has to be substituted into the boot script. The password must be supplied as a hash such as bcrypt, not as plain text. The block device path has to match the actual disk, which is the difference between `/dev/sda` and `/dev/nvme0n1`. And the cluster VIP must be an unused address in the same subnet as the server's public IP, reserved for cluster management alone. Get those right and the first node installs unattended; further nodes join using a second config file.

Suits: Kubernetes-native workloads, small footprints, one UI for VMs and containers. Does not suit: sites that need separate storage networks, or anyone who cannot get a routed IP block.

### MicroCloud: LXD, Ceph and OVN for multi-machine clusters

MicroCloud is Canonical's version of the same idea at cluster scale. LXD runs virtual machines and system containers, MicroCeph provides replicated distributed storage, MicroOVN handles software-defined networking. It is AGPLv3, written in Go, and it takes preseed configuration files for automated, repeatable deployment, including automatic joining of new machines to an existing cluster.

This is a cluster product. Ceph replication across members and redundant networking only pay off with three or more machines, and the design assumes separate network planes: one for Ceph management, one for cluster communication, one for OVN underlay traffic. Live migration of compatible VMs, LXD projects for multi-tenancy and resource isolation, fine-grained access control, GUI plus CLI plus REST API are all included. Canonical positions it for edge computing, development and test environments, distributed offices and small private data centres, which is a fair description of who it fits.

Suits: several machines, containers and VMs side by side, teams who want deployment driven from a config file rather than a console. Does not suit: a single server, or an operator who does not want three networks to reason about.

### Proxmox and containers: cheapest, most manual

The low-cost route is a hypervisor on hardware you already own. A public homelab write-up runs Immich, Nextcloud, n8n and a local LLM on a repurposed Dynabook Portégé X30-F: Core i5-8265U (4 cores/8 threads), 16 GB DDR4, 256 GB NVMe, plus a 1 TB Samsung 860 EVO over USB. Proxmox VE on bare metal, LXC containers, Docker Compose and Portainer nested inside, Cloudflare Tunnel instead of open ports, NordVPN Meshnet in gateway mode for admin access.

Three problems from that write-up are the ones you will hit. A laptop on AC power around the clock damages the battery, so the fix is a BIOS charge limit at 80% plus a Bash script that reads `/sys/class/power_supply` and shuts the machine down gracefully when power is lost and the battery drops below 15%. LXC containers cannot see a USB disk on the host, which surfaces as `EACCES` or `ENOENT` inside Docker: mount by UUID in the host's fstab, add a bind mount to the container (`mp0: /mnt/ssd,mp=/mnt/external_drive`), then correct the ownership mapping with `chown 1000:1000`. And 16 GB of RAM will not carry photo machine learning alongside everything else, so a cron job stops the Immich ML container in the daytime and starts it again at 22:00.

Suits: learning, tight budgets, spare hardware. Does not suit: anyone who needs a support contract, or who cannot absorb downtime while debugging a storage mount.

## Work out IPs and node count before you order

A /29 is eight IP addresses. Harvester wants at least one block for the cluster and its VMs, and two if management and VM traffic are separated. On top of the blocks, the cluster VIP has to be an unused address in the same subnet as the public IP and dedicated to cluster management, so it cannot be one you intend to hand to a VM.

That is where a plan with a single IPv4 stops being enough. SpinServers includes one IPv4 per server; whether a routed /29 can be added, and at what price, is a question for their sales team before you pay. Atlantic.Net charges $3.65/month for an extra IPv4 on its VPS line, which gives a sense of scale: a block of eight addresses is a recurring line item, not a rounding error.

Node count moves the bill more than CPU does. Harvester runs on one server and recommends three for high availability. MicroCloud's replication and redundant networking assume a cluster. Three nodes triples the monthly cost and multiplies the IP planning and the failure modes you have to debug. 👉 [see which SpinServers configurations are currently in stock](https://bit.ly/Spinservers)

## What one machine actually handles

Sixteen gigabytes is the entry point for a mixed workload, and it is tight. The homelab case above runs four services on 16 GB and still has to move its ML container into night hours to make room. Thirty-two gigabytes gives you Harvester plus a handful of small VMs, or Proxmox with a dozen containers, without scheduling tricks. Past that you are buying density: the $149 Growth package lists 256 GB of RAM, the point at which ten or twenty VMs per box stops being remarkable, assuming the unpublished CPU keeps up.

Storage matters for a different reason. Harvester and MicroCloud both replicate data between nodes, so three identical local disks do more for you than one large one. The 1.6 TB NVMe listings are sized for that; the 8 × 2 TB machine is sized for hosting many tenants on one box.

## If you would rather rent a VM

If the actual requirement is a Linux box that runs applications, a VPS is cheaper than anything described above. UpCloud's Starter line, priced in euros:

| Memory | CPU | Storage | Price |
| --- | --- | --- | --- |
| 1 GB | 1 | 10 GB | €3/mo |
| 4 GB | 2 | 30 GB | €12/mo |
| 8 GB | 4 | 40 GB | €20/mo |
| 16 GB | 4 | 50 GB | €28/mo |

Starter carries a 99.99% SLA and outbound transfer is not charged. Atlantic.Net bills hourly at the monthly rate divided by 672 hours, includes one IPv4, charges $0.02/GB for egress beyond the plan, and completes deployment in about 30 seconds across five US regions plus Toronto, London and Singapore.

What you rent there is a guest instance. The hypervisor, the routing and the IP block stay with the provider, which is why neither replaces a private cloud built on Harvester or MicroCloud.

## Matching a SpinServers configuration to your stack

Single-node Harvester: the $79 Dallas E3-1280 v5. Four cores and eight threads gets thin once you pass a handful of VMs, but 32 GB of ECC memory and a 1 TB NVMe cover the install and the first workloads, and 30 TB of transfer on a 10 Gbps port is more headroom than most private clouds consume.

Read the $99 Starter package carefully before defaulting to it. It lists the same 32 GB of RAM and 1 TB NVMe but 10 TB of transfer instead of 30 TB, and no CPU model. Ask what the extra $20 buys.

Three-node high availability with Harvester: three of the 16-core/32-thread, 64 GB, 1.6 TB NVMe machines in San Jose. Identical local disks make replication predictable, and the dual 10 Gbps ports let you keep replication traffic away from VM traffic.

MicroCloud: also a multi-machine purchase, and the dual 10 Gbps ports are the practical reason to look at these listings for it. Three network planes on one 10 Gbps pair is workable; on a single 1 Gbps link, Ceph replication and VM traffic compete for the same pipe.

Before ordering, know the constraints. Every listing includes one IPv4. Popular configurations sell out. Provisioning is automatic on in-stock inventory. And Harvester installs through a custom iPXE or ISO reinstall, so confirm that path is available on the plan you pick. 👉 [look at the $79 Dallas node](https://bit.ly/Spinservers)

## Linux private cloud server questions that decide the purchase

### How many servers does a private cloud need?

One, with Harvester. Three if VMs must keep running when a node dies. MicroCloud's storage replication and redundant networking only mean something across several machines.

### How many IPs is a /29?

Eight addresses. How many are usable depends on whether the provider routes the block to your server or bridges it, which is worth confirming in writing before you pay.

### Can I start with the single IPv4 that comes with the server?

You can install the cluster, but Harvester expects a block for VMs, and the VIP has to sit in the same subnet as the public IP without being in use. Plan on requesting a /29 from the start rather than migrating later.

### Do the software stacks need licences?

No. Harvester is open source and MicroCloud is AGPLv3. Your costs are hardware, IPs, power if the machine sits on your premises, and the time you put into the install.

### Is 10 Gbps necessary?

For one node, no. For three nodes replicating storage, separating that traffic onto its own port helps, which is what the dual-port listings provide.

### Can this run on a VPS instead of a dedicated server?

A VPS gives you a guest. It does not give you hypervisor control, a custom iPXE reinstall or block devices, all of which Harvester and MicroCloud assume. Start with dedicated hardware.

## Pick by scenario

- Learning, with hardware already in the room: Proxmox, no purchase.
- One box serving VMs and a small Kubernetes cluster: Harvester on the $79 Dallas node with one routed /29.
- Three-node HA with storage replication: three 16-core/32-thread 64 GB nodes with dual 10 Gbps.
- Multi-tenant or multi-site with automated redeploys: MicroCloud on three or more machines, with separate Ceph, cluster and OVN networks.
- A Linux server to run applications: a VPS at €3 to €28 a month, and no private cloud at all.

Two things to confirm before you pay: that the provider will route a /29 or larger block to your server, and that reinstall supports a custom iPXE script or ISO. Harvester installs unattended once both are in place, and not before. 👉 [start with a single SpinServers node and add two when HA matters](https://bit.ly/Spinservers)
