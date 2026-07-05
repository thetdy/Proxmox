# WireGuard VPN (LXC 200)

## Overview
* **Status**: Deployed and operational (Phase 3 Complete)
* **Purpose**: VPN Ingress for remote administration and secure access.

## Container Properties
* **ID**: 200
* **Hostname**: wireguard-vpn
* **Type**: LXC (Unprivileged container expected, requires `/dev/net/tun` passthrough mapping)

## Network Configuration
* **Static IP**: `192.168.0.11/24`
* **Gateway**: `192.168.0.1`
* **Cryptographic Routing**:
    * **Interface**: `wg0` active
    * **Tunnel Port**: UDP `51820`
* **External Ingress**: Awaiting edge router NAT rule (UDP `51820` -> `192.168.0.11`) to finalize remote handshake capabilities.

## Management Interface
* **UI Management**: WGDashboard active at `http://192.168.0.11:10086`
