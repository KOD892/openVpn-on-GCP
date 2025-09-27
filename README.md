# openVpn-on-GCP

## Abstract

This report details the process of deploying a personal OpenVPN server on the Google Cloud Platform (GCP). The primary objective was to establish a secure, private, and encrypted network connection by routing traffic through a dedicated Virtual Machine (VM). The project involved provisioning a VM, configuring network settings, installing OpenVPN server software, and verifying the connection with a client. All objectives were successfully met, resulting in a functional and secure VPN tunnel.

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
 <p align="center">
  <img src="./screenshots/set region and zone vars.png" alt="Setting Region and Zone" width="70%">
  <br>
  <em>Figure 1: Setting region and zone variables.</em>
</p>


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
<p align="center">
  <img src="./screenshots/vm setup complete edited.png" alt="VM Setup Complete" width="70%">
  <br>
  <em>Figure 2: VM instance `ovpntest` successfully created.</em>
</p>
---

## STEP 2: IP Address Assignment & Firewall Setup

1. **Create a Static External IP Address**  
   This reserves an IP address that won't change when the VM restarts.
   ```bash
   gcloud compute addresses create ovpn-static-address --region=$REGION
   gcloud compute addresses list
   ```
   Take note of the reserved IP address.
   <p align="center">
  <img src="./screenshots/check reserved address addre.png" alt="Reserved Static IP" width="70%">
  <br>
  <em>Figure 3: Static IP address reserved for the VPN server.</em>
</p>

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
<p align="center">
  <img src="./screenshots/firewall rule created.png" alt="Firewall Rule" width="70%">
  <br>
  <em>Figure 4: Firewall rule created to allow OpenVPN traffic.</em>
</p>

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
  <p align="center">
  <img src="./screenshots/install begin.png" alt="OpenVPN Installation" width="70%">
  <br>
  <em>Figure 5: Initiating the OpenVPN installation script.</em>
</p>
   - At the end, note the path to the generated `.ovpn` client configuration file.

3. **Verify OpenVPN is Running**
   ```bash
   sudo service openvpn status
   ```
   - The status should indicate that OpenVPN is active/running.
  <p align="center">
  <img src="./screenshots/openvpn status.png" alt="OpenVPN Status" width="70%">
  <br>
  <em>Figure 6: Verifying that the OpenVPN service is active.</em>
</p>

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
 <p align="center">
  <img src="./screenshots/config file downloaded.png" alt="Downloading Client Config" width="70%">
  <br>
  <em>Figure 7: Transferring the .ovpn configuration file.</em>
</p>

2. **Download to Your Client Device**  
   Transfer the `.ovpn` file to your client device (e.g. Parrot OS, Windows, Mac).  
   You can do this via the Cloud Shell interface.
    

3. **Check current IP address and location** 
   - Before connecting to the vpn server I first check my ip address and location using [what is my ip address](https://whatismyipaddress.com/)
   This confirms my current Location and ISP.
    <p align="center">
  <img src="./screenshots/my ip before running ovpn edited.png" alt="IP Before VPN" width="70%">
  <br>
  <em>Figure 8: Public IP address before establishing the VPN connection.</em>
</p>

4. **Import the .ovpn File into Your Client**  
   - Use [OpenVPN GUI](https://openvpn.net/community-downloads/) or [NetworkManager](https://wiki.archlinux.org/title/NetworkManager#OpenVPN) depending on your OS. 
```bash
 sudo openvpn client.ovpn
```
   - Connect and verify that your client receives an IP address in the VPN subnet (usually `10.8.0.x`).
   - Test connectivity to the internet and check your IP address and location using [what is my ip address](https://whatismyipaddress.com/)
  <p align="center">
  <img src="./screenshots/ip successful edited.png" alt="IP After VPN" width="70%">
  <br>
  <em>Figure 9: Public IP address after connecting, showing the server's IP.</em>
</p>

---

## 4. Conclusion

The deployment of the OpenVPN server on GCP was successful. By following a structured approach of provisioning infrastructure, installing server software, and configuring client access, a secure and private network tunnel was created. This setup provides an effective means of encrypting internet traffic and masking the client's original IP address.

## Troubleshooting & Notes

- **GCP Quotas**: Ensure project quotas are sufficient for the desired resources.
- **Firewall Rules**: Incorrect or missing firewall rules are a common cause of connection issues. Verify that UDP port 1194 is open to the correct network tag.
- **Network Interface**: The NAT rule assumes the primary network interface is `ens4`. This should be verified with `ip a` and adjusted if necessary.
- **Client Management**: The installation script can be re-run to add or revoke client configuration files.
- **Security**: For production environments, further security hardening is recommended, such as implementing `fail2ban`, disabling root SSH login, and regularly updating the system.

---

## References

- [Google Cloud Compute Engine Documentation](https://cloud.google.com/compute/docs)
- [Nyr OpenVPN Install Script](https://github.com/Nyr/openvpn-install)
- [OpenVPN Community Downloads](https://openvpn.net/community-downloads/)
