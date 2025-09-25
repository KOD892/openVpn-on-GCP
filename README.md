# openVpn-on-GCP

This guide provides step-by-step instructions for setting up a personal OpenVPN server on Google Cloud Platform (GCP).  
It covers provisioning a VM, assigning a static IP, setting up firewall rules, installing OpenVPN, configuring networking, and retrieving the client configuration.

---

## STEP 1: Cloud VM Setup

1. **Create a GCP Project**  
   If you haven't already, create a Google Cloud project from the [Google Cloud Console](https://console.cloud.google.com/).

2. **Open Cloud Shell**  
   Launch Google Cloud Shell from the console for command-line access.

3. **Set Region and Zone**  
   For consistency, set your desired region and zone. Example uses `us-central1` and `us-central1-a`:
   ```bash
   gcloud config set compute/region us-central1
   export REGION=us-central1
   export ZONE=us-central1-a
   ```
   <figure>
  <img src="./screenshots/set region and zone vars.png" alt="Alt text">
  <figcaption style="font-size:8px; text-align: center">Setting Region and Zone</figcaption>
</figure>


4. **Create the VM Instance**  
   Create a VM named `ovpntest` with a lightweight Ubuntu image and enable IP forwarding:
   ```bash
   gcloud compute instances create ovpntest \
     --zone=$ZONE \
     --machine-type=e2-micro \
     --image-family=ubuntu-2204-lts --image-project=ubuntu-os-cloud \
     --can-ip-forward \
     --tags openvpn-server
   ```
   - `--can-ip-forward` allows the VM to route packets (needed for VPN).
   - `--tags openvpn-server` helps identify the VM and is used for firewall rules.
<figure>
  <img src="./screenshots/vm setup complete edited.png" alt="Alt text">
  <figcaption style="font-size:8px;text-align: center ">VM Setup Complete</figcaption>
</figure>
---

## STEP 2: IP Address Assignment & Firewall Setup

1. **Create a Static External IP Address**  
   This reserves an IP address that won't change when the VM restarts.
   ```bash
   gcloud compute addresses create ovpn-static-address --region=$REGION
   gcloud compute addresses list
   ```
   Take note of the reserved IP address.
   <figure>
  <img src="./screenshots/check reserved address addre.png" alt="Alt text">
  <figcaption style="font-size:8px; text-align: center">Reserved Address</figcaption>
</figure>

2. **Remove Temporary External IP**  
   When first created, the VM has an ephemeral external IP. Remove it:
   ```bash
   gcloud compute instances delete-access-config ovpntest \
     --zone=$ZONE --access-config-name="external-nat"
   ```

3. **Assign the Static IP to the VM**  
   Replace `<ipv4 address>` with the actual reserved IP:
   ```bash
   gcloud compute instances add-access-config ovpntest \
     --zone=$ZONE --address=<ipv4 address>
   ```

4. **Create a Firewall Rule for OpenVPN**  
   OpenVPN uses UDP port 1194 by default, which is preferred over TCP for VPNs due to lower latency and better performance for real-time traffic.
   ```bash
   gcloud compute firewall-rules create allow-openvpn \
     --allow udp:1194 \
     --target-tags=openvpn-server \
     --description="Allow OpenVPN UDP 1194"
   ```
   <figure>
  <img src="./screenshots/firewall rule created.png" alt="Alt text">
  <figcaption style="font-size:8px; text-align: center">Creating Firewall Rule</figcaption>
</figure>

---

## STEP 3: SSH to VM & Install OpenVPN (Nyr Quick Installer)

1. **SSH into the VM**  
   ```bash
   gcloud compute ssh ovpntest --zone=$ZONE
   ```
   - `ovpntest` is the VM name.
   - You may be prompted for a password; you can leave it blank or set one.

2. **Install OpenVPN Using Nyr's Script**  
   This script simplifies the setup and configuration process. Accept defaults unless you have custom requirements.
   ```bash
   wget https://git.io/vpn -O openvpn-install.sh && sudo bash openvpn-install.sh
   ```
    <figure>
  <img src="./screenshots/install begin.png" alt="Alt text">
  <figcaption style="font-size:8px; text-align: center">OpenVPN Install Begin</figcaption>
</figure>
   - At the end, note the path to the generated `.ovpn` client configuration file.

3. **Verify OpenVPN is Running**
   ```bash
   sudo service openvpn status
   ```
   - The status should indicate that OpenVPN is active/running.
    <figure>
  <img src="./screenshots/openvpn status.png" alt="Alt text">
  <figcaption style="font-size:8px; text-align: center">openvpn status</figcaption>
</figure>

4. **Enable IP Forwarding & Configure NAT**  
   This allows VPN clients to route traffic through the VM to the internet.

   - Enable IP forwarding (needed for routing):
     ```bash
     sudo sysctl -w net.ipv4.ip_forward=1
     echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
     ```

   - Set up NAT so VPN clients' traffic is masqueraded out your network interface (`ens4` is typical for GCP VMs, but verify with `ip a`):
     ```bash
     sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o ens4 -j MASQUERADE
     ```

   - Make iptables rules persistent (so they survive reboot):
     ```bash
     sudo apt update && sudo apt install -y iptables-persistent
     sudo netfilter-persistent save
     ```

---

## STEP 4: Retrieve .ovpn Client File & Test VPN

1. **Copy the Client Config to Cloud Shell**  
   Replace `<.ovpn file path>` with the actual path output by the installer, e.g. `/root/client.ovpn`:
   ```bash
   exit
   gcloud compute scp ovpntest:<.ovpn file path> ./ --zone=$ZONE
   ```
    <figure>
  <img src="./screenshots/config file downloaded.png" alt="Alt text">
  <figcaption style="font-size:8px; text-align: center">Copying Client Config to Cloud Shell</figcaption>
</figure>

2. **Download to Your Client Device**  
   Transfer the `.ovpn` file to your client device (e.g. Parrot OS, Windows, Mac).  
   You can do this via the Cloud Shell interface.
    <figure>
  <img src="./screenshots/download file 1.png" alt="Alt text">
  <figcaption>Click on options</figcaption>
</figure>
 <figure>
  <img src="./screenshots/download file 2.png" alt="Alt text">
  <figcaption style="font-size:8px;text-align: center">Click on download</figcaption>
</figure>

3. **Import the .ovpn File into Your Client**  
   - Use [OpenVPN GUI](https://openvpn.net/community-downloads/) or [NetworkManager](https://wiki.archlinux.org/title/NetworkManager#OpenVPN) depending on your OS.
   - Connect and verify that your client receives an IP address in the VPN subnet (usually `10.8.0.x`).
   - Test connectivity to the internet and internal resources as needed.

---

## Troubleshooting & Notes

- **Check your GCP quotas**: Free-tier VMs may be limited.
- **Firewall issues**: Ensure UDP 1194 is open and attached to the VM's network tag.
- **Network interface name**: If `ens4` is not your VM's primary interface, adjust iptables rules accordingly.
- **Revoke or add VPN clients**: Re-run the installer script on the VM to manage client configs.
- **Security**: Change default passwords and consider additional hardening (fail2ban, disabling root SSH, etc).

---

## References

- [Google Cloud Compute Engine Documentation](https://cloud.google.com/compute/docs)
- [Nyr OpenVPN Install Script](https://github.com/Nyr/openvpn-install)
- [OpenVPN Community Downloads](https://openvpn.net/community-downloads/)
