# Wasabi & Boltz Pipeline Architecture & Setup

## 1. Target Architecture

- **Hypervisor:** Proxmox VE
- **Environment:** Debian 12 Standard LXC (1 Core, 1GB RAM minimum)
- **Network Interface:** Bridged (`vmbr0`) to the local subnet.
- **Base Node Target:** Bitcoin Knots running via StartOS VM on the same local subnet. RPC port `8332`.
- **Primary Goal:** Automate a deployment that runs a headless Wasabi daemon (`wassabeed`) and integrates with Boltz and Ride The Lightning (RTL) to act as a localized, fully automated CoinJoin-to-Lightning liquidity pipeline.

## 2. System Dependencies

Provisioning script to install the prerequisites:

```bash
#!/bin/bash
sudo apt-get update
sudo apt-get install -y tor jq wget gnupg
```

## 3. Wasabi Deployment & Configuration

### 3.1 Cryptographic Verification & Extraction

To ensure a completely pristine environment free of orphaned files, execute a clean installation and mathematically verify the binaries against the official developer keys.

1. Pull the latest Linux standalone binary for `wassabeed`:

   ```bash
   # Download binary and PGP signature
   wget https://github.com/WalletWasabi/WalletWasabi/releases/download/v2.8.0/Wasabi-2.8.0-linux-x64.tar.gz
   wget https://github.com/WalletWasabi/WalletWasabi/releases/download/v2.8.0/Wasabi-2.8.0-linux-x64.tar.gz.asc

   # Verify SHA256 checksum
   echo "fd1053949660e20c280fe79e8d6b43e9149694b712247d1ae3ac3b7486616ec0  Wasabi-2.8.0-linux-x64.tar.gz" | sha256sum -c -

   # The Key Import: Pull the official zkSNACKs PGP public key directly into the root Linux keyring.
   # (zkSNACKs Key: 6FB3 872B 5D42 292F 5992 0797 8563 4832 8949 861E)
   wget -qO- https://raw.githubusercontent.com/zkSNACKs/WalletWasabi/master/PGP.txt | gpg --import

   # The Signature Match & Verification:
   # Run the `gpg --verify` command against the two files to achieve the `Good signature from "zkSNACKs"` output, proving the archive was untampered with.
   gpg --verify Wasabi-2.8.0-linux-x64.tar.gz.asc Wasabi-2.8.0-linux-x64.tar.gz

   tar -xzf Wasabi-2.8.0-linux-x64.tar.gz
   ```

### 3.2 Bypassing the StartOS SSL & Tor Network Blocks

