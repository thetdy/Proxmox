# Zeus & StartOS Sovereign Node Cheat Sheet

## 1. Network & Interface Hardening

- **Tor-Only Mode:** Activating the **Tor Only** toggle under your Core Lightning (CLN) configuration seals the external firewall. It strips away your public IPv4 block, leaving only an obfuscated .onion proxy for peer connections.
- **The WireGuard / mDNS Protocol Rule:** Your private WireGuard tunnel operates strictly at Layer 3, meaning it does not route multicast/broadcast traffic.
  - **The Symptom:** Local domain names like [http://leather-icepack.local/](http://leather-icepack.local/) will fail to resolve on your phone when connected via the VPN.
  - **The Fix:** Bypass mDNS entirely by using your raw internal IP address (192.168.0.12) for all mobile browser bookmarks and app configurations. Alternatively, manually map the domain to the IP inside your Pi-hole **Local DNS Records**.

## 2. Zeus Mobile Remote Control Setup

Zeus does not hold your seed phrase or sign transactions directly; it acts purely as an encrypted remote control for your home server.

- **Wallet Interface:** Select **Core Lightning (CLNRest)**. StartOS utilizes this lightweight Python plugin to translate your node's raw commands into a REST API that Zeus understands.
- **Authentication (Runes):** Unlike LND's Macaroons, CLN uses **Runes**. These are disposable cryptographic permission slips. They do _not_ contain your Bitcoin and do _not_ need to be backed up. If your phone is lost or stolen, simply log into StartOS and **revoke** the Rune to immediately cut off the phone's access.
- **Certificate Pinning:** The app will automatically read your node's unique self-signed TLS certificate fingerprint from the StartOS connection QR code. Pinning this certificate ensures "briefcase inside an armored car" security, protecting your traffic even if a malicious device penetrates your local LAN.

## 3. Bulletproof Privacy Lockdown

To prevent your mobile device from quietly leaking network data or tracking your IP address via public clearnet queries, apply these exact parameters:

| Setting                     | Configuration                                     | Execution Details                                                                                                                          |
| --------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Block Explorer**          | [http://192.168.0.12](http://192.168.0.12):[PORT] | Points Zeus directly to your local StartOS Mempool instance for private block lookups.                                                     |
| **Mempool Fee Suggestions** | **OFF** (or set to local IP)                      | When off, Zeus falls back to native CLN commands (feerates), querying your own Bitcoin Core daemon over the VPN tunnel.                    |
| **Fiat Price Source**       | **DISABLED**                                      | Eliminates the hardcoded third-party API pings (Coinbase, Kraken, etc.) that log your phone's IP whenever the app refreshes the USD price. |

**Understanding the Fee Markets:**

- **Off-Chain Routing Fees (Buying Coffee):** Calculated automatically by Zeus in milliseconds before you hit pay. It consists of a Base Fee (0–1 sat) + a variable Fee Rate (measured in ppm, parts per million). You never need to look these up manually.
- **On-Chain Miner Fees (Heavy Infrastructure):** Measured in sat/vByte. You only need to check these manually on your StartOS Mempool dashboard when performing on-chain activities like opening channels, closing channels, splicing, or sweeping UTXOs.

## 4. Phase 4: Channel Liquidity & Routing Management

To establish a highly connected routing node, liquidity must be actively managed using your sovereign stack.

- **Outbound Liquidity (Opening Channels):** Securely bridge your cold storage to your hot node. Send Bitcoin from your Coldcard hardware wallet, managed via Sparrow, and routed privately through your local Fulcrum indexer to your node's on-chain address. Open channels to well-connected Tier 1 hubs and Lightning Service Providers (LSPs) to minimize hop counts.
- **Inbound Liquidity (Receiving Capacity):** Secure inbound capacity without trusting centralized exchanges by utilizing submarine swaps via services like Boltz or Lightning Loop. Alternatively, coordinate trustless liquidity triangles using platforms like LightningNetwork+.
- **Active Maintenance:** Monitor channel health and liquidity balances via node management tools like ThunderHub or RideTheLightning (RTL). Access these tools strictly over the WireGuard VPN to close inactive channels and redeploy capital efficiently.

## 5. Sovereign Capital Inflow (The Liquid Exit)

To move funds off the Liquid sidechain and fund your on-chain node wallet without compromising your initial privacy pipeline:

- **The Right Way (Decentralized Swap):** Use a trustless atomic swap bridge like Boltz. You send L-BTC on Liquid, and Boltz sends mathematically unlinked mainnet BTC directly to a fresh Taproot address on your node. This breaks the transaction chain, leaves no identity paper trail, and appears as a standard Taproot spend to outside observers.
- **The Wrong Way (Official Peg-Out):** Avoid direct, official federation peg-outs. They require burning L-BTC through the Liquid Federation, which involves centralized whitelists, legal paper trails, and strict KYC/AML checkpoints.

## 6. Disaster Recovery & The Proxmox Rule

> ### ⚠️ CRITICAL WARNING: THE PROXMOX TRAP
>
> **Never use Proxmox VM snapshots, automated VM backups, or replication tasks for your StartOS node.**
>
> Rolling a Lightning node back to a snapshot—even one that is minutes old—causes immediate state-loss. When your node broadcasts an outdated database state (lightningd.sqlite3), the Lightning Network protocol automatically triggers its **penalty mechanism**. Your routing peers will immediately interpret this as a theft attempt and permanently seize 100% of the funds inside those channels.

**The Correct Backup Hierarchy:**

1. **The Master Defense (StartOS Native Backup):** Use the native StartOS backup utility to output to an external physical drive or network share. This safely captures both your on-chain HD wallet and your real-time channel database without risking a malicious state rollback.
2. **The Bare-Metal Extraction (Plan B / Disaster Recovery):** If the server melts, you can recover completely from bare-metal using two files stored safely off-site:
   - **hsm_secret:** The byte-form master seed of your node. This is the cryptographic root of your wealth; if you have this, your _on-chain_ funds can always be restored.
   - **emergency.recover:** Your static channel backup. This file must be backed up every time a new channel is opened. When loaded into a clean node, it instructs your node to request that all peers **force-close** your channels, sweeping your channel liquidity safely back onto your on-chain wallet. It will _not_ reopen your routing channels.

## Zeus & StartOS Sovereign Node Cheat Sheet - Part 2: The Privacy Pipeline

This section documents the headless `wasabi_joinmarket_pipeline.md` architecture deployed alongside your main `startos_node_100.md` within the overarching Proxmox-main.zip environment.

### 1. The Proxmox Architecture (LXC over VM)

- **Lightweight Containers:** Deploy Wasabi and JoinMarket in a dedicated Ubuntu Server LXC container (ID: 300, Hostname: privacy-pipeline) rather than a full Virtual Machine. This shares the host's Linux kernel and drastically reduces RAM and CPU overhead.
- **Resource Allocation:** Assign 2 CPU cores, 2048 MB of RAM, and roughly 20GB of disk space.
- **Storage Placement:** Install the container strictly on the Proxmox host's internal local-zfs drive to avoid fighting Fulcrum for I/O bandwidth on your external USB database drive.
- **Network Sequencing:** Assign a static internal IP (192.168.0.13) to seamlessly follow the `wireguard_vpn_200.md` tunnel at .11 and the StartOS node at .12. Ensure the container remains Unprivileged for maximum host security.

### 2. Network Isolation (The "Two Tor" Solution)

- **The Conflict:** JoinMarket requires a system-wide Tor daemon to run its yield-generating Maker service. If Wasabi is allowed to run its own internal Tor instance simultaneously, it causes port conflicts and wastes RAM.
- **The Unified Gateway:** Install a single system-wide Tor daemon (`apt install tor`) inside the container. Wasabi is smart enough to detect this master service, disable its internal Tor, and route its CoinJoins through the shared proxy.
- **Verification:** Always verify the container's exit node is successfully obfuscated by querying the local port:
  `curl --socks5 localhost:9050 https://check.torproject.org/api/ip`

### 3. The Proxmox Snapshot Strategy

- **The Lightning Exemption:** Unlike a Lightning routing node, the privacy-pipeline container strictly handles on-chain CoinJoins and does not hold real-time channel states. Therefore, it is entirely immune to the Lightning Network's malicious state-loss penalty mechanism.
- **Strategic Save Points:** You can and should use Proxmox snapshots during the build process. Creating snapshots like tor-verified-base and wasabi-extracted-base provides instant, clean rollback points if a script fails or a directory gets cluttered.

### 4. Headless Wasabi Deployment (wassabeed)

- **Background Engine:** Strip away the graphical user interface by exclusively running the `wassabeed` daemon. This minimizes resource consumption and runs silently in the background.
- **Directory Management:** Never extract `.tar.gz` archives directly into the root home folder (`~`). Always create and navigate to a dedicated directory (`mkdir -p /opt/wasabi && cd /opt/wasabi`) to prevent system dependencies from scattering across the server.
- **Dynamic Downloads:** Because GitHub repository names and version tags fluctuate, use `jq` to parse the official API and dynamically pull the exact browser download link:

  ```bash
  DOWNLOAD_URL=$(curl -sL https://api.github.com/repos/WalletWasabi/WalletWasabi/releases/latest | jq -r '.assets[] | select(.name | endswith("linux-x64.tar.gz")) | .browser_download_url')
  ```

### 5. StartOS Node Integration (The Final Link)

- **Sovereign Block Queries:** By default, Wasabi looks for a local Bitcoin node at `127.0.0.1`. Left unchanged, it will fall back to requesting block filters from public Tor nodes.
- **The Local Reroute:** Modify the daemon's internal `Config.json` using the command line to permanently point both the `BitcoinP2pEndPoint` and `BitcoinRpcEndPoint` to your sovereign StartOS IP (`192.168.0.12`).
- **The Result:** Wasabi securely verifies your CoinJoin balances by pulling blocks directly from your own Fulcrum indexer over the private Proxmox virtual network, ensuring zero data leakage to third-party servers.
