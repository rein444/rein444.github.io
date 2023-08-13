---
title: 'Building a simple proxy with V2ray'
date: 2021-04-13 07:37:15
tags: [tech]
published: true
hideInList: false
feature: 
isTop: false
---
V2ray is "a platform for building proxies to bypass network restrictions". You can find the github page [here](https://github.com/v2fly/v2ray-core), the official English website [here](https://www.v2ray.com/en/index.html), and also [here](https://www.v2fly.org/en_US/). As you might have noticed, v2ray is not a tool invented primarily for privacy (but it is very tweakable for all kinds of purposes), bur is rather used to bypass the GFW in China. Most of you will never use it probably, but it is always fun to try new things, am I right? And remember what I have said, its customization gives you a variety of options to play around with, like obfuscation, transparent proxy, or even ad blocking.

In a nutshell, v2ray uses a inbound-and-outbound structure to control the flow of packets. If v2ray is used on a client, then the inbound will receive data from the browser and the outbound will send data to the server; if v2ray is used on a server, then the inbound will receive data from the client and the outbound will send data to where the client has requested. You can have multiple outbounds and inbounds. The way inbound and outbound handles data is determined by different protocols (Socks, VMess, VLess, Freedom, etc.), and each protocol may have its own transport (TCP, mKcp, WebSocket, etc.).

VMess is an original and the most widely used protocol, developed by V2ray specifically for passing the deep packet inspection imposed by the GFW. VLess is an updated version of it that does not require time synchronization but does not have the traffic encrypted. Here we are going to setup a basic proxy using VMess.

---

## Server
Log into your server and set the time if it's not what you have on the client.
```bash
date -s 'year-month-day hour:minute:second'
```
It would be the easiest to install V2ray via their script.
```bash
wget https://install.direct/go.sh
bash go.sh
```
After the installation is finished, we can proceed to edit the config file. It is included as part of the installation script, named `config.json`. It should be edited like this:
```json
{
  "log": {
      "access": "/var/log/v2ray/access.log",
      "error": "/var/log/v2ray/error.log",
      "loglevel": "info"
  },
  "inbounds": [
    {
      "port": [port number],
      "protocol": "vmess",   
      "settings": {
        "clients": [
          {
            "id": "[your uuid]",
            "alterId": [your alterid]
          }
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```
`inbounds vmess port`: the listening port on the server
`id`: the main id of the user. you can generate one [here](https://www.uuidgenerator.net/)
`alterid`: used to pass the inspection. can be anywhere between 0 to the largest number your server can support
`freedom`: passes all TCP or UDP connection to their destinations

Now start the service.
```bash
systemctl start v2ray.service
```
If you have a firewall enabled, make sure to open the listening port. I am using ufw here.
```bash
ufw allow [port number]/tcp
ufw allow [port number]/udp
```

## Linux Client
Find your system's architecture [here](https://github.com/v2fly/v2ray-core/releases/tag/v4.37.3). Download and extract it. Now find the `config.json` file and write it like this.
```json
{
  "log":{
   "loglevel": "info",
   "access": "/var/log/v2ray/access.log",
   "error": "/var/log/v2ray/error.log"
  },
  "inbounds": [
    {
      "port": [port number],
      "protocol": "socks",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "settings": {
        "auth": "noauth"  
      }
    },
    {
      "port": [port number],
      "protocol": "http",
      "sniffing": {
          "enabled": true,
          "destOverride": ["http", "tls"]
        }
     }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "[server's ip address]",
            "port": [port number],  
            "users": [
              {
                "id": "[your uuid]",  //need to be the same as the server config
                "alterId": [your alterid], //need to be the same as the server config
                "security": "auto"
              }
            ]
          }
        ]
      },
      "mux": {"enabled": true}
    }
  ]
}
```
`inbounds socks port`: the listening port on your client for Socks protocol
`sniffing`: content sniffing, can be used for solving DNS spoofing and setting rules for domains
`noauth`: anonymous authentication
`inbounds http port`: the listening port on your client for HTTP protocol
`outbounds vmess port`: the listening port on your server
`security`: can be `aes-128-gcm` for PC use, `chacha20-poly1305` for phone use, or `auto` for automatic detection
`mux`: designed to reduce TCP handshake latency by using one physical TCP connections for multiple virtual TCP connections. might not be needed

To allow v2ray to autostart, you could add a systemd entry to the `/etc/systemd/system` folder. Make sure the config file and the executable file are in the same folder.
```bash
[Unit]
Description=v2ray
After=network.target

[Service]
ExecStart=/path/to/v2ray

[Install]
WantedBy=multi-user.target
```
Start it and enable it.
```bash
systemctl start v2ray.service
systemctl enable v2ray.service
```
If it can't be started, you might need to create the log files manually.

## Windows Client
Find your system's architecture [here](https://github.com/v2fly/v2ray-core/releases/tag/v4.37.3). Download and extract it. The config file should be the same as what I wrote above for linux and the only change you need to make is to delete the `log` sector because windows doesn't support it.

To autostart, create a .vbs file. Same thing here, make sure the config file and the exe file are in the same folder.
```
Dim WinScriptHost
Set WinScriptHost = CreateObject("WScript.Shell")
WinScriptHost.Run Chr(34) & "path\to\v2ray.exe" & Chr(34), 0
Set WinScriptHost = Nothing
```
Create a shortcut of it and add it to the `shell:startup` folder.

---

I also tried the WebSocket+TLS+Web+CDN combo, but somehow it didn't work. It might be some problem related to Cloudflare that I am too <del>dumb</del> lazy to figure out. The server is working fine, but the access log at the client is dumping on me tons of ambiguous errors. But if I do find the solution, I might update this later.
