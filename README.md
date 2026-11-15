# VNC-via-SSH
VNC via SSH python program by Alterhimez. Needs to use some external vnc viewer such as TightVNC (tested), TigerVNC, RealVNC, UltraVNC

## Why

x11vnc's `-ssl` option wraps the *entire* VNC connection in TLS. Ordinary Windows viewers cannot speak that, so they can't connect directly. This app handles the SSH tunnel and the TLS layer itself and exposes a plain VNC port on `127.0.0.1`, which any viewer can use.

```
VNC viewer ──plain──> 127.0.0.1:LPORT (this app)
                          │
                          └─ TLS ─ SSH tunnel (direct-tcpip) ──> 127.0.0.1:5900 on the server (x11vnc -ssl)
```

## Features

- SSH login with password, key file, or SSH agent
- Trust-on-first-use for the **SSH host key** (stored in `known_hosts`)
- Trust-on-first-use **certificate pinning** for the x11vnc certificate (SHA-256); a changed certificate is refused
- Local listener bound to `127.0.0.1` only
- Auto-detects common viewer install paths; viewer and arguments are configurable
- Tunnel closes automatically when the viewer exits
- Settings remembered between runs (passwords are never saved)

## Requirements

- Windows with Python 3.9+
- `paramiko` (`pip install -r requirements.txt`)
- A VNC viewer installed (e.g. TigerVNC, TightVNC, RealVNC Viewer)

## Usage

```
pip install -r requirements.txt
python x11vnc_viewer.py
```

Fill in:

| Field | Meaning |
| --- | --- |
| SSH host / port / user | The server you log into over SSH |
| SSH password / key passphrase | Password for SSH, or the passphrase of your key |
| SSH key file | Optional private key path |
| x11vnc port on server | Port x11vnc listens on, relative to the server itself (default `5900`) |
| Local port | Port opened on your PC; `0` picks a free one |
| VNC viewer exe | Path to your viewer; leave blank to only open the tunnel |
| Viewer arguments | Default `127.0.0.1::{port}` (works with TigerVNC and TightVNC) |

Click **Connect**. On first use you'll be asked to confirm the SSH host key and the x11vnc certificate fingerprint.

The VNC password asked by your viewer is the x11vnc password, not the SSH password.

### RealVNC note

RealVNC Viewer warns that the connection is unencrypted. That hop is only between the viewer and this app on `127.0.0.1`; the rest is TLS inside SSH. Set a fixed **Local port** (e.g. `15900`) and tick "Remember this decision" to stop the prompt.

## Server setup

Run x11vnc as the user who owns the desktop session:

```bash
x11vnc -storepasswd                      # sets ~/.vnc/passwd
x11vnc -display :0 -auth guess -rfbauth ~/.vnc/passwd \
       -ssl SAVE_NOPROMPT -localhost -forever -shared -rfbport 5900
```

`-localhost` restricts x11vnc to loopback connections, which is all this app needs because the tunnel connects from the server itself.

### Autostart with systemd (user service)

`~/.config/systemd/user/x11vnc.service`:

```ini
[Unit]
Description=x11vnc (SSL, localhost only)
After=graphical-session.target
PartOf=graphical-session.target

[Service]
ExecStart=/usr/bin/x11vnc -display :0 -auth guess -rfbauth %h/.vnc/passwd -ssl SAVE_NOPROMPT -localhost -forever -shared -rfbport 5900
Restart=on-failure
RestartSec=5

[Install]
WantedBy=graphical-session.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now x11vnc
```

## Build a standalone .exe

```
pip install pyinstaller
python -m PyInstaller --onefile --noconsole x11vnc_viewer.py
```

The executable appears in `dist/`.

## Files

Config and pinned keys live in `%APPDATA%\x11vnc-viewer\`:

- `config.json`: last-used settings (no passwords)
- `known_hosts`: trusted SSH host keys
- `known_certs.json`: pinned x11vnc certificate fingerprints. Delete an entry if you regenerated the server certificate on purpose.

## Troubleshooting

| Log message | Likely cause |
| --- | --- |
| `No authentication methods available` | Password field empty and no usable key found |
| `Authentication failed` | Wrong SSH password (not the VNC password) or key not authorized |
| `ChannelException(2, 'Connect failed')` | x11vnc isn't running or isn't on that port (`ss -tlnp \| grep 5900`) |
| TLS handshake timeout | x11vnc is running **without** `-ssl` |
| `certificate CHANGED` | Server certificate was regenerated; remove its entry from `known_certs.json` |

## Security notes

- SSH already encrypts the link; TLS is required only because x11vnc was started with `-ssl`.
- Never expose x11vnc to untrusted networks; keep `-localhost` and reach it through SSH or a VPN such as Tailscale.
- Classic VNC passwords are truncated to 8 characters.

## License

GPL-3.0 license