Connecting an external daemon to StartOS requires navigating two distinct security walls: StartOS uses self-signed SSL certificates for its local network proxy (which Wasabi's `.NET` core rejects), and Wasabi inherently forces traffic through Tor (which drops local `192.168.x.x` IP addresses).

- **The Solution:** Bypass the local network and route the connection directly to the node's Tor interface (`.onion` address) on the standard RPC port `8332`.
- **Why it works:** Tor natively provides the end-to-end encryption Wasabi demands (eliminating the SSL certificate conflict) while seamlessly satisfying Wasabi's strict Tor proxy routing rules.

### 3.3 Minting RPC Credentials

StartOS actively strips standard `rpcuser` and `rpcpassword` lines from its configuration files to prevent local brute-force attacks.

- Instead of using the main server password, navigate to the **StartOS Dashboard -> Bitcoin Knots -> Actions** menu.
- Execute the **Generate RPC User Credentials** action to mathematically mint a dedicated, randomized password exclusively for Wasabi's use.

### 3.4 Injecting the Configuration Parameters

Modern Wasabi architecture relies entirely on direct JSON-RPC connections rather than legacy P2P endpoints.

- **Clearing the Markdown Bug:** We encountered `.NET` daemon crashes due to hidden rich-text markdown formatting (`//[https://...]`) carrying over from the clipboard into the terminal editor. Bypass this by manually typing the strings or using direct `sed` commands to programmatically inject the clean text.
- **The Final Parameters:** Populate `Config.json` with the `.onion` endpoint (formatted as standard `http://` since Tor natively handles the encryption) and the newly minted RPC credentials.
- **The Cache Flush:** Wipe the `~/.walletwasabi/client/IndexStore/` directory before the final boot to ensure Wasabi doesn't fall back on dirty block filters previously fetched from public nodes.

### 3.5 Validating Node Sovereignty

- **The Mempool Fallbacks:** The `Config.json` defaults point the exchange rate, fee estimation, and external broadcaster to Mempool.space. Wasabi queries Mempool.space strictly over Tor for fiat exchange rates (which the local node cannot provide) and uses it only as an emergency broadcast fallback.
- **The Proof:** Confirm the terminal output (`Fetched fee rate from RPC node...`) mathematically proves the daemon successfully bypassed the fallback and is routing primary transaction data strictly through the sovereign local node.

### 3.6 Deploying the Background Service

Because launching the daemon in the foreground causes it to crash the moment the terminal is closed (via the `Ctrl+C` interrupt signal), wrap the program in a permanent system daemon.

1. Create an unprivileged user for the daemon and set ownership of the binary:

   ```bash
   sudo useradd -r -s /bin/false wasabi
   sudo chown -R wasabi:wasabi WalletWasabi-2.8.0 # Adjust path to the extracted directory
   ```

2. Create a standard `systemd` file at `/etc/systemd/system/wasabi.service` to manage the daemon's uptime:

   ```ini
   [Unit]
   Description=Wasabi Daemon
   After=network.target

   [Service]
   ExecStart=/opt/wasabi/WasabiWallet/wassabeed --loglevel=trace --jsonrpcserverenabled=true --wallet=Wasabi
   Restart=always
   User=wasabi
   Group=wasabi

   [Install]
   WantedBy=multi-user.target
   ```

   _Enable and start the service to lock the daemon into the Proxmox startup sequence:_

   ```bash
   sudo systemctl enable wasabi
   sudo systemctl start wasabi
   ```

3. **Note to User:** Wallet files (`.json`) will be manually transferred via `scp` from the desktop client to inherit the GUI-configured CoinJoin rules.

### 3.7 Video Resource

[Wasabi Wallet Setup and Synchronization Guide](https://www.youtube.com/watch?v=tKwGkR3EcJY)

This tutorial provides a clear visual walkthrough of the wallet installation and initial setup steps, which is extremely helpful when verifying that your backend configurations are synchronizing correctly with the front-end interface.

## 4. Execution Wrappers

Lightweight, executable shell scripts to wrap manual commands.

- **Wasabi RPC Wrapper (`wasabi_rpc.sh`):**

  ```bash
  #!/bin/bash
  METHOD=$1
  PARAMS=${2:-"[]"}
  curl -s --data-binary "{\"jsonrpc\":\"2.0\",\"id\":\"1\",\"method\":\"$METHOD\", \"params\":$PARAMS}" http://127.0.0.1:37128/<WALLET_NAME> | jq
  ```

- **Wasabi Status (RPC) (`check_balance.sh`):**

  ```bash
  #!/bin/bash
  ./wasabi_rpc.sh getwalletinfo
  ```

- **Wasabi Receive Address (RPC) (`get_address.sh`):**

  ```bash
  #!/bin/bash
  ./wasabi_rpc.sh getnewaddress "[\"<LABEL>\"]"
  ```

## 5. The Boltz Liquidity Workflow

The automated sweep has been replaced by a more private, decentralized workflow using Boltz and the Liquid Network.

### Phase 1: The Disconnect (Wasabi to Liquid)

1. Open the Tor Browser (Identity A) and navigate to Boltz.
2. Generate an inbound swap: Bitcoin (BTC) -> Liquid (L-BTC).
3. Set the destination to your Aqua Liquid wallet address.
4. In Wasabi, use Manual Send to select your specific mixed UTXOs.
5. **Crucial:** Subtract the exact on-chain miner fee from the total UTXO amount so no toxic dust is left behind. Send to the Boltz invoice.

### Phase 2: The Air-Gap (The Tor Reset)

1. Wait for the L-BTC to hit Aqua.
2. In the Tor Browser, click **New Identity**. This mathematically destroys your session cookies and forces a completely new IP circuit, ensuring Boltz cannot link Phase 1 and Phase 3.

### Phase 3: The Re-Entry (Liquid to Node)

1. In Ride The Lightning (RTL), generate a fresh Taproot on-chain receive address.
2. In your clean Tor Browser (Identity B), generate an outbound Boltz swap: Liquid (L-BTC) -> Bitcoin (BTC).
3. Paste the RTL Taproot address as the destination.
4. Open Aqua and send the consolidated L-BTC to the Boltz payment address.

### Phase 4: The Channel Open

1. Wait for the Boltz transaction to confirm on-chain (3-6 blocks) in RTL.
2. Navigate to Peers -> Add your target routing node (e.g., ACINQ).
3. Open Channel -> Set your capacity amount.
4. **Crucial:** Toggle **Private Channel** to ON to prevent network gossip from linking your Taproot UTXO to your Lightning identity.

## 6. RTL's Built-In Boltz Feature (Submarine Swaps)

RTL integrates a cryptographic tool called a Submarine Swap, seamlessly managed via its built-in Boltz tab. This allows you to move funds back and forth between the Lightning Network and standard on-chain/Liquid without opening or closing channels, perfectly managing node liquidity.

### 6.1 "Looping Out" (Creating Inbound Liquidity)

When you open a new channel, it has 100% outbound liquidity. You cannot receive payments until you push funds to the other side.

1. Use RTL's Boltz feature to execute a **Swap Out**.
2. Set the destination to your cold storage hardware wallet or your Aqua Liquid wallet.
3. You pay Boltz for that on-chain transaction by sending them a payment over your new Lightning channel.

- **Result:** You physically move funds to the other side of your channel, creating inbound capacity, while securing your on-chain stack in cold storage.

### 6.2 "Looping In" (Refilling a Drained Channel)

If you spend all the sats in your channel, it becomes empty on your side, meaning you can no longer spend.

1. Instead of paying mainchain fees to open a new channel, execute a **Swap In**.
2. Send on-chain Bitcoin (or Liquid L-BTC) to Boltz.
3. Boltz pays a Lightning invoice directly into your node.

- **Result:** Your existing channel is instantly refilled with outbound spending power.

## 7. Strict Networking Constraints

- Wasabi natively builds its own Tor circuits. Do not configure system-wide transparent Tor proxying.
- The LXC must have direct LAN access to the StartOS IP to hit the Knots RPC without routing through external VPNs.
