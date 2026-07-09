# PROXMOX 9.2 & STARTOS ARCHITECTURE MATRIX: STATUS REPORT

# Target Audience: Jules (Security & Integrity Auditor)

# Status: Initial Host Baseline & Guest Provisioning Complete. Host is running headless

## 1. SYSTEM BASELINE & HARDWARE TOPOLOGY

* Host Machine: Minisforum NAB9 Plus (Intel Core i9, 64GB RAM, Dual 2.5GbE NICs).
* Hypervisor OS: Proxmox VE 9.2 (Bare-metal deployment based on Debian Trixie layer).
* Core Workstation: Native Pop!_OS system executing administration via Mozilla Firefox & SSH.
* Primary Host Storage: 500GB Internal NVMe (`/dev/nvme0n1`) initialized as a single-disk ZFS RAID0 array. This profile handles host binaries and local guest root partitions while enforcing native ZFS copy-on-write capabilities for rapid snapshot delivery to the Synology NAS backend (`THETDY-SYN`).

## 2. SYSTEM-LEVEL HARDENING POLICIES

* Management Network Layer: Hardcoded static footprint (`192.168.0.10/24`) bound directly to physical port `enp2s0` (permanently pinned as internal hardware target `nic0` via Proxmox 9 predictable hardware mapping). Accessible via `https://192.168.0.10`.
* IP Conflict Prevention: Router-level DHCP reservation remains active for MAC `58:47:ca:7d:b9:bf` to form a hybrid static reservation boundary.
* CLI Security Profile: Host SSH daemon updated to completely disable password entry fields (`PasswordAuthentication no`). Authentication is strictly limited to cryptographic Ed25519 public key verification signatures.
* Web UI Security Profile: Data-center level authentication hardened via root-account Time-Based One-Time Password (TOTP) 2FA.

## 3. ACTIVE GUEST RUNTIME ARCHITECTURE: VM 100 (startos-node)

* Guest OS: StartOS x86_64 architecture (Native self-custody environment).
* Local Identity Matrix: Permanently accessible via local network domain `http://leather-icepack.local` and static IP `https://192.168.0.12`.
* Workstation Trust Layer: StartOS local Root Certificate Authority (Root CA) manually integrated into the Firefox NSS Certificate Database on Pop!_OS. This enforces valid local TLS end-to-end encryption across all configuration paths.

### Provisioned Virtual Hardware Footprint

* Chipset / Firmware: Modern `q35` machine chipset architecture running on top of an isolated `OVMF (UEFI)` firmware layer.
* Compute Resources: 4 vCPU cores allocated with direct `host` passthrough execution to pass bare-metal Intel cryptographic instructions straight into the guest kernel.
* Volatile Memory: 16GB of static unballooned RAM to guarantee dedicated performance for blockchain database processing.
* Internal OS Volume (`scsi0`): 100GB thin-provisioned allocation backed by the host ZFS pool. Native guest TRIM optimization enabled (`discard=on`) to automatically reclaim dead sectors.
* Data Storage Volume (`usb0`): Physical 2TB external USB SSD (SSK Storage, `0bda:9210`) passed completely through to the VM layout.
* Data Strategy (Path A): Drive was cleanly formatted and initialized using native StartOS LUKS storage encryption layers. Initial Block Download (IBD) is 100% complete.
* Further Details: See `vms/startos_node_100.md` for extended privacy, networking, and Bitcoin stack (Knots, Fulcrum, Mempool) details.

## 4. ACTIVE GUEST RUNTIME ARCHITECTURE: LXC 200 (wireguard-vpn)

* Guest Environment: WireGuard VPN container (Phase 3 Complete).
* Network Layer: Static IP `192.168.0.11/24` (Gateway: `192.168.0.1`). Accessible via `https://192.168.0.11`.
* Cryptographic Routing: Interface `wg0` active. Tunnel port set to UDP `51820`.
* UI Management: WGDashboard active at `http://192.168.0.11:10086`.
* Telemetry Note: Proxmox LXC metrics do not reflect kernel-space WireGuard encryption; refer to main node metrics for true CPU load.
* Further Details: See `containers/wireguard_vpn_200.md` for extended configuration information and external ingress status.
