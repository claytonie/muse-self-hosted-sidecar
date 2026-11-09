# Self-Hosted AI Sidecar: Setup Guide

Build an always-on Windows desktop into a home AI sidecar: a headed Linux VM you can watch and drive from anywhere, plus a local AI agent that runs your scheduled jobs on your own hardware and answers on Telegram.

This guide is written from a working build (November 2026). It gives you the commands and the gotchas, not just the happy path.

## What you are building

```
                        Tailscale (private network, 100.64.0.0/10)
                                    |
   +----------------+               |               +------------------+
   | Your phone /   |               |               | Windows desktop  |
   | laptop         |---------------+---------------| (always on)      |
   +----------------+               |               |  - Tailscale     |
                                    |               |  - WSL2 Ubuntu   |
                                    |               |    - Hermes agent |
                                    |               |    - Ollama      |
                                    |               |  - VirtualBox VM |
                                    |               |    - Ubuntu +    |
                                    |               |      Chromium    |
                                    |               |      (headed,    |
                                    |               |       VNC/noVNC, |
                                    |               |       CDP)       |
                                    |               +------------------+
                                    |
                     +-----------------------------+
                     | Raspberry Pi (optional)     |
                     |  - data collector / sensors |
                     +-----------------------------+
```

Three pieces, each useful on its own:

1. **Headed Chromium VM.** An Ubuntu VM on the Windows host, running Chromium on a virtual display you can watch in any browser via noVNC, and drive remotely through Chrome DevTools Protocol (CDP). Use it for sites that block headless or cloud browsers, and to watch the agent work.
2. **Hermes agent in WSL2.** An AI agent (Hermes, from Nous Research) running inside WSL2 Ubuntu on the same Windows host, with your scheduled jobs and a Telegram gateway, using a local model (Ollama) where that is enough so you do not burn API credits.
3. **Remote recovery.** Tailscale plus a startup task so you can reach the Windows host after a reboot or power loss, without anyone typing a PIN.

## Prerequisites

- A Windows 10 or 11 desktop that stays on. 16 GB RAM and a few hundred GB free disk is a comfortable minimum.
- A Tailscale account and tailnet. Install Tailscale on the Windows host and on your phone and laptop.
- Administrator access on the Windows host.
- A Telegram bot token, if you want the agent to answer on Telegram.
- No purchases are required. Everything here uses free software.

## Part 1: Windows host, Tailscale, and power-loss recovery

### 1.1 Tailscale on the host

Install Tailscale on the Windows host and sign in. Note the host's Tailscale IP (it will be in the 100.x range). Enable the features you need in the Tailscale admin panel: subnet routes if you want to reach your home LAN, and exit node if you want it.

### 1.2 SSH into the host, tailnet-only

Enable OpenSSH Server on the Windows host and set its service to Automatic. Confirm the firewall rule for port 22 is scoped to the tailnet range (100.64.0.0/10) only, not to the whole LAN or the internet. You will use this SSH access for everything that follows.

From your own machine, SSH to the host over Tailscale. Ordinary networking does not reach the tailnet from a cloud VM; if you are working from one, route TCP through your egress proxy (change the proxy port to 3130 for Tailscale traffic) and use a ProxyCommand.

### 1.3 Reboot survival (the PIN problem)

A Windows desktop that requires a PIN or password at login is unreachable after a reboot until someone logs in, even though Tailscale and SSH services are set to Automatic. Fix that with an unattended Tailscale connection:

1. In the Tailscale admin panel, generate a reusable auth key. Note its expiry date and plan to rotate it.
2. Save the key to `C:\ProgramData\Tailscale\authkey.txt` on the host. Only the host needs to read it.
3. Create a PowerShell script that reads the key and runs `tailscale.exe up --authkey <key>` if the host is not already connected, and logs what it did.
4. Create a scheduled task (you must do this yourself; an agent generally cannot create scheduled tasks on your behalf) that runs that script at startup as SYSTEM. Trigger: AtStartup. Settings: allow start on batteries, do not stop on batteries, start when available.
5. Test it: reboot the host, then SSH in while it is still sitting at the login screen. If SSH works, recovery is proven.

