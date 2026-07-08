# WireGuard VPN (LXC 200)

## Overview

* **Status**: Deployed, stress-tested, and fully operational (Phase 3 Complete)
* **Purpose**: VPN Ingress for remote administration and secure access.

## Container Properties

* **ID**: 200
* **Hostname**: wireguard-vpn
* **Type**: LXC (Unprivileged container expected, requires `/dev/net/tun` passthrough mapping)

## Network Configuration

* **Static IP**: `192.168.0.11/24` (Accessible via `https://192.168.0.11`)
* **Gateway**: `192.168.0.1`
* **Cryptographic Routing**:
  * **Interface**: `wg0` active
  * **Tunnel Port**: UDP `51820`
* **External Ingress**: Endpoint active at `thetdy.synology.me:51820` (UDP). Sustained 4K 60fps remote stream successful. Host limits: 14-Core Intel i9 processing pool confirmed sufficient for heavy cryptographic tunneling.

## Management Interface

* **UI Management**: WGDashboard active at `http://192.168.0.11:10086`
