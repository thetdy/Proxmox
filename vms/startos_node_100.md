# StartOS Sovereign Bitcoin Stack (VM 100)

## Overview

* **Status**: STACK SECURED & OPERATIONAL. Currently awaiting Fulcrum database indexing completion.
* **Purpose**: Privacy-first, self-hosted Bitcoin node, indexer, and explorer stack via StartOS on Proxmox.

## Architecture & Network Privacy (Hybrid Model)

* **Primary Goal Achieved**: 100% clearnet IP masking. Zero ports forwarded on the home router.
* **Internal Access**: High-speed LAN/mDNS addresses configured for local wallet connections.
* **Remote Access**: Dedicated Tor `.onion` services generated for secure, encrypted external tunneling to the stack.
* **Clearnet Leaks Plugged**: NAT-PMP explicitly disabled in Bitcoin Knots to prevent automated router port-forwarding backdoors.

## Bitcoin Knots (The Foundation Layer)

* **Sync Status**: Initial Block Download (IBD) 100% Complete.
* **Privacy Settings**: Outbound P2P traffic strictly restricted to `onion (Tor)` and `i2p`. Tor system package successfully installed and routing.
* **Data Configuration**: Archival mode confirmed (Pruning: 0 MiB). Transaction Index (`txindex`) enabled.
* **App Messaging**: ZeroMQ (ZMQ) enabled for instantaneous local app communication (no Tor overhead).

## Dependent Services (Fulcrum & Mempool)

* **Hardware Provisioning**: Proxmox StartOS VM resources aggressively scaled (16GB RAM, 4+ Cores) to prevent Out-Of-Memory (OOM) failure during Fulcrum's heavy cryptographic indexing.
* **Fulcrum (Indexer)**: Installed and linked to Knots. Tor `.onion` service generated for private remote Electrum wallet connections (Sparrow/Wasabi).
* **Mempool (Explorer)**: Installed. Tor `.onion` service generated to allow private remote block and fee exploration without third-party metadata tracking.

## Provisioned Virtual Hardware Footprint

* **Chipset / Firmware**: Modern `q35` machine chipset architecture running on top of an isolated `OVMF (UEFI)` firmware layer.
* **Compute Resources**: 4 vCPU cores allocated with direct `host` passthrough execution to pass bare-metal Intel cryptographic instructions straight into the guest kernel.
* **Volatile Memory**: 16GB of static unballooned RAM to guarantee dedicated performance for blockchain database processing.
* **Internal OS Volume (`scsi0`)**: 100GB thin-provisioned allocation backed by the host ZFS pool. Native guest TRIM optimization enabled (`discard=on`) to automatically reclaim dead sectors.
* **Data Storage Volume (`usb0`)**: Physical 2TB external USB SSD (SSK Storage, `0bda:9210`) passed completely through to the VM layout.
* **Data Strategy (Path A)**: Drive was cleanly formatted and initialized using native StartOS LUKS storage encryption layers. IBD 100% complete.