### 1.4 Auto-start WSL and Ollama (optional but recommended)

WSL and Ollama only start at user logon, so after a reboot they are down even though Tailscale is up. Create two more startup scheduled tasks (again, create these yourself):

- **WSL:** action `wsl.exe -d Ubuntu -e sleep infinity`, trigger AtStartup, run as your user, execution time limit zero. The `sleep infinity` keeps WSL alive; services inside it then start themselves.
- **Ollama:** action `powershell.exe` with a command that sets `OLLAMA_HOST` to `0.0.0.0:11434` and runs `ollama.exe serve`, trigger AtStartup, run as your user, execution time limit zero.

If WSL does not come up under a scheduled task, recreate the task in Task Scheduler with "Run whether user is logged on or not" and your password entered there.

**Launch long-running servers with WMI, not Start-Process.** A process started with `Start-Process` over SSH dies when the SSH session ends. Launch it with `Invoke-CimMethod Win32_Process Create` so it survives disconnects.

## Part 2: Ubuntu VM with headed Chromium

### 2.1 VirtualBox and the VM

Install VirtualBox on the Windows host (winget works). Download an Ubuntu Desktop ISO (24.04 or newer; this build used 26.04.1) and verify its SHA256 against the published hash before you install.

Create the VM:

- Name: `Ubuntu-Chromium` (or your choice)
- Memory: 4096 MB, CPUs: 2, VRAM: 128 MB, graphics: VMSVGA
- Disk: 40 GB VDI on SATA
- Network: NAT
- Audio and USB off

Install Ubuntu unattended with `VBoxManage unattended install`, using a hostname that includes a domain (for example `ubuntu-chromium.local`), your timezone, and `--install-additions` so Guest Additions are present. Set a strong VM password and change it again if you ever share it.

### 2.2 Tailscale inside the VM

Copy a Tailscale auth key into the VM (file copy via VirtualBox guestcontrol works). In a VM terminal, install Tailscale with the official install script, then join with the auth key and a hostname:

```
sudo tailscale up --authkey <key> --hostname ubuntu-chromium
```

Delete the key file from the VM after joining. The VM now has its own Tailscale IP.

**Install curl first.** A fresh Ubuntu install does not ship with `curl`, and the Tailscale install script needs it. Run `sudo apt update && sudo apt install -y curl` before anything else.

### 2.3 The headed desktop: Xvfb, Openbox, x11vnc, noVNC

The VM's GNOME session runs on Wayland by default, and x11vnc cannot capture a Wayland session. So run a separate X11 desktop on a virtual display:

- **Xvfb :99** at 1920x1080x24.
- **Openbox** as the window manager on :99.
- **x11vnc** attached to :99 on port 5900.
- **websockify** serving the noVNC web client on port 6080, proxying to localhost:5900.

Put all of this in a script (for example `~/.vnc/run-vm-desktop.sh`) and add a `.desktop` file in `~/.config/autostart/` so it starts at login.

**Critical: strip the Wayland variables.** Ubuntu sets `WAYLAND_DISPLAY` (and `XDG_SESSION_TYPE`) in terminal sessions. x11vnc exits immediately if it sees `WAYLAND_DISPLAY`, even when you pass `-display :99`. Launch x11vnc with `env -u WAYLAND_DISPLAY -u XDG_SESSION_TYPE`.

**Same for Chromium.** The snap Chromium ignores `--display=:99` when `WAYLAND_DISPLAY` is set, and opens on the GNOME session instead. Then the Xvfb display is black in VNC even though the browser is running. Launch Chromium with the same variables stripped.

### 2.4 Chromium remote debugging on the Tailscale address

Start Chromium on :99 with remote debugging:

```
chromium-browser --display=:99 --remote-debugging-address=<tailscale-ip> --remote-debugging-port=9222 --no-first-run --disable-dev-shm-usage about:blank
```

