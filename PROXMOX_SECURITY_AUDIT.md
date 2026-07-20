# PROXMOX MIGRATION INITIAL SECURITY AUDIT

1. MAC Address Caching & DHCP Lease Overlap (enp2s0 to vmbr0)
   The "Gotchas" & Integrity Risks:

When transitioning enp2s0 from hosting an Ubuntu/Docker stack to a Proxmox bridge (vmbr0), the physical MAC address (58:47:ca:7d:b9:bf) remains the same, but the networking stack handles the address differently.

    ARP/MAC Caching: The router and other devices on the flat LAN currently have ARP entries mapping the static IP to enp2s0's MAC. When Proxmox creates vmbr0, it inherits the MAC address of the physical interface. Because the MAC stays the same, ARP caching is unlikely to break connectivity entirely, but the switch/router might briefly get confused by the port state cycling during the wipe and installation.
        Mitigation: A simple reboot of the router or clearing its ARP cache after the Proxmox installation finishes will resolve any stale entries and ensure clean routing to the bridge.

    DHCP Lease Reservation vs. Static Configuration: You noted the static IP is managed via a DHCP reservation on the router. However, Proxmox's management interface (on vmbr0) typically requires a static IP configured directly within the OS (/etc/network/interfaces), bypassing the DHCP client entirely.
        Risk: If the router's DHCP server doesn't actively see a DHCP request, but the IP is in use, it could theoretically cause a conflict if the reservation is removed.
        Mitigation: Keep the DHCP reservation intact on the router. This explicitly excludes your Proxmox IP from the general pool, ensuring the router never accidentally hands it out to a random device, even though Proxmox isn't actively requesting a lease.

2.  Architectural Vulnerabilities & Missing Isolation Layers
    A. Flat LAN Subnet (Missing Network Isolation)

        Risk: Operating on a single, flat LAN subnet means that if any single service (e.g., Plex or a misconfigured Nginx) is compromised, the attacker has unobstructed line-of-sight to the Proxmox Management UI, the Start9 VM, the Synology NAS, and all other local devices.
        Recommendation: While you can start flat, you should heavily plan to migrate to VLANs at the router/switch level.
            VLAN 1: Management (Proxmox UI, Synology admin interface).
            VLAN 2: Services (Plex, Nginx, Pi-hole).
            VLAN 3: Start9/High Security.

B. Unprivileged vs. Privileged LXC Containers

    Risk: Running services in Unprivileged LXCs is excellent for security. However, specific workloads require deeper access to host hardware:
        Plex (/dev/dri): Passing the Intel iGPU to an unprivileged container requires precise group ID (GID) mapping between the Proxmox host and the LXC guest. If configured incorrectly, users often default to Privileged containers, breaking isolation.
        WireGuard (/dev/net/tun): WireGuard needs access to the host's TUN device.
    Recommendation: Enforce unprivileged containers. Use Proxmox's lxc.cgroup2.devices.allow and lxc.mount.entry in the container configuration files to pass through only the specific /dev/dri and /dev/net/tun devices. Avoid the temptation to make them privileged if permissions issues arise during setup.

C. Nginx Proxy Manager (NPM) Access Lists

    Risk: Relying on NPM's Access Control Lists (ACLs) to block WAN traffic implies you are considering forwarding port 80/443 to NPM.
    Recommendation: Since you stated "No services... are exposed broadly to the WAN" except WireGuard, NPM should not have ports 80/443 forwarded from the internet edge router at all. The edge router's firewall must drop all unsolicited inbound WAN traffic by default, only allowing the WireGuard UDP port. NPM should only act as a local reverse proxy for user-friendly DNS names on the LAN and through the VPN tunnel.

D. Start9 VM Isolation & Outbound Rules

    Risk: Residing on a flat LAN exposes the Start9 VM to local broadcast traffic or scanning by other compromised LXCs.
    Recommendation: Utilize the built-in Proxmox Datacenter/Node Firewall. Apply a firewall policy directly to the Start9 VM's virtual network interface (net0):
        Default Policy: DROP all Inbound and Outbound.
        Allow Outbound: Explicitly allow outbound traffic strictly to necessary ports (e.g., Tor relay ports).
        This ensures that even on a flat LAN, the Proxmox hypervisor acts as a strict firewall for the VM.

E. NFS Backup Traffic over Main LAN

    Risk: NFS traffic is typically unencrypted in transit. Transmitting backup snapshots—which contain the full state of the VMs/LXCs, including the Start9 VM's memory state and any decrypted data—over the flat LAN exposes this highly sensitive data to packet sniffing if any local device (like an IoT device or Plex container) is compromised.
    Benefit of a Dedicated VLAN/Network: Yes, there is a massive benefit. Isolating NFS traffic onto a dedicated storage VLAN guarantees that general LAN devices cannot sniff the backup stream.
        Pro Tip: Since your enp3s0 interface is currently dormant, consider running a direct physical ethernet cable from enp3s0 to a secondary NIC on the Synology NAS. Assign them both IPs in an isolated subnet (e.g., 10.99.99.x) with no gateway. Bind the Proxmox Storage/NFS backup target strictly to this interface. This creates a physically air-gapped, zero-switch backup pipeline.

F. Unencrypted Host for High Availability

    Risk: Physical theft of the Minisforum host exposes all LXC configurations, Proxmox root passwords, and any data not encrypted at the guest level.
    Mitigation: You have accepted this risk for HA purposes. Ensure the Start9 VM's internal LUKS encryption uses a strong passphrase, and verify the Synology Hyper Backup encryption for offsite protection is functioning correctly. Ensure no sensitive keys are hardcoded in the unencrypted LXCs (like NPM configs or cleartext WireGuard client configs).

3.  Hardware & OS Compatibility
    StartOS Image Selection

        Image: StartOS v0.4.0 x86_64/AMD64-slim (FOSS-only)
        Compatibility: This image is perfectly compatible with the hardware.
            Architecture Match: The Minisforum NAB9 Plus runs an Intel Core i9, which utilizes the x86_64 instruction set.
            Proxmox VM Support: Proxmox uses KVM to virtualize x86_64 hardware natively. When creating the VM for Start9, you can pass through the virtualized x86_64 CPU cores directly to the guest OS.
            Security Posture: The FOSS-only ("slim") variant removes proprietary binary blobs, which cleanly aligns with the high-security and self-custody goals for the Start9 node.
