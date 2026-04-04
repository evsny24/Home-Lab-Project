# Hardware Setup

- Installed a spare SSD into my desktop system to dedicate to Proxmox installation.

- Ensured the system BIOS recognized the SSD and set the appropriate boot order.

  - Required careful handling to avoid overwriting existing data on my main SSD.


# Proxmox Installation

- Installed Proxmox Virtual Environment (VE) on the dedicated SSD.

- Configured VM network interfaces and checked IP addresses, gateways, and routing using `ip a/route`

- Encountered failed `apt-get update` after installation due to Linux package management errors from unauthorized IP access and unsigned repository metadata

  - Disabled the paid enterprise source repo and added a custom source list for the no-subscription repo instead

    - `echo "deb http://proxmox.com trixie pve-no-subscription" > /etc/apt/sources.list.d/pve-no-sub.list`



# Virtual Machine Creation

- Created three virtual machines (VMs) in Proxmox, including Kali, Ubuntu, and Security Onion.

- Allocated CPU cores and RAM based on VM requirements (1–2 cores per VM, 8GB-16GB ram each).

- Encountered vncproxy errors when attempting to launch VMs.

  - Resolved by adjusting the video adapter to Standard VGA with `qm set 102 --vga std`



# Security Onion and Network Interface Card configurations

- Added two VirtIO NICs for my Security Onion VM: one for management, one for monitoring/sniffing traffic.

- Management NIC connected to main Proxmox bridge `vmbr0` for network access.

- Monitoring NIC connected to the same bridge to capture traffic from other VMs.

- Installer displayed warning: “The IP being routed by Linux is not the IP assigned to the management interface.”

  - Resolved by manually assigning a static IP to the management NIC `ens19` and leaving monitoring NIC (ens18) without an IP:

    - `sudo ip addr add 192.168.0.150/24 dev ens18`

    - `sudo ip link set ens18 up`

    - `sudo ip route add default via 192.168.0.1`



# Security Onion Console Dashboard Access

- After a successful installation of Security Onion, I had trouble accessing the SOC. I fixed this after much testing.

  - Checked that all Security Onion services were running correctly.

    - `sudo so-status`

  - Verified that required ports were listening to confirm web services were active.

    - `sudo ss -tulnp grep | 443`

  - Inspected firewall rules with iptables to ensure no traffic was being blocked.

    - `sudo iptables -L | less`

  - Tested dashboard access using curl and confirmed HTTP/HTTPS responses.

    - `curl -kL https://192.168.0.150/`

  - Identified that browser access was blocked due to firewall restrictions.

    - Whitelisted my machines through the firewall to allow SOC dashboard access.

      - `sudo so-firewall includehost analyst 192.168.0.x`

- Ensured the functionality of Security Onion services by creating and triggering detections in the dashboard.

  - Wrote my own Sigma rule to detect when nmap scans are used to scan the endpoints and successfully triggered it by running an `nmap` scan.



# Adding Security Onion Endpoints

- My elastic agent installer was not working, and the log file described that the connectivity failed to Fleet Host.

  - Configured firewall hostgroups within the Security Onion framework to authorize specific subnets, ensuring secure and reliable log ingestion.

- Managed Elastic Fleet server configurations to ensure endpoint agents could successfully enroll and communicate over required TCP ports.

  - `curl -k https://192.168.0.150:8220`

- Validating successful data flow into the centralized management interface and identifying agent hostnames.



# WireGuard VPN Deployment \& Secure Configuration

- Installed WireGuard on a Debian XLS container; wrote config file for `wg0.conf` with internal VPN subnet `10.0.0.1/24`.

- Applied preshared keys in `[Peer]` sections to ensure cryptographic integrity.

- Implemented firewall rules restricting VPN traffic to only required hosts and ports by the concept of least privilege

  - `iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT`

  - `iptables -A FORWARD -i wg0 -d 192.168.0.58 -p tcp --dport 8006 -j ACCEPT`

  - `iptables -A FORWARD -i wg0 -d 192.168.0.150 -j ACCEPT`

  - `iptables -A FORWARD -i wg0 -j REJECT`

  - `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`

- Traffic not received by VPN client

  - Verified MTU (1420) and NAT configuration; adjusted forwarding rules.

- Ensured IP forwarding enabled; monitored UDP GRO warnings, and confirmed stability of tunnel.

  - `echo "net.ipv4.ip\_forward=1" >> /etc/sysctl.conf`

  - `echo "net.ipv6.conf.all.forwarding=1" >> /etc/sysctl.conf`

- Unable to forward ports externally; confirmed VPN still allowed internal routing through container and LAN.

  - Pivoted to using Tailscale to bypass the need for port forwarding



# Tailscale VPN Deployment \& Network Integration

- Created a secure Debian container in Proxmox VE.

  - applied least-privilege access by creating a non-root user and disabling unnecessary services.

- Installed Tailscale on Debian container, authenticated via CLI, and verified node connectivity.

- Monitored key expiry and ensured device authorization before enabling routes

  - `tailscale up --advertise-routes=192.168.0.0/24`

- Confirmed laptop and other endpoints added to Tailscale network; ensured persistent Tailscale service post-reboot.

  - `systemctl enable tailscaled`

- Achieved secure remote access to Proxmox host, VMs, and the Security Onion Console internally without exposing services directly to the internet.



# Contingency planning

- Engineered a secure, multi-stage solution to remotely power on a Proxmox server via the internet using a low-power laptop as a local network relay.

  - Developed and deployed a custom PowerShell "Magic Packet" sender on Windows to trigger hardware wake-up events on Proxmox host.

  - Performed BIOS/UEFI power management adjustments and Linux kernel-level network interface tuning `ethtool`.

- Ensured system configurations were permanent by editing config files rather than temporary bypasses; enabled the VPN container to start on boot.

- The Proxmox PC would lose its Wake-on-LAN settings every time it shut down.

  - Modified `/etc/network/interface`s with `post-up`  to force the `wol g` flag to remain active on every boot.

- My laptop relay was sending packets, but the Proxmox PC stayed off due to power-saving features.

  - Disabled `ErP Ready` and `Deep Sleep` in the BIOS to keep the network card powered while the system was off.

- Attempted passwordless SSH for "one-click" ease

  - Chose to retain standard SSH authentication (password-based) to ensure every remote wake event requires manual authorization.

- Windows blocked the custom wake script from running remotely via SSH.

  - Implemented an `-ExecutionPolicy Bypass` flag in the remote command string to allow the automation to run without lowering global system security.