**The snap Chromium ignores `--remote-debugging-address`.** It binds DevTools to 127.0.0.1 only. Forward the Tailscale address to localhost with socat:

```
socat TCP-LISTEN:9222,bind=<tailscale-ip>,fork TCP:127.0.0.1:9222
```

Verify from outside the VM: `http://<tailscale-ip>:9222/json/version` should return the Chrome version and protocol version, and `http://<tailscale-ip>:6080/vnc.html` should load the noVNC page.

**Wait for the Tailscale IP before starting Chromium.** If the desktop script reads the Tailscale IP before the interface is ready, it gets an empty value and debugging binds to localhost. Loop until `tailscale ip -4` returns a value (up to 60 seconds) before launching Chromium, and log the address used.

### 2.5 VNC password, or no password

You can set a VNC password with `x11vnc -storepasswd <password> ~/.vnc/passwd` and pass `-passwdfile ~/.vnc/passwd` to x11vnc. Note that VNC passwords are truncated to 8 characters.

In this build, password authentication kept failing even with freshly stored passwords, for a cause never established. The working configuration runs x11vnc with `-nopw` (no password). That is acceptable here because the VM is reachable only over the Tailscale network, so the tailnet itself is the access control. If you use a password, verify it works from a real client before you rely on it, and re-enable it once you know why authentication failed.

**If the desktop script still uses a password file, a reboot or logout reverts to password auth.** Make the startup script and the running server agree, or you will be locked out after a restart.

## Part 3: Hermes agent in WSL2 (optional)

This part is optional. It gives you an AI agent on the same Windows host that can run scheduled jobs and answer on Telegram, using a local model where that is enough.

### 3.1 WSL2 with mirrored networking

Enable the WSL and Virtual Machine Platform features, then install Ubuntu in WSL2. Create your user and set a password yourself (do not pass a password through an agent). Use mirrored networking so WSL shares the Windows host's Tailscale interface and does not become a second tailnet node: add `networkingMode=mirrored` under `[wsl2]` in `C:\Users\<you>\.wslconfig`, then `wsl --shutdown`.

Do not also install Tailscale inside WSL2 and join it as a second node. Pick one mode (mirrored is cleaner on a desktop you control) before you copy files or set up the agent.

**Check virtualization with `systeminfo`, not with the firmware flag.** `Win32_Processor.VirtualizationFirmwareEnabled` returns False whenever a hypervisor is already active, because the firmware flag is hidden once the hypervisor owns it. `systeminfo` showing "A hypervisor has been detected" means virtualization is on.

### 3.2 Install Hermes and migrate from OpenClaw

Install Hermes (Nous Research) in WSL2 with the Linux installer. If you have an existing OpenClaw setup (for example on a Raspberry Pi), copy `~/.openclaw` into the WSL2 home directory before you run the Hermes setup, so the wizard can offer to migrate it. Use `rsync` over Tailscale for the copy.

Install the `agent-importers` plugin, then run the migration with secrets and your workspace target. Cron jobs do not migrate automatically if they live outside the migrator's expected location; recreate them by hand.

### 3.3 Scheduled jobs and the Telegram gateway

Recreate your scheduled jobs in Hermes (for example a data-collection job every few minutes and daily/weekly/monthly digests). Install the Telegram gateway as a user systemd service with linger enabled, so it starts when WSL starts.

**Give the bot to one gateway only.** If two gateways poll the same Telegram bot token, they fight each other and you get "Conflict: terminated by other getUpdates request". Pause or stop the old gateway so only Hermes owns the bot. If you keep the old system running as a fallback, keep its data collection (cron) but stop its gateway.

**Pin routine jobs to the local model.** Set the jobs and the gateway default to your Ollama model (for example `llama3.1:8b`) so routine digests and Telegram replies do not burn API credits. Use the plain model name (`llama3.1:8b`), not a prefixed name, or the model lookup fails.

### 3.4 Local models with Ollama

