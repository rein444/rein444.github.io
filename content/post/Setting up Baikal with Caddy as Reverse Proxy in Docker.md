---
title: "Setting Up Baikal With Caddy as Reverse Proxy in Docker"
date: 2023-08-12T23:01:44-04:00
tags: [tech]
---

Baikal is a lightweight CalDav/CardDav server that offers a web interface for management and is simple to setup. You can read more about the internet standards it supports [here](https://sabre.io/dav/). If you are looking for an alternative for Baikal, check out [Radicale](https://radicale.org/v3.html). As far as I know, the main difference between these two is that instead of storing data in a SQL database, Radicale stores all information in .ics files. So far I am having no problem with Baikal (and by so far I mean...a week) and it is running stably.

This post will be a short guide on how to run Baikal with a reverse proxy for SSL certificate in Docker and import settings to clients that support Dav synchronization.

---

## Server

I used [this premade Docker image](https://github.com/ckulka/baikal-docker) to run Baikal with Docker Compose. The creater of the repo recommended Traefik for HTTPS support but I found Traefik to be a pain in the ass to configure, so in this guide I will be using Caddy.

First, create a network dedicated for containers that you will put behind the Caddy proxy.
```bash
docker network create caddy
```
Create a Caddyfile, preferably in the same folder where you will place your Docker Compose file but it can be anywhere you want.

In the `docker-compose.yaml` for Caddy, write this:
```yaml
version: '3.7'

networks:
  default:
    external:
      name: caddy

services:

  caddy:
    container_name: caddy
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /path/to/Caddyfile:/etc/caddy/Caddyfile
      - /path/to/site:/srv
      - caddy_data:/data
      - caddy_config:/config
      
volumes:
  caddy_data:
  caddy_config:
```
`caddy_data` and `caddy_config` are optional and according to the documentation, you can define the data volume as external to make sure `docker-compose down` does not delete it.

Before you put Baikal on the caddy network, you can check whether it is running fine with the default config to facilitate troubleshooting later. Create a folder for Baikal, and fetch the default config from the repo.
```bash
wget https://raw.githubusercontent.com/ckulka/baikal-docker/master/examples/docker-compose.yaml
```
The default config is exposing the HTTP port 80, but I mapped it to something else. This part is not necessary. You can start the container now.
```bash
docker-compose up -d
```
Verify that Baikal runs correctly by typing your public IP into the browser and you should be able to see the installation page. A troubleshooting section is also provided in the end of the guide.

Now we are ready to encrypt the connection. Overwrite the following into your `docker-compose.yaml` for baikal:
```yaml
# Docker Compose file for a Baikal server

version: "2"

networks:
  default:
    external:
      name: caddy


services:
  baikal:
    image: ckulka/baikal:nginx
    container_name: baikal
    restart: always
    ports:
      - "[bounded port]:80"
    volumes:
      - config:/var/www/baikal/config
      - data:/var/www/baikal/Specific

volumes:
  config:
  data:
```
Go back to your Caddyfile and write the following:
```
{
   email user@yourdomain.com
}

yourdomain.com {
   reverse_proxy baikal:80 
}
```
Start the caddy container and check if HTTPS is enabled for the domain you linked to the server's IP address. Complete the setup and in the database section, select MySQL if you are going to have a large database. It will not be automatically configured so do it yourself on the server.

Once you are logged in, go to "Users and resources" and create as many users as needed.

---

## Client

On Android devides, download [DAVx⁵](https://www.davx5.com/). To import your account, choose the "Login with URL and user name" option and put "https://yourdomain.com/dav.php" for base URL. After that, synchronize both empty folders. This step might not be necessary, but I can't access the Dav synchonization option in other apps if I haven't done so.

I use [Simple Calender](https://f-droid.org/en/packages/com.simplemobiletools.calendar.pro/) and it is able to discover my synced calender when I checked the option in the settings. For contact, I use the default Contacts app that came with the OS and it picked up the synced address book automatically.

On the laptop, I use [Thunderbird](https://www.thunderbird.net/en-US/). When it prompts for whether you want to create a new calender/address book, just select "On the Network" and type the same address you used for base url in the DAVx⁵ app.

---

## Troubleshooting

1. ***Unable to connect to the server with the default config***

First, make sure that Baikal is actually running with the command `docker -ps`. Next, check if the firewall has port 80 and 443 opened. With ufw, you can simply issue `ufw allow http` and `ufw allow https`.

If you are mapping a port other than 80, make sure to add ":the-port-you-chose" after the IP address when you type into the browser.

2. ***Able to connect to the server with the default config, but not with Caddy***

Again, check if the container is running and if the Caddyfile you just edited is correctly synced with the command `docker exec caddy cat /etc/caddy/Caddyfile`. If it doesn't match with what you wrote, execute `docker exec -w /etc/caddy caddy caddy reload`. Pay attention to the `reverse_proxy` line in your Caddyfile and use the port that the Docker network is listening to (in this case, 80) rather than the port assigned to the server.

Also check if the path to Caddyfile and the domain name are typed correctly.

Otherwise, check the DNS setting with your domain name registar and ping the domain to see if the A Record is pointing your domain to the right server.

3. ***The port failed to bind***

There is an existing service running that conflicts with the port you exposed on the server. Find the service by running `netstat -ltpn | grep [port-num]` and terminate it or assign a different port.

4. ***Unable to connect to the server via URL***

Try a different URL. Here is a list of examples to try:
```
https://yourdomain.com/baikal/dav.php/
https://yourdomain.com/baikal/html/dav.php/
https://yourdomain.com/baikal/cal.php/calendars/username/default
```

The [official documentation](https://sabre.io/baikal/troubleshooting/) mentioned that Windows 10 clients do not support Digest authentication, so switch to Basic auth.
