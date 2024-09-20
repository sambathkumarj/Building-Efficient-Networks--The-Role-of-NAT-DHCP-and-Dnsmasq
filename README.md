# Building Efficient Networks:- The Role of NAT DHCP and Dnsmasq

# NAT - Network Address Translation

NAT is a technique used in computer networking to allow multiple devices on a local network (private IP addresses) to share a single public IP address to communicate with the external internet. NAT is crucial in reducing the depletion of IPv4 addresses by enabling multiple devices to use a single external IP for communication.

There are different types of NAT, but masquerade is commonly used for dynamic IP addresses in private networks that need to access the internet.

# How NAT Works on Linux
On Linux, you can configure NAT to allow devices in a private network to access the internet through a single network interface (which has internet access). This is achieved by enabling IP forwarding and setting up NAT rules with iptables.

# Enable IPv4 Forwarding in sysctl.conf:

By default, IP forwarding (the ability to pass packets between networks) is disabled. This step enables packet forwarding so that the machine can act as a router.

```
sudo vim /etc/sysctl.conf

#Inside the file, look for the line:

# net.ipv4.ip_forward=1
Un-comment it by removing the # to enable IP forwarding:
net.ipv4.ip_forward=1
```
# Apply the sysctl Changes:

After editing the sysctl.conf file, use the sysctl -p command to apply the changes immediately.

```
sudo sysctl -p
```
This ensures that IP forwarding is enabled without requiring a reboot.

# List the NAT Table Rules in iptables:

iptables is a command-line utility that allows administrators to configure the rules that control network traffic. The -L -t nat command lists all rules in the NAT table.

```
sudo iptables -L -t nat
```
This shows the existing NAT rules, if any. NAT rules are stored in the nat table.

# Add a NAT Rule (Masquerade):

The masquerade rule tells iptables to modify outgoing packets from your local network and make them appear to originate from the machine’s external (internet-facing) interface. This allows the machine to forward traffic from the internal network to the internet.

```
sudo iptables -t nat -A POSTROUTING -o wlp0s20f3 -j MASQUERADE
```
# Breakdown:

-t nat: Specifies the nat table where NAT rules are applied.
-A POSTROUTING: Adds a rule in the POSTROUTING chain, which manipulates packets after they are routed.
-o wlp0s20f3: Specifies the outgoing interface (internet-facing interface, e.g., wlp0s20f3) that packets are leaving from.
-j MASQUERADE: Applies masquerading, meaning that packets will appear to come from the machine’s external IP.

# Save the iptables Configuration:

After adding rules, you must save them so that they persist after a reboot. The iptables-save command writes the current configuration to a file.

```
sudo iptables-save
```
This command will save the current iptables configuration, typically in /etc/iptables/rules.v4 or similar.

# List NAT Table Again:
Running the command again after adding the NAT rule will display the updated NAT rules, showing the changes.

```
sudo iptables -L -t nat
```
You should now see the POSTROUTING rule with masquerading applied, similar to:

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  anywhere             anywhere

# Enable IP Forwarding:
```
sudo vim /etc/sysctl.conf

#Un-comment net.ipv4.ip_forward=1, save, and then apply:

sudo sysctl -p
```
# View Existing NAT Rules:

```
sudo iptables -L -t nat
```

# Add NAT Rule (Masquerade):

```
sudo iptables -t nat -A POSTROUTING -o wlp0s20f3 -j MASQUERADE
```

# Save iptables Configuration:
```
sudo iptables-save
```

# View Updated NAT Rules:
```
sudo iptables -L -t nat
```

This setup allows the machine to forward traffic from an internal private network to the internet by using NAT masquerading on the outgoing interface.

# DHCP - Dynamic Host Configuration Protocol

DHCP (Dynamic Host Configuration Protocol) is a network management protocol used to automate the process of configuring devices on IP networks. It works in a client-server architecture where the DHCP server dynamically assigns IP addresses and other network configuration parameters (like subnet mask, gateway, and DNS) to DHCP clients. This eliminates the need for manual IP address configuration for each client on the network.

The DHCP server "leases" an IP address to the client for a certain period of time. If the client wishes to continue using the IP, it must request a renewal of the lease before it expires. Otherwise, the IP address is returned to the pool for other devices to use.

# How to Configure a DHCP Server on Ubuntu 20.04


# 1. Update the Package Index
Ensure your package list is up-to-date, so you can install the latest available version of the DHCP server.
```
sudo apt update
```
This command updates the local package index with the latest information from the package repositories.

