# Jules' Sovereign Privacy Pipeline: Cheat Sheet

## Architecture Overview

- **VLAN 30 (Sovereign Stack):** Proxmox LXC hosting Wasabi.
- **Node:** StartOS (Bitcoin Knots & Fulcrum) via Tor.
- **Workflow:** Deposit -> Wasabi Mixes -> Boltz (via Liquid) -> Ride The Lightning (RTL).

---

## 1. Wasabi Quick Commands

_Run all Wasabi commands from `/opt/wasabi/`._

**Start the Headless Daemon:**
./wassabeed &

**Recover an Existing Seed:**
./wassabocli wallet recover --wallet:boltz_pipeline

**Start Auto-Coinjoin:**
./wassabocli mix start --wallet:boltz_pipeline

**Stop Auto-Coinjoin:**
./wassabocli mix stop --wallet:boltz_pipeline

---

## 2. The Execution Workflow (Boltz/Liquid Exit)

Follow this 4-phase strict workflow to securely bridge your mixed Wasabi funds to your Lightning node.

### Phase 1: The Disconnect (Wasabi to Liquid)

1. Open the Tor Browser (Identity A) and navigate to Boltz.
2. Generate an inbound swap: Bitcoin (BTC) -> Liquid (L-BTC).
3. Set the destination to your Aqua Liquid wallet address.
4. In Wasabi, use Manual Send to select your specific UTXOs.
5. **Crucial:** Subtract the exact on-chain miner fee from the total UTXO amount so no toxic dust is left behind. Send to the Boltz invoice.

### Phase 2: The Air-Gap (The Tor Reset)

1. Wait for the L-BTC to hit Aqua.
2. In the Tor Browser, click **New Identity**. This mathematically destroys your session cookies and forces a completely new IP circuit.

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

---

## 3. RTL's Built-In Boltz Feature (Submarine Swaps)

RTL natively integrates Boltz Submarine Swaps to seamlessly manage node liquidity.

- **"Looping Out" (Creating Inbound Liquidity):** Use RTL's Swap Out to send funds from your full channel to your Aqua Liquid wallet (or cold storage). You gain inbound receiving capacity while moving sats out.
- **"Looping In" (Refilling a Drained Channel):** Use RTL's Swap In to send Liquid L-BTC to Boltz. Boltz instantly refills your depleted Lightning channel with outbound spending power.

---

## 4. Critical File Locations

- **Systemd Service File:** `/etc/systemd/system/wasabi.service`
- **Global Engine Configuration:** `/root/.walletwasabi/client/Config.json`
- **Hot Wallet Config:** `/root/.walletwasabi/client/Wallets/Wasabi.json`

---

## 5. Configuration Changes & States

- **Systemd Auto-Mounting:** Modified the `ExecStart` line in `wasabi.service` to auto-load the wallet into active memory on boot, while forcing JSON-RPC and Trace logging. The line now reads:
  `ExecStart=/opt/wasabi/WasabiWallet/wassabeed --loglevel=trace --jsonrpcserverenabled=true --wallet=Wasabi`
- **Target Anonymity Score (`AnonScoreTarget`):** Changed from default (`5`/`10`) to `35` inside `Wasabi.json`.
- **Dust Shield (`DustThreshold`):** Verified it remains at the default `0.00001` BTC (1,000 sats) inside `Config.json` to automatically ignore and isolate chain-analysis dusting attacks.

---

## 6. The Core Command Cheatsheet

### Process & Daemon Management

- **Reload Systemd (after config edits):** `systemctl daemon-reload`
- **Restart Daemon:** `systemctl restart wasabi.service`
- **Filter Live WabiSabi Logs:** `journalctl -fu wasabi.service | grep -i "WabiSabi\|CoinJoin"`

### Wasabi RPC Commands (Port 37128)

- **Generate Fresh Hot Wallet Address:**
  `curl -s --data-binary '{"jsonrpc":"2.0","id":"1","method":"getnewaddress","params":{"label":"Remix"}}' http://127.0.0.1:37128/Wasabi`
