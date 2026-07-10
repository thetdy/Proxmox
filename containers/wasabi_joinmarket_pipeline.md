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
sudo apt-get install -y python3-dev python3-pip python3-venv git build-essential automake pkg-config libtool libffi-dev libssl-dev tor jq wget libsodium-dev
```

## 3. Component A: JoinMarket-NG Deployment
Follow these steps to deploy the modern JoinMarket-NG client:

1. Clone the repository:
   ```bash
   git clone https://github.com/JoinMarket-NG/joinmarket-ng.git
   cd joinmarket-ng
   ```
2. Execute the installation (accepting default Tor configurations):
   ```bash
   ./install.sh -y
   ```
3. Generate the default `config.toml` (if not already created by the installer):
   ```bash
   source ~/.joinmarket-ng/activate.sh
   jm-wallet generate --save --prompt-password --output ~/.joinmarket-ng/wallets/wallet.mnemonic
   ```
4. Configuration Patching:
   The installer automatically sets up Tor and places `config.toml` in `~/.joinmarket-ng/config.toml`.
   The following `sed` commands automate the modification to point to the StartOS Knots local IP, setting `rpc_port = 8332`, and inserting placeholders for `rpc_user` and `rpc_password` under the `[bitcoin]` block.

   ```bash
   sed -i 's/^rpc_host = .*/rpc_host = "<STARTOS_IP>"/' ~/.joinmarket-ng/config.toml
   sed -i 's/^rpc_port = .*/rpc_port = 8332/' ~/.joinmarket-ng/config.toml
   sed -i 's/^rpc_user = .*/rpc_user = "<RPC_USER>"/' ~/.joinmarket-ng/config.toml
   sed -i 's/^rpc_password = .*/rpc_password = "<RPC_PASSWORD>"/' ~/.joinmarket-ng/config.toml
   ```

## 4. Component B: Wasabi Deployment
1. Pull the latest Linux standalone binary for `wassabeed`:
   ```bash
   wget https://github.com/WalletWasabi/WalletWasabi/releases/download/v2.8.0/Wasabi-2.8.0-linux-x64.tar.gz
   tar -xzf Wasabi-2.8.0-linux-x64.tar.gz
   ```
2. Generate a `systemd` service file to manage the daemon's uptime:
   Create a file `/etc/systemd/system/wassabeed.service`:
   ```ini
   [Unit]
   Description=Wasabi Daemon
   After=network.target

   [Service]
   ExecStart=/path/to/wassabeed --wallet=<WALLET_NAME> --jsonrpcserverenabled=true
   Restart=always
   User=root

   [Install]
   WantedBy=multi-user.target
   ```
   *Enable and start the service:*
   ```bash
   sudo systemctl enable wassabeed
   sudo systemctl start wassabeed
   ```
3. **Note to User:** Wallet files (`.json`) will be manually transferred via `scp` from the desktop client to inherit the GUI-configured CoinJoin rules.

## 5. Execution Wrappers
Lightweight, executable shell scripts to wrap manual commands.

- **Wasabi Status (RPC) (`check_balance.sh`):**
  ```bash
  #!/bin/bash
  curl -s --data-binary '{"jsonrpc":"2.0","id":"1","method":"getwalletinfo"}' http://127.0.0.1:37128/<WALLET_NAME> | jq
  ```

- **Wasabi Receive Address (RPC) (`get_address.sh`):**
  ```bash
  #!/bin/bash
  curl -s --data-binary '{"jsonrpc":"2.0","id":"1","method":"getnewaddress", "params":["<LABEL>"]}' http://127.0.0.1:37128/<WALLET_NAME> | jq
  ```

- **JoinMarket-NG Extract Zpub (`jm_extract_zpub.sh`):**
  ```bash
  #!/bin/bash
  # Run this within the joinmarket-ng venv environment
  jm-wallet xpub -m 0 ~/.joinmarket-ng/wallets/wallet.mnemonic
  ```

- **JoinMarket-NG Taker Sweep (To Lightning On-Chain) (`sweep_to_lightning.sh`):**
  ```bash
  #!/bin/bash
  # Run this within the joinmarket-ng venv environment
  jm-taker send -f ~/.joinmarket-ng/wallets/wallet.mnemonic --mixdepth 0 --amount 0 --makers 5 --destination <DESTINATION_ADDRESS>
  ```

## 6. Strict Networking Constraints
- **No nested Tor.** JoinMarket-NG configures Tor dynamically and defaults to Tor control port `9051`. Wasabi natively builds its own circuits. Do not configure system-wide transparent Tor proxying.
- The LXC must have direct LAN access to the StartOS IP to hit the Knots RPC without routing through external VPNs.
