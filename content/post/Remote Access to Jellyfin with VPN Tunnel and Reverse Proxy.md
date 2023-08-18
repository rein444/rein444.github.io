---
title: "Remote Access to Jellyfin with VPN Tunnel and Reverse Proxy"
date: 2023-08-18T00:22:46-04:00
tags: [tech]
draft: false
---

Jellyfin is a noob-friendly media server that anyone can host at home. The basic setup is very easy since all the configuration can be done from the interactive web UI, and there exists many options for accessing the Jellyfin server from Internet securely. This is a guide for myself and others to reference in the future.

Make sure you have done the following tasks already:

1. Jellyfin is installed and can be accessed from local network. If not, read [this](https://jellyfin.org/docs/general/installation/).
2. Remote access is enabled. 
3. Nothing else besides the remote access option is changed from its defaults on the networking section. Jellyfin has built-in HTTPS functionality but it is known to be buggy. But if you do want to use it, Jellyfin has a [documentation](https://jellyfin.org/docs/general/networking/) for it.
4. You have a domain name that points to the server you will use for Wireguard and Caddy reverse proxy.

---

## Overview

In my specific situation, I already have a VPS running as a Wireguard server and a desktop running Jellyfin as a Wireguard peer. I do not want to forward any port in my home router, nor do I want to create a mesh VPN network unless I could find a way to connect to multiple interfaces concurrently on Android. In order to connect to Jellyfin without VPN, I would need to bind it to a domain.

Therefore I need to put the private address of my desktop assigned by Wireguard behind a reverse proxy on my VPS and forward the port used by the Jellyfin web UI from the public network interface back to my desktop via the Wireguard tunnel. 

---

## Step 1: Wireguard Server

Install Wireguard on the VPS and generate the private key.
```bash
wg genkey | sudo tee /etc/wireguard/private.key
```
Generate the public key derived from the private key and store it in the `/etc/wireguard` folder.
```bash
cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
```
Create a new configuration file named `/etc/wireguard/wg0.conf` and add the following lines:
```
[Interface]
PrivateKey = your_private_key
Address = 10.0.0.1/24
ListenPort = 51820

PostUp = ufw route allow in on wg0 out on eth0
PostUp = iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
PreDown = ufw route delete allow in on wg0 out on eth0
PreDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```
`PrivateKey`: the key you generated from the first command
`Address`: the ipv4 (and/or ipv6) private address range for the interface
`ListenPort`: the port Wireguard listens to and you would have to open it by issuing `ufw allow 51820/udp`; TCP access is not needed unless you are tunneling Wireguard packets over TCP connection
`PostUp`: automatically adding routing rules after Wireguard is up; the first one allows traffic to enter through the Wireguard interface and exit through the public network interface (check the name with `ifconfig`); the second one sets up NAT to route inbound traffic to the external internet
`PreDown`: wiping all the routing rules we added before the interface goes down

For ipv6 access, just copy the two iptables rules and change "iptables" to "ip6tables".

BTW some guides will suggest you to add `SaveConfig = true`, but this will just overwrite all the edits you make in the config when you restart the service. Do not add it unless you prefer using the command line to add peers.

Next, configure forwarding in the `/etc/sysctl.conf` that corresponds with the ufw route rule we added. Uncomment the following line:
```
net.ipv4.ip_forward=1
```
Confirm and load.
```bash
sudo sysctl -p
```
Enable and start Wireguard via systemd.
```bash
sudo systemctl enable wg-quick@wg0.service --now
```

---

## Step 2: Wireguard Peer

On the device running Jellyfin, generate the private key and the public key in the same way we did before. After this is done, edit the `/etc/wireguard/wg0.conf` file:
```
[Interface]
PrivateKey = your_private_key
Address = 10.0.0.2/24

[Peer]
PublicKey = your_server's_public_key
AllowedIPs = 10.0.0.0/24
Endpoint = server_ip:51820
```
`PrivateKey`: the private key for the peer
`Address`: the private address assigned to the peer
`PublicKey`: the public key for the server
`AllowedIPs`: instructs the peer to only route traffic via Wireguard if the destination is within the range; to route all trafic, change it to `0.0.0.0/0` for ipv4 and `::/0` for ipv6
`Endpoint`: the public address of your VPS and the port Wireguard listens to

Now go back to the server and add the peer to the configuration file:
```
[Peer]
PublicKey = the_peer's_public_key
AllowedIPs = 10.0.0.2/32
```
Restart the service.
```
sudo systemctl restart wg-quick@wg0.service
```
Go back to the peer again and start the service. Optionally, you can import the Wireguard profile to NetworkManager.
```bash
sudo nmcli connection import type wireguard file /etc/wireguard/wg0.conf
```
If you happen to be using a RHEL based distro and cannot start Wireguard with systemd, SELinux might be causing problems. To resolve this, the easiest way is to just disable it entirely.
```bash
sudo setenforce 0
```
To make this permanent, edit the following line in `/etc/sysconfig/selinux`:
```
SELINUX=permissive
```
It should now start without errors.

---

## Step 3: Reverse Proxy

Verify that Jellyfin can be accessed from `10.0.0.2:8096` on the local machine. There are different ways to open up ports for Jellyfin and what I did was to allow access from every device on the VPN network. This might be overkill but wouldn't be a security concern.
```
sudo ufw allow in on wg0 from 10.0.0.0/24
```
On the server side, edit your Caddyfile so your Jellyfin web UI can be accessed from your domain name.
```
your.domain {
	reverse_proxy 10.0.0.2:8096
}
```
If you haven't done so, allow HTTP/HTTPS trafic.
```
ufw allow http
ufw allow https 
```
You will see nothing when you try to visit the domain because ports haven't been forwarded yet. The VPS is basically serving as a router now so you need to add IP masquerading rules and forward the port Jellyfin listens to for incoming traffic, which will be explained in the next section.

---

## Step 4: Firewall Rules

Unfortunately UFW, like its name suggests, is only a simple firewall, so we have to tinker with iptables for port forwarding rules to work.

First, add a route rule for port 8096 with ufw.
```
sudo ufw route allow proto tcp to 10.0.0.2 port 8096
```
Next, open `/etc/ufw/before.rules` file and put the following rules inside the `*nat` block. The location of the block is important and you will receive errors otherwise, so I will include more parts of the file.
```
#
# rules.before
#
# Rules that should be run before the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-before-input
#   ufw-before-output
#   ufw-before-forward
#

*nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A PREROUTING -i eth0 -p tcp --dport 8096 -j DNAT --to-destination 10.0.0.2
-A POSTROUTING -o eth0 -j MASQUERADE

COMMIT


# Don't delete these required lines, otherwise there will be errors
*filter
:ufw-before-input - [0:0]
:ufw-before-output - [0:0]
:ufw-before-forward - [0:0]
:ufw-not-local - [0:0]
# End required lines
```
The `PREROUTING` chain of the NAT table is used to change the destination address of the packets before any packets are sent and in this case, we are redirecting TCP packets received on port 8096 to our desktop. The `POSTROUTING` chain is used to route packets to the external network after they are sent and this is exactly what we have in the Wireguard config, but I still included it here even though you might not need to. 

You would have to reload UFW after the rules are updated.
```
sudo ufw reload
```
Visit your domain and this time it should be showing the login interface.


