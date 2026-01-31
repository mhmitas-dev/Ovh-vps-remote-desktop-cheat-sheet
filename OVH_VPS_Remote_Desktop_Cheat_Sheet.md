# OVH VPS Remote Desktop Cheat Sheet
**XFCE + TigerVNC + NoMachine + Chrome on Ubuntu 24.04 LTS**

---

## Prerequisites
- An OVH VPS (any plan) with Ubuntu 24.04 LTS installed
- A Windows (or any) machine with an SSH client and VNC Viewer
- VNC Viewer download: https://www.realvnc.com/en/connect/download/viewer/windows/
- NoMachine client download: https://www.nomachine.com/download (choose Windows)

---

## Step 1 — First SSH Connection

From your Windows PowerShell or Terminal:

```bash
ssh ubuntu@<YOUR_VPS_IP>
```

On first login, you'll be prompted to create a new password. Do so, then reconnect with the new password.

---

## Step 2 — Update the System

```bash
sudo apt update
sudo apt upgrade -y
```

Wait for this to finish completely before moving on.

---

## Step 3 — Install XFCE Desktop

```bash
sudo apt install -y xfce4 xfce4-goodies
```

During installation, if prompted to choose a display manager, select **lightdm**.

---

## Step 4 — Install TigerVNC and dbus-x11

Both are required. Install them together:

```bash
sudo apt install -y tigervnc-standalone-server tigervnc-common dbus-x11
```

> **Why dbus-x11?** Without it, the desktop throws "Unable to contact settings server" and "Failed to execute child process dbus-launch" errors.

---

## Step 5 — Set a VNC Password

```bash
vncpasswd
```

Enter a password (minimum 6 characters). This is separate from your SSH password. Decline the view-only password when prompted.

---

## Step 6 — Create the xstartup File

This tells VNC what to launch when a session starts.

```bash
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/bash
export XDG_SESSION_TYPE=x11
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec /usr/bin/xfce4-session
EOF

chmod +x ~/.vnc/xstartup
```

> **Key points:**
> - `exec` (not `&`) keeps the process in the foreground — required or VNC will think the session exited immediately.
> - The two `unset` lines prevent conflicts when running under systemd.

---

## Step 7 — Create the Systemd Service File

```bash
sudo tee /etc/systemd/system/vncserver@.service << 'EOF'
[Unit]
Description=VNC Server for display :%i
After=network.target

[Service]
Type=simple
User=ubuntu
ExecStart=/usr/bin/vncserver -fg -geometry 1920x1080 -depth 24 -localhost no :%i
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

> **Key points:**
> - `-fg` runs vncserver in the **foreground**. This is critical — without it, vncserver forks into the background and systemd thinks the process crashed.
> - `Type=simple` pairs with `-fg`. Do **not** use `Type=forking`.
> - `:%i` is a systemd placeholder — it gets replaced with the display number (e.g., `1`).
> - `-localhost no` allows connections from outside the server.
> - Change `1920x1080` to your preferred resolution if needed.

---

## Step 8 — Enable and Start the Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable vncserver@1.service
sudo systemctl start vncserver@1.service
sudo systemctl status vncserver@1.service
```

You should see **Active: active (running)** in the status output.

---

## Step 9 — Connect from Windows

1. Open RealVNC Viewer on your Windows machine.
2. Enter your VPS IP and port: `<YOUR_VPS_IP>::5901`
3. Enter the VNC password you set in Step 5.
4. You should see the XFCE desktop.

---

## Step 10 — Verify Auto-Start on Reboot

```bash
sudo reboot
```

Wait ~15–20 seconds, then reconnect via VNC Viewer. If it connects successfully, everything is working.

---

## Step 11 — Install NoMachine (for better performance)

NoMachine runs alongside VNC — it does not replace it. Both will share the same XFCE desktop session.

```bash
wget https://download.nomachine.com/download/8.20/Linux/nomachine_8.20.1_1_amd64.deb
sudo dpkg -i nomachine_8.20.1_1_amd64.deb
```

NoMachine auto-starts and auto-enables itself. Verify:

```bash
sudo systemctl status nxserver.service
```

You should see **Active: active (running)**. It listens on **port 4000** by default.

---

## Step 12 — Connect via NoMachine from Windows

1. Download and install the NoMachine client on your Windows machine: https://www.nomachine.com/download
2. Open NoMachine, click **Add**.
3. Enter your VPS IP: `213.32.23.2`
4. Port: `4000`, Protocol: **TCP**
5. Connect. It will auto-detect and attach to the existing XFCE session.

> NoMachine typically feels noticeably faster than VNC, especially for mouse and window movement. Keep both installed and use whichever feels better to you.

---

## Step 13 — Install Google Chrome

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
```

Open Chrome from the XFCE desktop or application menu inside your remote session.

---

## Quick Reference: Useful Commands

| What you need | Command |
|---|---|
| Start VNC | `sudo systemctl start vncserver@1.service` |
| Stop VNC | `sudo systemctl stop vncserver@1.service` |
| Restart VNC | `sudo systemctl restart vncserver@1.service` |
| Check VNC status | `sudo systemctl status vncserver@1.service` |
| Change VNC password | `vncpasswd` then `sudo systemctl restart vncserver@1.service` |
| Change resolution | Edit `-geometry` in the service file, then reload (see below) |
| Start NoMachine | `sudo systemctl start nxserver.service` |
| Stop NoMachine | `sudo systemctl stop nxserver.service` |
| Restart NoMachine | `sudo systemctl restart nxserver.service` |
| Check NoMachine status | `sudo systemctl status nxserver.service` |
| Remove NoMachine completely | `sudo systemctl stop nxserver && sudo apt remove -y nomachine` |

**To change resolution or any service setting:**
```bash
sudo nano /etc/systemd/system/vncserver@.service
# Edit the ExecStart line, save, then:
sudo systemctl daemon-reload
sudo systemctl restart vncserver@1.service
```

---

## Common Pitfalls and Fixes

| Problem | Cause | Fix |
|---|---|---|
| "Session exited too early" | xstartup uses `&` instead of `exec`, or missing `unset` lines | Rewrite xstartup exactly as in Step 6 |
| "Unable to contact settings server" / "dbus-launch not found" | `dbus-x11` not installed | `sudo apt install -y dbus-x11`, then restart VNC |
| Service times out on start | `Type=forking` used without `-fg`, or wrapper script exiting | Use `Type=simple` with `-fg` flag as in Step 7 |
| PID file not found | Systemd looking for wrong PID file path | Use `-fg` + `Type=simple` — no PID file needed |
| VNC connects but desktop is blank | Display manager conflict | Make sure lightdm is installed: `sudo apt install -y lightdm` |

---

*Tested on: OVH VPS 1, Ubuntu 24.04.3 LTS, January 2026*
