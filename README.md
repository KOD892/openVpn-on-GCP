# openVpn-on-GCP
This guide documents how I set up a personal OpenVPN server on Google Cloud Platform  
## STEP 1: Cloud VM Setup
1. open a gcp project and open cloudshell
2. on cloudshell set region, i used us-central1
     
```bash 
    gcloud config compute/region us-central1
```
    
2.next set up variables for regon and zone, this is done to have a consistent value for both
   commands: 
```bash
   export REGION=us-central1; export ZONE=us-central1-a
```
3. create vm instance using the commnad:
```bash
    gcloud compute instances create ovpntest \
     --zone=$ZONE \
     --machine-type=e2-micro \
     --image-family=ubuntu-2204-lts --image-project=ubuntu-os-cloud \
     --can-ip-forward \
     --tags openvpn-server
```
## STEP 2: IP address assignment/ firewall rule settup
1. create static ip address using the command below:
```bash 
    gcloud compute addresses create < adress name | ovpn-static-address> --region=$REGION
    #to view the created ip address run the command
    gcloud compute addresses list
```
 take note of the static address created

2. Delete existing ip assigned to the VM instance when it was created before assigning to it the static IP address
```bash 
    gcloud compute instances delete-access-config ovpntest --zone=$ZONE --access-config-name="external-nat"
```
3. Assign the static ip to the VM
```bash 
    gcloud compute instances add-access-config ovpntest --zone=$ZONE --address=<ipv4 address> 
```
4.  Create firewall rule to allow OpenVPN
openvpn uses udp port 1194 by default, udp is prefered due to....
```bash
    gcloud compute firewall-rules create allow-openvpn \
     --allow udp:1194 \
     --target-tags=openvpn-server \
     --description="Allow OpenVPN UDP 1194"
```

## STEP 3:  SSH to the VM and install OpenVPN (Nyr quick installer)
1. Access the vm instance using SSH
```bash 
       gcloud compute ssh ovpntest --zone=$ZONE
```
       Note: ovpntest is the name of the vm instance set during VM creation
       you will be prompted to provide a password, you can leave it blank or provide a password
2. Install openvpn leave all configuration options as default during the installation process, 
```bash 
    wget https://git.io/vpn -O openvpn-install.sh && sudo bash openvpn-install.sh
```
    It should output the file path for the openvpn client configuration file. take note of that Path.
3. check open vpn is running

```bash
   sudo service openvpn status
```

4. Enable ip forwadding
    This is done to ensure that....
  
```bash
    sudo sysctl -w net.ipv4.ip_forward=1
    
    # enable immediately
    sudo sysctl -w net.ipv4.ip_forward=1
    
    # make permanent
    echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

    # NAT rule for IPv4 (adjust ens4 if your interface name differs)
    sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o ens4 -j MASQUERADE
    
    # persist iptables rules (Ubuntu example)
    sudo apt update && sudo apt install -y iptables-persistent
    
    sudo netfilter-persistent save

```
    
## STEP 4: Retrieve the client .ovpn file and test the vpn server

1. Exit vm and run scp command to copy the configuration file to your cloud shell environment.
```bash
   gcloud compute scp ovpntest:<.ovpn file path> ./ --zone=$ZONE
   
```

2. Download the file to a client device(i am using parrot OS in my case).


  
   