# 2. Install the DHCP Server and Dependencies
Install the ISC DHCP server, which is a widely used software for providing dynamic IP addressing.
```
sudo apt install isc-dhcp-server -y
```
This installs the ISC DHCP server along with its dependencies:

-y: Automatically answers "yes" to any prompts during the installation.

# 3. Configure the DHCP Server

# A. Specify the Network Interface for DHCP
Indicate the interface on which the DHCP server should listen for client requests.
```
sudo vi /etc/default/isc-dhcp-server
#File Content:

INTERFACESv4="enp0s8"
```
Explanation:

INTERFACESv4="enp0s8": This tells the DHCP server to provide IP addresses on the network interface enp0s8. In your case, you could use the relevant interface such as eno1 or another one that connects to the network.

# B. Edit the DHCP Configuration File
Set up the configuration for the IP addresses, subnet, and lease time.
```
sudo vi /etc/dhcp/dhcpd.conf
#Edit the File:

#Comment out the DNS parameters (if no internal DNS is needed):

#option domain-name "example.org";
#option domain-name-servers ns1.example.org, ns2.example.org;
#Uncomment the authoritative directive (DHCP server becomes the authority for the local network):

authoritative;
```

# Configure Lease Time (Optional):

Default lease time (how long the IP is leased to the client):
```
default-lease-time 600;
Maximum lease time:
max-lease-time 7200;
#Add the Subnet and IP Range:

#Specify the subnet, IP range, and gateway for the DHCP server. In this case, we use the network 172.18.1.0/24 with the gateway at 172.18.1.225 (the gateway of eno1 which has internet access).

Example configuration:

subnet 172.18.1.0 netmask 255.255.255.0 {
  range 172.18.1.200 172.18.1.250;
  option routers 172.18.1.225;
}
```
# Explanation:

subnet 172.18.1.0 netmask 255.255.255.0: Defines the subnet from which the DHCP server will allocate IPs.
range 172.18.1.200 172.18.1.250: Specifies the range of IP addresses that the DHCP server can allocate.
option routers 172.18.1.225: Sets the default gateway for clients to use (in this case, the gateway of eno1).

# 4. Start and Enable the DHCP Server
Start the DHCP service and ensure it runs automatically on system boot.
# Start the Service:
```
sudo systemctl start isc-dhcp-server
```
This command starts the isc-dhcp-server service, making it available to lease IPs to clients.

# Enable the Service at Boot:
```
sudo systemctl enable isc-dhcp-server
```
This command enables the DHCP server to start automatically when the system boots.

# Verify the DHCP Server Status:
```
sudo systemctl status isc-dhcp-server
```
This command displays the current status of the DHCP service, showing whether it is running and if there are any errors.

# 5. Configure a DHCP Client
Configure a client machine to obtain an IP address automatically from the DHCP server.
Client-Side Configuration:
On the client machine, go to the network settings for the interface (e.g., eno1) and set the IP configuration to Automatic (DHCP).

Once the client is set to obtain an IP automatically, the DHCP server will assign it an IP address from the range defined (172.18.1.200 to 172.18.1.250) along with the subnet mask, default gateway, and DNS settings.

# Update Package Index:
```
sudo apt update
```
# Install DHCP Server:

```
sudo apt install isc-dhcp-server -y
```
# Configure DHCP Interface:
```
sudo vi /etc/default/isc-dhcp-server
```
# Set the interface:
```
INTERFACESv4="enp0s8"
```

# Edit DHCP Configuration:
```
sudo vi /etc/dhcp/dhcpd.conf

#Add the subnet configuration:

subnet 172.18.1.0 netmask 255.255.255.0 {
  range 172.18.1.200 172.18.1.250;
  option routers 172.18.1.225;
}
```

# Start the DHCP Service:
```
sudo systemctl start isc-dhcp-server
```
# Enable the DHCP Service:
```
sudo systemctl enable isc-dhcp-server
```
# Check the DHCP Service Status:

```
sudo systemctl status isc-dhcp-server
```
By following these steps, you can configure a DHCP server to dynamically assign IP addresses to clients in your network, ensuring efficient network management and IP address allocation.


# DNSMASQ:
		dnsmasq is a lightweight, easy to configure DNS forwarder, designed to provide DNS (and optionally DHCP and TFTP) services to a small-scale network. It can serve the names of local machines which are not in the global DNS.
	

 # Purpose of dnsmasq:
		dnsmasq caches DNS records, reducing the load on upstream nameservers and improving performance, and can be configured to automatically pick up the addresses of its upstream servers. dnsmasq accepts DNS queries and either answers them from a small, local cache or forwards them to a real, recursive DNS server.

