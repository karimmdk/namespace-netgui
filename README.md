# NetGUI Web

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![Flask](https://img.shields.io/badge/Flask-3.0-black)](https://flask.palletsprojects.com/)

**NetGUI Web** is a browser-based frontend for managing nested network namespaces, VPN tunnels, and SOCKS5 proxies – a modern, dark-themed web UI that works alongside (not instead of) the PyQt6 desktop app.  
It uses the exact same privileged backend (`netgui/core/helper.py` via `pkexec`) and shared profile storage (`~/.config/netgui/profiles.json`) so you can switch between desktop and web seamlessly.

---

## ✨ Features

- **Namespace Management** – Create, inspect, and delete isolated network namespaces (host-attached or nested chains).
- **Chain Wizard** – Build multi‑hop VPN chains with a single click (FortiClient, OpenConnect, OpenVPN, Psiphon, Nekoray).
- **Built‑in SOCKS5 Proxy** – Launch a pure‑Python SOCKS5 server inside any namespace.
- **Port Bridges** – Expose any internal port to the host using `socat` (with automatic handling of quoted commands).
- **Live Log Viewer** – Tail tunnel/proxy/bridge logs in real time and send interactive input (e.g., `yes` to accept certificates).
- **Profile Manager** – Save and load VPN configurations (without storing passwords).
- **Diagnostics** – Test public IP, ping, and run leak tests directly from the UI.
- **Dark/Light Theme** – Automatically adapts to your system preference or manual toggle.

---

## 📦 Requirements

- **Linux** (with `ip netns`, `iptables`, `pkexec`)
- **Python 3.8+**
- **Dependencies** (installed automatically via `requirements.txt`):
  - Flask >= 3.0.0
- **Runtime dependencies** (installed on demand or manually):
  - `socat` – for port forwarding
  - `openconnect`, `openfortivpn`, `openvpn`, `psiphon`, `nekoray` – depending on your VPN type
  - `iproute2`, `iptables` – usually pre‑installed
  - `polkit` – for `pkexec`

---

## 🚀 Quick Start

```bash
# Clone the repository
git clone https://github.com/your-username/netgui-web.git
cd netgui-web

# Create and activate a virtual environment (recommended)
python3 -m venv venv
source venv/bin/activate

# Install Python dependencies
pip install -r requirements.txt

# Start the web server (as a normal user – never with sudo!)
python3 run_web.py
```

The server will start on `http://127.0.0.1:8765` and automatically open your default browser.  
**Do not run it with `sudo` or `pkexec`** – all privileged actions are delegated to `helper.py` via `pkexec` on demand.

---

## 🖥️ Usage Overview

### 1. Namespaces  
- Create a new namespace by giving it a name, optional parent, IP addresses, and DNS servers.
- Existing namespaces are listed with their public IP (retrieved on demand).
- Delete a namespace and all its children/tunnels/bridges in one click.

### 2. Chain Wizard  
- Choose a preset (e.g., `FortiClient + Psiphon`) or manually configure each hop.
- Adjust the number of hops (1–6) and fill in VPN‑specific parameters (config paths, server, username, etc.).
- Click **Build chain & connect** – it automatically creates the nested namespaces and launches each tunnel.
- The **route bar** at the top visually shows the live signal path from Host → Hop1 → Hop2 → … → App.

### 3. Proxy  
- Select a namespace, internal port (default 1080), and optionally expose it on the host at a chosen port.
- Click **Start proxy** – the built‑in SOCKS5 server starts inside the namespace.
- If “Also expose on host” is checked, a `socat` bridge is created automatically.
- Use `127.0.0.1:<host_port>` as your SOCKS5 address in any application (browser, curl, Telegram, etc.).

### 4. Bridges  
- Manually add raw port forwards: `host_port` → `namespace:target_port` using `socat`.
- Useful when you want to expose arbitrary services running inside a namespace.

### 5. Profiles  
- Save VPN configurations with a name and type.
- Load a profile into the Chain Wizard with one click.
- Import/export all profiles as a JSON file.

### 6. Diagnostics & Logs  
- Test public IP, ping, and run DNS/IP leak tests from inside any namespace.
- Launch graphical applications (Firefox, Chromium, etc.) directly inside a namespace.
- View live logs of tunnels, proxies, and bridges.
- Send interactive input (e.g., `yes` to accept a certificate) to a running tunnel.

### 7. Settings  
- Switch between dark and light themes.
- Check and install missing system dependencies (on Arch Linux with `pacman`).

---

## 🧱 Project Structure

```
netgui-web/
├── run_web.py                 # Launcher (like run.py, but opens a browser)
├── requirements.txt
├── README.md                  # This file
├── netgui/
│   ├── core/                  # Unchanged from the desktop app, + SOCKS5 fix
│   │   ├── bridge_client.py   # pkexec wrapper
│   │   ├── constants.py
│   │   ├── helper.py          # Privileged backend (run via pkexec)
│   │   ├── presets.py
│   │   └── profiles.py
│   └── web/
│       ├── app.py             # Flask REST API
│       ├── static/
│       │   ├── app.js         # Frontend logic (vanilla JS)
│       │   └── style.css      # Dark/light theme
│       └── templates/
│           └── index.html     # Single‑page UI
```

---

## 🔐 Security & Privilege Model

- The Flask server **runs as your normal user** – it does **not** require root.
- Every write action (creating namespaces, starting tunnels, adding bridges) is performed by `helper.py` through `pkexec`, which triggers a graphical password prompt (just like the desktop app).
- State is stored in `/run/netgui/state.json` and logs in `/run/netgui/logs/` (world‑readable so the web UI can tail them).
- Profiles are stored in `~/.config/netgui/profiles.json` – plaintext server/usernames, but **never** plaintext passwords (they are passed directly to the helper only when needed).

---

## 🐛 Troubleshooting

### “socat: wrong number of parameters” or proxy connection closed  
This happens when the `EXEC` argument is not quoted correctly. The current code uses `shlex.quote()` to ensure the entire command is passed as a single string. If you still encounter issues:

1. Ensure `socat` is installed (`which socat`).
2. Check the bridge log: `tail -f /run/netgui/logs/bridge-*.log`.
3. Manually test the `socat` command from a terminal:
   ```bash
   socat TCP-LISTEN:3080,fork,reuseaddr EXEC:'ip netns exec chain-1 /usr/bin/socat STDIO TCP:127.0.0.1:1080'
   ```
   Then test with `curl --socks5 127.0.0.1:3080 https://checkip.amazonaws.com`.

### Proxy works inside namespace but not from host  
- Make sure the bridge is running (`ss -tlnp | grep <host_port>`).
- Verify that the SOCKS5 server is listening on `127.0.0.1` inside the namespace.
- If the bridge is up but `curl` fails, read the bridge log – it often reveals a missing `socat` path or permission issue.

### “pkexec not found”  
Install `polkit` (e.g., `sudo pacman -S polkit` on Arch).

### “ip netns: command not found”  
Install `iproute2` (usually pre‑installed).

---

## 🤝 Contributing

Contributions are welcome! Feel free to open issues or pull requests for improvements, bug fixes, or new features.

- Make sure to test your changes in a clean environment.
- Follow the existing code style (PEP 8 for Python, vanilla JS for frontend).
- Update this README if you add or change functionality.

---

## 📄 License

This project is licensed under the **GNU General Public License v3.0** – see the [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgements

- Built with [Flask](https://flask.palletsprojects.com/) and vanilla JavaScript.
- Inspired by the original NetGUI desktop app and the `vpn_chain.sh` script.
- Special thanks to the open‑source community for providing the tools (`socat`, `openconnect`, etc.) that make this possible.