Install Ollama on the Windows host. Bind it to `0.0.0.0:11434` (not to a single IP) so WSL can reach it, and add a Windows Firewall rule that allows port 11434 only from the tailnet range (100.64.0.0/10) and loopback. From inside WSL, reach Ollama at `http://127.0.0.1:11434` (mirrored networking makes the host's loopback reachable).

**Ollama and Tailscale hairpin:** from inside WSL, connecting to the host's own Tailscale IP on port 11434 times out. Use the loopback address instead.

## Part 4: Controlling the VM browser from anywhere (CDP)

Once the VM is up, you can drive its Chromium from any machine on the tailnet through the Chrome DevTools Protocol.

- List tabs: `GET http://<vm-tailscale-ip>:9222/json/list`
- Open a page: `PUT http://<vm-tailscale-ip>:9222/json/new?{url-encoded-url}`
- Read a page and take a screenshot: connect to the tab's `webSocketDebuggerUrl` and use `Runtime.evaluate` and `Page.captureScreenshot`.

**WebSocket Origin.** Chrome rejects the DevTools WebSocket handshake when the Origin header is the Tailscale address. Suppress the Origin header in your WebSocket client (for example `suppress_origin=True` in Python's websocket-client), or launch Chromium with `--remote-allow-origins=*`.

A reusable helper script for this is in the companion skill (see "Further reading").

## Part 5: When you cannot run a command in the VM

VirtualBox `guestcontrol run` and `guestcontrol start` do not execute processes in the guest on this setup (they return `VERR_NOT_IMPLEMENTED`). File copy in and out (`guestcontrol copyto` / `copyfrom`) does work, and it authenticates with the VM user's password.

If you need to run a command in the VM and have no SSH server there, drive the keyboard:

1. Take a screenshot with `VBoxManage controlvm <vm> screenshotpng <file>` to see the screen state.
2. If the screen is locked, send a key to wake it, type the VM password with `keyboardputstring`, and press Enter.
3. Open a terminal with Ctrl+Alt+T, or click the terminal in the dock.
4. Type the command with `keyboardputstring` and press Enter with `keyboardputscancode 1c 9c`.
5. Screenshot again to read the result.

Watch out: if a terminal has a foreground process running (for example a server started without `&`), typed text goes nowhere and Enter does nothing. Open a fresh terminal in that case.

**Installing an SSH server in the VM is the better long-term fix.** Ubuntu Desktop does not install one by default. Add `openssh-server` so you can SSH to the VM over Tailscale directly.

## Troubleshooting

- **noVNC page loads but "Failed to connect to server".** The VNC server (x11vnc) is not running or not attached to the display. Check the x11vnc log for "Wayland display server detected" and strip the Wayland variables.
- **noVNC shows a black screen with a cursor.** Chromium opened on the GNOME/Wayland session, not on the Xvfb display. Strip `WAYLAND_DISPLAY` when launching Chromium so it renders on :99.
- **CDP returns empty or connection refused.** The snap Chromium binds DevTools to 127.0.0.1. Use the socat forward, and confirm the Tailscale IP was known when Chromium started.
- **WebSocket CDP handshake 403.** Suppress the Origin header, or add `--remote-allow-origins=*` to Chromium.
- **Telegram "Conflict: terminated by other getUpdates request".** Two gateways are polling the same bot token. Stop one of them.
- **Host/VM briefly unreachable, then recovers.** Can be a transient network or Tailscale blip. Retry after 20 to 30 seconds before assuming something is down.
- **VNC password check fails with a freshly stored password.** Cause unknown in this build. The workaround is `-nopw` behind the tailnet. Do not keep guessing at passwords.

## Security notes

- Nothing here is exposed to the internet. Tailscale is the only way in. Keep it that way: do not port-forward the VNC, CDP or Ollama ports on your router.
- The VNC server, when run without a password, relies entirely on the tailnet for access control. Only do this on a private tailnet you trust.
- Auth keys and passwords live on the host, never in a chat log or a shared document. Rotate the Tailscale auth key when it expires.
- Change the VM password again if you ever shared it, even for one-time setup use.
- The Windows host is the single point of failure for this whole setup. Keep it powered on and awake; do not let it sleep.

## Prompts to give your Muse

Copy and paste these into your own Muse to build the same setup. Fill in the bracketed parts with your own details. The first prompt starts the whole build; the others cover each piece on its own if you only want part of it.

### Master prompt: the whole sidecar

```
Set up a self-hosted sidecar environment on my Windows desktop [hostname].
Work step by step, confirm each step works before moving on, and ask me before
anything that costs money or needs credentials. If a step is outside what you
can do, say so plainly rather than attempting a workaround.

1. A Linux VM (Debian or Ubuntu, my preference is [Ubuntu/Debian]) on this
   Windows host, joined to my Tailscale network, with Chromium running headed
   under a persistent desktop session (Xvfb / lightweight window manager +
   VNC / noVNC) that I can watch at the VM's Tailscale IP, and Chromium's
   remote-debugging port exposed on the Tailscale interface only.

2. Hermes Agent (Nous Research) in WSL2 on this host (Linux installer, not the
   Windows build), with Tailscale access to WSL. If I have an existing OpenClaw
   setup on [my Raspberry Pi / another machine], copy ~/.openclaw from there
   before setup and run `hermes claw migrate`, confirming my scheduled jobs
   survive. Put the Telegram gateway on my existing bot [bot name], and leave
   the old gateway running on the Pi until I verify Hermes for a week and
   decommission it myself. Keep everything under ~/.hermes so I can rsync it
   to another machine later.

3. An agent-to-agent channel: Tailscale SSH from you (Muse) to the Hermes
   WSL2 instance, with every transcript logged for my review. Plus a profile
   document describing how I like to work, kept current and persisted.
```

### Prompt: headed Chromium VM only

```
Set up a Linux VM on my Windows desktop [hostname] with VirtualBox, joined to
my Tailscale network, running headed Chromium on a persistent virtual desktop
that I can watch in a browser via noVNC at the VM's Tailscale IP, with
Chromium's remote-debugging port (CDP) exposed on the Tailscale interface only.
Work step by step and confirm each step works before moving on. Ask me before
anything that costs money or needs credentials. If a step is outside what you
can do, say so plainly rather than attempting a workaround.
```

### Prompt: Hermes agent in WSL2 only

```
Set up Hermes Agent (Nous Research) in WSL2 Ubuntu on my Windows desktop
[hostname], using the Linux installer, not the Windows build. Give WSL
Tailscale access (mirrored networking preferred). If I have an existing
OpenClaw setup on [my Raspberry Pi / another machine], copy ~/.openclaw from
there first and run `hermes claw migrate`, confirming my scheduled jobs
survive. Put the Telegram gateway on my existing bot [bot name]. Pin routine
jobs and Telegram replies to my local Ollama model [e.g. llama3.1:8b] where
that is enough, so I do not burn API credits. Work step by step and confirm
each step works before moving on.
```

### Prompt: power-loss recovery only

```
Make my Windows desktop [hostname] recoverable after a power loss or reboot
with no one at the keyboard. Tailscale must reconnect at the login screen so
you can SSH in over the tailnet without anyone typing a PIN, and WSL plus
Ollama must come back up too. I will create the scheduled tasks myself from
steps you give me; do not create them on my behalf. Test it with a reboot and
confirm SSH works while the machine is still at the login screen.
```

### Prompt: drive the VM browser via CDP

```
Using the Chrome DevTools Protocol, read the page currently open in the VM
Chromium at [VM Tailscale IP]:9222 and tell me the product title, brand,
price and seller. If the page is blocked in your own browser, read it from
the VM browser instead. Do not sign in to anything, add anything to a cart,
or buy anything.
```

## Further reading

- The companion reusable skill for driving the VM browser through CDP, including a helper script: `vm-chromium-cdp` skill.
- Hermes Agent docs: https://hermes-agent.nousresearch.com/docs
- Tailscale docs: https://tailscale.com/kb/
- VirtualBox docs: https://www.virtualbox.org/wiki/Documentation
- noVNC: https://novnc.com/