# STEP 1: Set a Static IP for the Ethernet Interface

Objective: Assign a static IP address to the network interface for handling the internal network.

Example Configuration:

IP Address: 192.168.10.1
Subnet Mask: /24 (which corresponds to 255.255.255.0)
DNS: 192.168.10.1

Command: Edit the network configuration file for your interface, which could vary depending on the distribution (e.g., /etc/network/interfaces or /etc/netplan/ on Ubuntu).

For instance, on Ubuntu with Netplan:

```
sudo nano /etc/netplan/00-installer-config.yaml

#Add the following:
yaml
Copy code
network:
  ethernets:
    enp43s0:
      addresses:
        - 192.168.10.1/24
      nameservers:
        addresses: [192.168.10.1]
      dhcp4: no
  version: 2
#Apply the changes:
```

```
sudo netplan apply
```
# STEP 2: Install dnsmasq
Install the dnsmasq package, which provides lightweight DNS, DHCP, and TFTP services, perfect for small-scale or private networks.
```
sudo apt install dnsmasq
```
This will install dnsmasq and its dependencies.

# STEP 3: Configure dnsmasq
Configure dnsmasq to assign IP addresses via DHCP and provide DNS resolution services.

Backup the Existing Configuration: Before editing the configuration, create a backup of the default configuration:

```
sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.bck
```
# Write a New Configuration: Open the dnsmasq configuration file for editing:

```
sudo nano /etc/dnsmasq.conf

#Add the following content:

interface=enp43s0
port=0
dhcp-range=192.168.10.10,192.168.10.253,48h
dhcp-option=option:dns-server,192.168.10.1,192.168.1.1

#interface=enp43s0: Defines the interface dnsmasq will listen on.
#port=0: Disables the DNS server provided by dnsmasq (optional, based on your needs).
#dhcp-range: Specifies the range of IPs for the DHCP server (from .10 to .253 with a lease time of 48 hours).
#dhcp-option: Defines DNS server addresses; the first (192.168.10.1) is local, and the second (192.168.1.1) is an external DNS server (your gateway with internet access).
```

# STEP 4: Stop systemd-resolved and Restart dnsmasq

Disable systemd-resolved to avoid conflicts with dnsmasq.

```
sudo systemctl stop systemd-resolved
sudo systemctl restart dnsmasq
```
systemd-resolved provides DNS services, which can conflict with dnsmasq. By stopping it, we ensure dnsmasq takes over DNS functionality.
Restarting dnsmasq applies the new configuration.

# STEP 5: Create and Configure rc.local Script
Set up a script to run at startup, which will handle stopping systemd-resolved, starting dnsmasq, and setting up NAT.

Navigate to /etc/ and create the rc.local file:

```
cd /etc/
sudo nano rc.local
```

# Write the following script:

```
#!/bin/sh
# rc.local

service systemd-resolved stop
service dnsmasq restart
iptables -t nat -A POSTROUTING -o wlp0s20f3 -j MASQUERADE
exit 0
```

# Explanation:
service systemd-resolved stop: Stops systemd-resolved on every boot.
service dnsmasq restart: Ensures dnsmasq starts and provides DHCP/DNS services.
iptables -t nat -A POSTROUTING -o wlp0s20f3 -j MASQUERADE: Enables NAT, where wlp0s20f3 is the interface connected to the internet. This allows the internal network (with the 192.168.10.0/24 subnet) to share internet access through this interface.

# STEP 6: Reload and Start rc-local

Objective: Ensure the rc.local script is executable and that it starts at boot.

# Reload the system daemon:

```
sudo systemctl daemon-reload
```

# Start the rc-local service:

```
sudo systemctl start rc-local
```

# Make rc.local executable:

```
sudo chmod +x /etc/rc.local
```

# Restart and check the status of the rc-local service:

```
sudo systemctl restart rc-local
sudo systemctl status rc-local
```

Check /etc/sysctl.conf: Ensure packet forwarding for IPv4 is enabled:

```
sudo nano /etc/sysctl.conf
```
Look for the line:
```
net.ipv4.ip_forward=1
```

If it's commented out, uncomment it and save the file.

# Restart Services: Restart both the rc-local and dnsmasq services:

```
sudo systemctl restart rc-local
sudo systemctl restart dnsmasq
```

This setup ensures that dnsmasq handles DHCP and DNS for a local network, while NAT allows devices on that network to access the internet through the main interface.

