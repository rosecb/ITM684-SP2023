# Wireguard VPN Server Using Docker Project Documentation

## Step 1: **Deploy an Ubuntu 22.0.4 LTS server to Digital Ocean**
* Choose region: New York 
	* Datacenter: NYC1
* Image: Ubuntu 22.04 LTS x64
* Size
	* Droplet: Basic
	* Plan: $4/mo
* Create root password & host name

## Step 2: Setup Wireguard VPN Server with Docker

Click [here](https://thematrix.dev/setup-wireguard-vpn-server-with-docker/) for the reference.

Setup Wireguard, paste the following below:
```
mkdir -p ~/wireguard/
mkdir -p ~/wireguard/config/
nano ~/wireguard/docker-compose.yml
```

In the nano file, copy and paste the content below:

```
version: '3.8'
services:
  wireguard:
    container_name: wireguard
    image: linuxserver/wireguard
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Pacific/Honolulu
      - SERVERURL=146.190.209.185
      - SERVERPORT=51820
      - PEERS=rosepc1,rosepc2,rosephone1
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.0.0.0
    ports:
      - 51820:51820/udp
    volumes:
      - type: bind
        source: ./config/
        target: /config/
      - type: bind
        source: /lib/modules
        target: /lib/modules
    restart: always
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
```

Start Wireguard by running the following commands:
```
cd ~/wireguard/
docker-compose up -d
```

Download Wireguard application. 

## Step 3: Test Wireguard Server 

Figure out where the `.conf` file is located on the Digital Ocean Droplet. 

In the terminal (on Mac), copy below the following:

`scp rose@146.190.209.185:~/wireguard/config/peer_rosepc1/peer_rosepc1.conf`

Add the `.conf` file onto Wireguard. You should get the following to show up:

Wireguard:

![image](https://user-images.githubusercontent.com/70050375/235273413-788324da-e66a-4d59-8da1-4d107c07f590.png)

