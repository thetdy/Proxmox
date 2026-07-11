# Wasabi GUI to StartOS RPC Connection

Switching to the GUI is a very smart move for troubleshooting. It gives you immediate visual feedback on your connection status without having to constantly parse terminal logs or fight with hidden clipboard formatting.

Here is a clear breakdown of the network conflict and the solution, formatted so you can easily reference it or pass it along:

## The Core Problem: Why Local IPs Fail

There are two distinct architectural walls preventing a standard local network connection between Wasabi and StartOS:

- **The Tor Routing Conflict:** Wasabi routes all of its traffic through the Tor network by default to ensure privacy. The Tor network inherently drops private, local network IP addresses (like 192.168.0.12), meaning the request never reaches your Proxmox server.
- **The SSL Certificate Block:** StartOS wraps its external local proxy port (64747) in a self-signed SSL certificate. Wasabi's strict internal security framework aggressively rejects unrecognized, self-signed certificates as untrusted.

## The Solution: The .onion RPC Bridge

Bypassing the local network entirely and routing the connection through your node's Tor interface natively solves both problems. Tor provides the end-to-end encryption Wasabi demands, eliminating the SSL certificate conflict, while seamlessly satisfying Wasabi's default Tor proxy routing.

## How to Configure the Wasabi GUI

Wasabi can connect to a Bitcoin full node by using the Bitcoin RPC. Instead of modifying the configuration file via the terminal, you can enter the Bitcoin RPC Endpoint and Bitcoin RPC Credential String directly in the Settings.

1. **Find Your Tor URL:** Log into your StartOS dashboard, navigate to **Bitcoin Knots** -> **Interfaces** -> **RPC Interface**, and copy the .onion address (it should end in port 8332).
2. **Open Wasabi Settings:** Launch the Wasabi Wallet GUI and click on the **Settings** gear icon.
3. **Configure the RPC:** Navigate to the **Bitcoin** tab.
4. **Enter the Endpoint:** In the "Bitcoin RPC Endpoint" field, enter your node's Tor URL formatted as standard HTTP (e.g., `http://your-address.onion:8332`). _(Note: Because Tor inherently encrypts the traffic, you must use `http://`, not `https://`)._
5. **Enter the Credentials:** In the "Bitcoin RPC Credential String" field, enter the custom keys you minted in the StartOS Actions menu earlier (e.g., `wasabi:YOUR_NEW_PASSWORD`).
6. **Verify the Connection:** Wasabi will add Bitcoin RPC to the status bar when the connection is enabled. It will display a warning triangle when it's not connected. Once the handshake is successful, the warning will disappear, and Wasabi will begin pulling blocks directly from your StartOS node.
