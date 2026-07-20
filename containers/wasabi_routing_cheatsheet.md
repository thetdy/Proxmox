# Jules Cheatsheet: Headless Wasabi Routing via StartOS

## Architecture Overview

- **Host Environment:** Proxmox LXC Container (Debian/Ubuntu)
- **Target Node:** StartOS Bitcoin Knots (Tor Interface)
- **Protocols:** Wasabi Daemon (`wassabeed`)
- **Routing Strategy:** `.onion` RPC tunneling to bypass StartOS LAN port blocking (8332) and self-signed SSL certificate rejections (64747).

---

## Phase 1: StartOS RPC Credential Minting

StartOS natively strips out standard `rpcuser` and `rpcpassword` lines from its UI configuration to prevent brute-force attacks. You cannot use your server password or manually define one.

1. Navigate to the StartOS Dashboard -> **Bitcoin Knots**.
2. Click **Actions**.
3. Execute **Generate RPC User Credentials**.
4. Copy the generated Username and highly randomized Password.

---

## Phase 2: Wasabi Daemon Deployment

Wasabi 2.0 deprecated the `BitcoinP2pEndPoint` and `UseBitcoinCore` settings. It relies strictly on direct RPC connections for block filters and mempool queries.

### 1. Generate Configuration

```bash
cd /opt/wasabi/WasabiWallet
./wassabeed
# Let it run for 10 seconds, then press Ctrl+C to build the directory.
```

### 2. Configure the Tor RPC Bridge

Avoid text editors if clipboard markdown corruption (e.g., `//[http...]`) is an issue. Use `sed` to programmatically inject the routing details into the JSON file.

```bash
# Inject the StartOS Tor address
sed -i 's|"BitcoinRpcEndPoint".*|"BitcoinRpcEndPoint": "http://YOUR_KNOTS_ADDRESS.onion:8332",|g' ~/.walletwasabi/client/Config.json

# Inject the generated StartOS RPC credentials
sed -i 's|"BitcoinRpcCredentialString".*|"BitcoinRpcCredentialString": "YOUR_USER:YOUR_PASSWORD",|g' ~/.walletwasabi/client/Config.json
```

### 3. Flush Public Fallback Data

If Wasabi previously failed to connect, it will have pulled block filters from public Tor nodes. Flush the index to force sovereign routing:

```bash
rm -rf ~/.walletwasabi/client/IndexStore/
./wassabeed
```
