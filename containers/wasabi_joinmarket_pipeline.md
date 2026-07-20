# Wasabi & JoinMarket Pipeline Architecture & Setup

## 1. Target Architecture

- **Hypervisor:** Proxmox VE
- **Environment:** Debian 12 Standard LXC (1 Core, 1GB RAM minimum)
- **Network Interface:** Bridged (`vmbr0`) to the local subnet.
- **Base Node Target:** Bitcoin Knots running via StartOS VM on the same local subnet. RPC port `8332`.
- **Primary Goal:** Automate a deployment that runs a headless Wasabi daemon (`wassabeed`) and the JoinMarket CLI in parallel to act as a localized, fully automated CoinJoin-to-Lightning liquidity pipeline.

## 2. System Dependencies

Provisioning script to install the prerequisites before pulling repositories:

```bash
#!/bin/bash
sudo apt-get update
sudo apt-get install -y python3-dev python3-pip python3-venv git build-essential automake pkg-config libtool libffi-dev libssl-dev tor jq wget gnupg
```

## 3. Component A: JoinMarket Deployment

Follow these steps to deploy JoinMarket:

1. Clone the repository:

   ```bash
   git clone https://github.com/JoinMarket-Org/joinmarket-clientserver.git
   cd joinmarket-clientserver
   ```

2. Execute the headless installation:

   ```bash
   ./install.sh --without-qt
   ```

3. Generate the default `joinmarket.cfg`:

   ```bash
   source jmvenv/bin/activate
   cd scripts
   python wallet-tool.py generate
   ```

4. Configuration Patching:
   The following `sed` script modifies `~/.joinmarket/joinmarket.cfg` to point to the StartOS Knots local IP, setting `rpc_port = 8332`, and inserting placeholders for `rpc_user` and `rpc_password`.

   ```bash
   sed -i -e '/^\[BLOCKCHAIN\]/,/^\[/ s/^rpc_host = .*/rpc_host = <STARTOS_IP>/' \
          -e '/^\[BLOCKCHAIN\]/,/^\[/ s/^rpc_port = .*/rpc_port = 8332/' \
          -e '/^\[BLOCKCHAIN\]/,/^\[/ s/^rpc_user = .*/rpc_user = <RPC_USER>/' \
          -e '/^\[BLOCKCHAIN\]/,/^\[/ s/^rpc_password = .*/rpc_password = <RPC_PASSWORD>/' \
          ~/.joinmarket/joinmarket.cfg
   ```

## 4. Component B: Wasabi Deployment & Configuration

### 4.1 Cryptographic Verification & Extraction

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

### 4.2 Bypassing the StartOS SSL & Tor Network Blocks

Connecting an external daemon to StartOS requires navigating two distinct security walls: StartOS uses self-signed SSL certificates for its local network proxy (which Wasabi's `.NET` core rejects), and Wasabi inherently forces traffic through Tor (which drops local `192.168.x.x` IP addresses).

- **The Solution:** Bypass the local network and route the connection directly to the node's Tor interface (`.onion` address) on the standard RPC port `8332`.
- **Why it works:** Tor natively provides the end-to-end encryption Wasabi demands (eliminating the SSL certificate conflict) while seamlessly satisfying Wasabi's strict Tor proxy routing rules.

### 4.3 Minting RPC Credentials

StartOS actively strips standard `rpcuser` and `rpcpassword` lines from its configuration files to prevent local brute-force attacks.

- Instead of using the main server password, navigate to the **StartOS Dashboard -> Bitcoin Knots -> Actions** menu.
- Execute the **Generate RPC User Credentials** action to mathematically mint a dedicated, randomized password exclusively for Wasabi's use.

### 4.4 Injecting the Configuration Parameters

Modern Wasabi architecture relies entirely on direct JSON-RPC connections rather than legacy P2P endpoints.

- **Clearing the Markdown Bug:** We encountered `.NET` daemon crashes due to hidden rich-text markdown formatting (`//[https://...]`) carrying over from the clipboard into the terminal editor. Bypass this by manually typing the strings or using direct `sed` commands to programmatically inject the clean text.
- **The Final Parameters:** Populate `Config.json` with the `.onion` endpoint (formatted as standard `http://` since Tor natively handles the encryption) and the newly minted RPC credentials.
- **The Cache Flush:** Wipe the `~/.walletwasabi/client/IndexStore/` directory before the final boot to ensure Wasabi doesn't fall back on dirty block filters previously fetched from public nodes.

### 4.5 Validating Node Sovereignty

- **The Mempool Fallbacks:** The `Config.json` defaults point the exchange rate, fee estimation, and external broadcaster to Mempool.space. Wasabi queries Mempool.space strictly over Tor for fiat exchange rates (which the local node cannot provide) and uses it only as an emergency broadcast fallback.
- **The Proof:** Confirm the terminal output (`Fetched fee rate from RPC node...`) mathematically proves the daemon successfully bypassed the fallback and is routing primary transaction data strictly through the sovereign local node.

### 4.6 Deploying the Background Service

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
   ExecStart=/opt/wasabi/WasabiWallet/wassabeed --loglevel=trace --jsonrpcserverenabled=true --wallet=Wasabi --wallet=JoinMarketHot
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

### 4.7 Video Resource

[Wasabi Wallet Setup and Synchronization Guide](https://www.youtube.com/watch?v=tKwGkR3EcJY)

This tutorial provides a clear visual walkthrough of the wallet installation and initial setup steps, which is extremely helpful when verifying that your backend configurations are synchronizing correctly with the front-end interface.

## 5. Execution Wrappers

Lightweight, executable shell scripts to wrap manual commands.

- **Wasabi RPC Wrapper (`wasabi_rpc.sh`):**

  ```bash
  #!/bin/bash
  METHOD=$1
  PARAMS=${2:-"[]"}
  PAYLOAD=$(jq -n --arg method "$METHOD" --argjson params "$PARAMS" '{jsonrpc: "2.0", id: "1", method: $method, params: $params}')
  curl -s --data-binary "$PAYLOAD" http://127.0.0.1:37128/<WALLET_NAME> | jq
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

- **JoinMarket Extract Jar 0 Zpub (`jm_extract_zpub.sh`):**

  ```bash
  #!/bin/bash
  # Run this within the jmvenv environment and scripts directory
  python wallet-tool.py -m 0 wallet.jmdat
  ```

- **JoinMarket Taker Sweep (To Lightning On-Chain) (`sweep_to_lightning.sh`):**

  ```bash
  #!/bin/bash
  # Run this within the jmvenv environment and scripts directory
  python sendpayment.py -N 5 -m 0 wallet.jmdat 0 <DESTINATION_ADDRESS>
  ```

## 6. Strict Networking Constraints

- **No nested Tor.** JoinMarket relies on the system Tor daemon (`9050`). Wasabi natively builds its own circuits. Do not configure system-wide transparent Tor proxying.
- The LXC must have direct LAN access to the StartOS IP to hit the Knots RPC without routing through external VPNs.
