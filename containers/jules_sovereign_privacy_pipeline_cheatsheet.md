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
