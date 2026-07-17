# Jules' Sovereign Privacy Pipeline: Cheat Sheet

## Architecture Overview

* **VLAN 30 (Sovereign Stack):** Proxmox LXC hosting Wasabi & JoinMarket.
* **Node:** StartOS (Bitcoin Knots & Fulcrum) via Tor.
* **Workflow:** Deposit -> Wasabi Mixes (Jar 0) -> JoinMarket Sweeps to Lightning.

---

## 1. Node Database Management (StartOS)

JoinMarket requires a dedicated descriptor wallet database on the Knots node to track addresses.

**Create the Node Database:**
curl -x socks5h://localhost:9050 --user "USER:PASSWORD" \
  --data-binary '{"jsonrpc": "1.0", "id": "1", "method": "createwallet", "params": {"wallet_name": "jm_wallet", "disable_private_keys": true, "blank": true, "descriptors": true, "load_on_startup": true}}' \
  -H 'content-type: text/plain;' http://YOUR_NODE.onion:8332/

**Force a Rescan (If importing an old seed):**
curl -x socks5h://localhost:9050 --user "USER:PASSWORD" \
  --data-binary '{"jsonrpc": "1.0", "id": "1", "method": "rescanblockchain", "params": [800000]}' \
  -H 'content-type: text/plain;' http://YOUR_NODE.onion:8332/wallet/jm_wallet

---

## 2. JoinMarket Quick Commands

*Run all JM commands from `/opt/joinmarket-clientserver/scripts/` using `torsocks`.*

**Wallet Summary (View Balances & Addresses):**
torsocks ./wallet-tool.py wallet.jmdat

**View Recovery Seed:**
torsocks ./wallet-tool.py wallet.jmdat showseed

**Extract Zpub (Mixdepth 0):**
torsocks ./wallet-tool.py -m 0 wallet.jmdat

**Start Yield Generator (Maker Mode):**
torsocks python yg-privacyenhanced.py wallet.jmdat

**Sweep to Lightning Node (Taker Mode):**
torsocks python sendpayment.py -N 5 -m 0 wallet.jmdat 0 <STARTOS_LIGHTNING_ADDRESS>

---

## 3. Wasabi Quick Commands

*Run all Wasabi commands from `/opt/wasabi/`.*

**Start the Headless Daemon:**
./wassabeed &

**Recover an Existing Seed:**
./wassabocli wallet recover --wallet:jm_pipeline

**Start Auto-Coinjoin:**
./wassabocli mix start --wallet:jm_pipeline

**Stop Auto-Coinjoin:**
./wassabocli mix stop --wallet:jm_pipeline

---

## 4. The Execution Workflow

To prevent database conflicts and privacy leaks on a shared seed, follow this strict order of operations:

1. **Stop JoinMarket:** Ensure `yg-privacyenhanced.py` is NOT running.
2. **Start Wasabi:** Launch `wassabeed` and begin the mix.
3. **Wait for 100%:** Allow Wasabi to achieve full anonymity set targets.
4. **Stop Wasabi:** Stop the mixing daemon entirely.
5. **Engage JoinMarket:** Launch the Yield Generator to earn routing fees on the
mixed UTXOs in Jar 0, OR sweep them directly to your Lightning Node.

---

## 5. Critical File Locations

* **Systemd Service File:** `/etc/systemd/system/wasabi.service`
* **Global Engine Configuration:** `/root/.walletwasabi/client/Config.json`
* **Hot Wallet Config:** `/root/.walletwasabi/client/Wallets/Wasabi.json`
* **Cold Destination Config:** `/root/.walletwasabi/client/Wallets/JoinMarket.json`

---

## 6. Configuration Changes & States

* **Systemd Auto-Mounting:** Modified the `ExecStart` line in `wasabi.service` to auto-load both wallets into active memory on boot. The line now reads:
  `ExecStart=/opt/wasabi/WasabiWallet/wassabeed --jsonrpcserverenabled=true --wallet=Wasabi --wallet=JoinMarket`
* **Target Anonymity Score (`AnonScoreTarget`):** Changed from default (`5`/`10`) to `35` inside **both** `Wasabi.json` and `JoinMarket.json`. This ensures the sweep command loops internally until the score hits 35 before executing the final bridge transaction.
* **Economic Safeguard (`PlebStopThreshold`):** Temporarily lowered in `Wasabi.json` to `0.001` to allow a small `0.0045` BTC test UTXO to be picked up by the automated sweep engine. *(Note: Should be returned to `0.005` for mainnet operation).*
* **Dust Shield (`DustThreshold`):** Verified it remains at the default `0.00001` BTC (1,000 sats) inside `Config.json` to automatically ignore and isolate chain-analysis dusting attacks.

---

## 7. The Core Command Cheatsheet

### Process & Daemon Management

* **Reload Systemd (after config edits):** `systemctl daemon-reload`
* **Restart Daemon:** `systemctl restart wasabi.service`
* **Filter Live WabiSabi Logs:** `journalctl -fu wasabi.service | grep -i "WabiSabi\|CoinJoin"`

### Wasabi RPC Commands (Port 37128)

* **Trigger Automated Pipeline:**
  `curl -s --data-binary '{"jsonrpc":"2.0","id":"1","method":"startcoinjoinsweep","params":["", "JoinMarket"]}' http://127.0.0.1:37128/Wasabi`
* **Force Internal Mix (Ignores PlebStop):**
  `curl -s --data-binary '{"jsonrpc":"2.0","id":"1","method":"startcoinjoin","params":["", true, true]}' http://127.0.0.1:37128/Wasabi`
* **Check Spendable Balances:**
  `curl -s --data-binary '{"jsonrpc":"2.0","id":"1","method":"listunspentcoins"}' http://127.0.0.1:37128/Wasabi`
* **Generate Fresh Hot Wallet Address:**
  `curl -s --data-binary '{"jsonrpc":"2.0","id":"1","method":"getnewaddress","params":{"label":"Remix"}}' http://127.0.0.1:37128/Wasabi`

### Live Dashboard Loop

Run this to track the real-time sweep destination:

```bash
while true; do
  curl -s --data-binary '{"jsonrpc":"2.0","id":"1","method":"listpaymentsincoinjoin"}' http://127.0.0.1:37128/Wasabi
  echo ""
  sleep 5
done
```

---

## 8. Problems Solved & Edge Cases Documented

* **Premature Sweep Execution:** Discovered that `startcoinjoinsweep` calculates its execution based on the *destination wallet's* configuration file. If `JoinMarket.json` is set to an anonymity score of 10, Wasabi will sweep after a single round. Setting the destination file to 35 forces Wasabi to hold the funds in the hot wallet and mix internally until the threshold is met.
* **Sweep Command Skipping Funds:** Found that the `startcoinjoinsweep` command strictly adheres to the total wallet balance requirement (`PlebStopThreshold`). Unlike the standard `startcoinjoin` command, the sweep command does not accept an inline override parameter. You must edit the threshold in the `.json` file to sweep smaller amounts.
* **Missing Dust / "Ghost" UTXOs:** Small UTXOs missing from the `listunspentcoins` output are intentionally hidden by the `DustThreshold` limit. It was determined best practice to leave these invisible and unspent to avoid linking mixed funds to chain-analysis dusting attacks.
* **"No Wallet Loaded" RPC Errors:** Resolved by explicitly passing `--wallet` arguments to the systemd service. If omitted, the Wasabi daemon boots in a blank state and rejects all RPC commands until wallets are manually loaded via API.
