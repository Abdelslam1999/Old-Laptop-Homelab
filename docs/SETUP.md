# Setup Guide

Step-by-step notes on turning the Dell Inspiron 15 into a headless Ubuntu homelab server.

## 1. Prepare the hardware

- Repurposed an old Dell Inspiron 15 laptop no longer in daily use.
- Ran it with the lid closed, configured to stay awake on lid-close (`HandleLidSwitch=ignore` in `/etc/systemd/logind.conf`) so it behaves as a headless appliance rather than a laptop.

## 2. Install Ubuntu Server 24.04.4 LTS

- Flashed Ubuntu Server (not Desktop) to a USB installer.
- Installed with OpenSSH server enabled during setup so SSH access works immediately after first boot — no monitor/keyboard needed afterward.
- Created a non-root administrative user (`monk`) rather than operating as `root` day-to-day.

## 3. Reserve a static IP

Rather than configuring a static IP directly on the machine (via netplan), the IP was reserved at the **router level**:

1. Logged into the home router's admin panel.
2. Located the DHCP client list and found the server's MAC address.
3. Created a DHCP reservation binding that MAC address to a fixed LAN IP.
4. Rebooted the server to confirm it always receives the same address.

This approach keeps IP management centralized at the router and avoids editing netplan YAML on every OS reinstall.

## 4. Install Docker & Docker Compose

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

## 5. Deploy Portainer

Portainer was deployed first as the central management layer for every other container:

```bash
docker volume create portainer_data
docker run -d -p 9000:9000 --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce
```

From here on, additional stacks (n8n, Netdata, Pi-hole) were deployed through the Portainer web UI under **Stacks → Add Stack**.

## 6. Deploy the remaining services

| Service | Deployment notes |
|---|---|
| n8n | Deployed as a stack, published on port `5678`, used for building automation/AI-agent workflows |
| Netdata | Deployed with access to the Docker socket and host `/proc`, `/sys` mounts for full system + container-level metrics, published on `19999` |
| Pi-hole | Deployed with a dedicated static container IP on the Docker network so it can reliably serve as the LAN's DNS resolver |

## 7. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Authenticated the device through the Tailscale admin console, which added it to the private tailnet alongside other personal devices (laptop, phone). From that point on, the server is reachable via its Tailscale IP from anywhere, without any router port forwarding.

## 8. Harden SSH access

- Restricted SSH login to key-based auth where possible.
- Added a login warning banner (`AUTHORIZED ACCESS ONLY — ALL ACTIVITY IS LOGGED & MONITORED`) via `/etc/update-motd.d/`.
- Verified `Last login` tracking to monitor for unexpected access.
