## Step 1: **Deploy an Ubuntu 22.0.4 LTS server to Digital Ocean**
* Choose region: New York 
	* Datacenter: NYC1
* Image: Ubuntu 22.04 LTS x64
* Size
	* Droplet: Basic
	* CPU: 
	* Plan: $6/mo
* Create root password & host name
	* ***openvpnserver***

## Step 2: **Initial Server Setup with Ubuntu 22.04**

This will be the OpenVPN server. Click [here](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04) for more information on references.

* Log in using root 
	* `ssh root@146.190.78.68`
* Create new user
	* `adduser rose`
* Grant administrative privileges
	* `usermod -aG sudo rose`

## Step 3: Set Up Basic Firewall Rules on VPN Server

 * View list of installed UFW profiles
	 * `ufw app list`
	 * make sure you have an output of OpenSSH
* Make sure firewalls allows SSH connections
	* `ufw allow OpenSSH`
* Enable firewalls
	* `ufw enable`
		* When prompted with “Command may disrupt existing ssh connections. Proceed with operation (y|n)” use  y
		* Make sure you have an output of “Firewall is active and enabled on system startup”
* See status with 
	* `ufw status`
* Sudo into user
	* `ssh rose@146.190.78.68`
* Change default SSH port to 21999
	* `sudo ufw allow 21999`
	* `sudo nano /etc/ssh/sshd_config`
	* Search for # Port22, and remove the # and change the port number to 21999
 * Veridy SSH is listening
	 * *`sudo ufw status`

## Step 4: **Set Up and Configure a CA Server**

This will be the Certificate Authority Server. We will set up a separate CA server. Click [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-on-ubuntu-22-04) for more information on references.

* Repeat step 1 with another droplet/server and name that ***ca server***
* Repeat step 2 with the initial server setup
	* `ssh root@192.241.132.220`
	* `adduser rose`
	* `usermod -aG sudo rose`
	* `ufw allow OpenSSH`
	* `ufw enable`
	* `ufw status`
	* `ssh rose@192.241.132.220`
* Install easy-rsa
	* `sudo apt update`
	* `sudo apt install easy-rsa`
* Prepare a Public Key Infrastructure Directory
	* `mkdir ~/easy-rsa`
		* You can view the new directory using the following command `ls`
* Create a symlink from the easyrsa script that the package installed into the ~/easy-rsa directory that you just created
	* `**ln -s /usr/share/easy-rsa/* ~/easy-rsa/**`
* Resist access to the new PKI directory for the owner only
	* `chmod 700 /home/rose/easy-rsa`
* Initialize the PKI inside the directory
	* `cd ~/easy-rsa`
	* `./easyrsa init-pki`
	* Output should be
		init-pki complete; you may now create a CA or requests.
		Your newly created PKI dir is: /home/rose/easy-rsa/pki
* Create a Certificate Authority 
	* `cd ~/easy-rsa`
	* `nano vars`
* Edit country, province, city, org, email, and community and paste the following
	```
	set_var EASYRSA_REQ_COUNTRY    "US"
	et_var EASYRSA_REQ_PROVINCE   "Hawaii"
	set_var EASYRSA_REQ_CITY       "Honolulu"
	set_var EASYRSA_REQ_ORG        "Manoa"
	set_var EASYRSA_REQ_EMAIL      "rosecb@hawaii.edu"
	set_var EASYRSA_REQ_OU         "Community"
	set_var EASYRSA_ALGO           "ec"
	set_var EASYRSA_DIGEST         "sha512"
	```
* `./easyrsa build-ca`

## Step 5: Set up and Configure an OpenVPN Server

Click [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-ubuntu-22-04)) for more information on references.

Install openvpn and easy-rsa
		* `ssh root@146.190.78.68`
	* Login as non-root user 
		* `ssh rose@146.190.78.68`
	* Create new directory 
		* `mkdir ~/easy-rsa`
	* Create a symlink from the easyrsa script that the package installed into the ~/easy-rsa directory that you just created
		* `ln -s /usr/share/easy-rsa/* ~/easy-rsa/`
	* Ensure directory's owner is your non-root sudo user and restrict assess to that user
		* `sudo chown rose ~/easy-rsa`
		* `chmod 700 ~/easy-rsa`

Create a PKI for OpenVPN
	* `cd ~/easy-rsa`
	* `nano vars`
	* In the nano file, paste the following
		* `set_var EASYRSA_ALGO "ec"
		* `set_var EASYRSA_DIGEST "sha512"`
	* Run easy rsa script
		* ``./easyrsa init-pki`

Create an OpenVPN Server Certificate Request and Private Key
* `cd ~/easy-rsa`
* `./easyrsa gen-req server nopass`
	* This creates a private key for the server and a certificate request file called ***server.req***
* `sudo cp /home/rose/easy-rsa/pki/private/server.key /etc/openvpn/server/`

## Step 6: **Sign the OpenVPN Server's Certificate Request**

In OpenVPN Server 
`scp /home/rose/easy-rsa/pki/reqs/server.req rose@192.241.132.220:/tmp`

In CA Server
* *`ssh root@192.241.132.220`
* `ssh rose@192.241.132.220`
* `cd ~/easy-rsa`
* `./easyrsa import-req /tmp/server.req server`
	* This imports the server certificate request
* `./easyrsa sign-req server server`
* Copy the ***server.crt*** and ***ca.crt*** files from the CA server to the OpenVPN server
	* `scp pki/issued/server.crt rose@146.190.78.68:/tmp`
	* `scp pki/ca.crt rose@146.190.78.68:/tmp`

**HEAD BACK TO the CA Server**
* `ssh root@146.190.78.68`
* `ssh rose@146.190.78.68`
* Configure 
	* `sudo cp /tmp/{server.crt,ca.crt} /etc/openvpn/server`

## Step 7: **Configure OpenVPN Cryptographic Material**

Generate tls-crypt key preshared key, run the following commands:
* `cd ~/easy-rsa`
* `openvpn --genkey secret ta.key`
* `sudo cp ta.key /etc/openvpn/server`

## Step 8: **Generate a Client Certificate and Key Pair**

* Go to home directory
	*  `cd ~`
	* `mkdir -p ~/client-configs/keys`
	* Lockdown its permissions as a security measure
		* `chmod -R 700 ~/client-configs`
	* Navigate back to the EasyRSA directory and run the script
		* `cd ~/easy-rsa`
		* `./easyrsa gen-req client1 nopass`
	* Copy file to the directory you created earlier
		* `cp pki/private/client1.key ~/client-configs/keys/`
		* `scp pki/reqs/client1.req rose@192.241.132.220:/tmp`

* Go back to the CA Server
	* `ssh root@192.241.132.220`
	* `ssh rose@192.241.132.220`
	* `cd ~/easy-rsa`
	* `./easyrsa import-req /tmp/client1.req client1`
	* Sign the request and specify client tpe 
		* `./easyrsa sign-req client client1`
		* Enter passphrase
	* **Transfer file back to the server**
		* `scp pki/issued/client1.crt rose@146.190.78.68:/tmp`

* **HEAD BACK INTO THE OPENVPN SERVER**
	* `ssh root@146.190.78.68`
	* `ssh rose@146.190.78.68`
	* Copy client certificate to directory
		* `cp /tmp/client1.crt ~/client-configs/keys/`
	* Copy files to the directory and set appropriate permissions
		* `cp ~/easy-rsa/ta.key ~/client-configs/keys/`
		* `sudo cp /etc/openvpn/server/ca.crt ~/client-configs/keys/`
		* `sudo chown rose.rose ~/client-configs/keys/`

## Step 9: **Configure OpenVPN**

* Copy the sample as a starting point for your own config file
	* `sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/`
	* `sudo nano /etc/openvpn/server/server.conf`
	* INSIDE NANO FILE, update the following:
		* “#”tls-auth ta.key 0 # This file is secret
		* `tls-crypt ta.key`
		* Update the cipher 
			* `;cipher AES-256-CBC`
			* `cipher AES-256-GCM`
		* Add auth directive
			* `auth SHA256`
		* Find the line with dh directive
			* `;dh dh2048.pem`
			* `dh none`
		* Uncomment lines
			* `user nobody`
			* `group nogroup`
		* Push DNS Changes to Redirect All Traffic Through the VPN. Uncomment the following by removing the ; in the beginning:
			* `push "redirect-gateway def1 bypass-dhcp"`
			* `push "dhcp-option DNS 208.67.222.222"`
			* `push "dhcp-option DNS 208.67.220.220"`
		* Adjust the Port and Protocol
			* `port 443`
			* `proto tcp`
		* Go to the end of the file and change value to 0
			* `explicit-exit-notify 0`
		* Save and close file

## Step 10: **Adjust OPENVPN SERVER NETWORK CONFIG**

* Open file using nano
	* `sudo nano /etc/sysctl.conf`
	* Add the following line at the buttom of the file
		* `net.ipv4.ip_forward = 1`
	* To read file and load new values:
		* `sudo sysctl -p`
		* Output would be net.ipv4.ip_forward = 1


## Step 11: Firewall Configuration 

* Find the public network interface of your machine
	* `ip route list default`
	* Output would be default via 157.245.240.1 dev eth0 proto static
	* Open the /etc/ufw/before.rules file
		* `sudo nano /etc/ufw/before.rules`
* Add the following text, then save and exit nano.
```
# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
# Allow traffic from OpenVPN client to eth0 (change to the interface you discovered!)
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES
```

* Tell UFW to allow forwarded packets by default as well
	* `sudo nano /etc/default/ufw`
	* Change DROP to ACCEPT for the DEFAULT-FORWARD_POLICY
		* `DEFAULT_FORWARD_POLICY="ACCEPT"`
		* Save and exit the file
	* `sudo ufw allow 443/tcp`
	* `sudo ufw allow 53/udp`
	* `sudo ufw allow OpenSSH`
* Disable and re-enable UFW to restart ur and load changes from all the files you modified
	* `sudo ufw disable`
	* `sudo ufw enable`

## Step 12: Start OpenVPN

* Enable the OpenVPN Service
	* `sudo systemctl -f enable openvpn-server@server.service`
* Start the OpenVPN Service
	* `sudo systemctl start openvpn-server@server.service`
* Check that the OpenVPN Service is active
	* `sudo systemctl start openvpn-server@server.service`

## Step 13: Create the Client Configuration Infrastructure

`mkdir -p ~/client-configs/files`

`cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf`

* Open baseline client config file and edit 
	* `nano ~/client-configs/base.conf`

* change `remote my-server-1 1194` to `remote 146.190.78.68 443`
* change protocol to `proto tcp`
* uncomment `user nobody` and `group nogroup`
* comment out these directives 
	* `;ca ca.crt`
	* `;cert client.crt`
	* `;key client.key`
	* `;tls-auth ta.key 1`
* change cipher and auth 
	* `cipher AES-256-GCM`
	* `auth SHA256`
* Add commented out lines to handle methods that linx based VPN clients will use for DNS resolution 

For Clients that do not use systemd-resolved
```
; script-security 2
; up /etc/openvpn/update-resolv-conf
; down /etc/openvpn/update-resolv-conf
```

For Clients that use systemd-resolved for DNS resolution
```
; script-security 2
; up /etc/openvpn/update-systemd-resolved
; down /etc/openvpn/update-systemd-resolved
; down-pre
; dhcp-option DOMAIN-ROUTE .
```

Save and close file

* Create a script to compile your base configuration 
	* `nano ~/client-configs/make_config.sh`
Add the following:
```
#!/bin/bash
# First argument: Client identifier
KEY_DIR=~/client-configs/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf
cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-crypt>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-crypt>') \
    > ${OUTPUT_DIR}/${1}.ovpn
```

* Save and exit file
* `chmod 700 ~/client-configs/make_config.sh`

## Step 14: Generating Client Configs

Run the script you create and check to make sure that the client config files mentioned are there
`cd ~/client-configs`
`./make_config.sh client1`
`ls ~/client-configs/files`

Transfer the client config file `client1.opvn` to the device you plan to use as the client. 
`sftp -P 21999 rose@146.190.78.68:client-configs/files/client1.ovpn ~/`

## Step 15: Connect and test your OpenVPN client 

* Download an OpenVPN client and test your OpenVPN Client 
	* [Tunnelblick] (https://tunnelblick.net/)
	* [OpenVPN] (https://openvpn.net/community-downloads/)
* Open a browser and go to [https://ipleak.net](https://ipleak.net/). Look at your IP address and DNS settings without being on the VPN 
* Do a speed test from Google
* Go to another tab and go to [https://ipleak.net](https://ipleak.net/)
	* Connect your OpenVPN to your server
	* Refresh the tab
	* You should see your DO server IP address
* Create a snapshot on digital ocean once you see that you 
* ANY Issues for connecting back to OpenVPN 
	* `sudo reboot`
	* `su rose`
	* `sudo ufw status`
	* If all ends fails, shut down your computer and log back into digital ocean and restore your previous snapshot 

OpenVPN Connect
![Pasted image 20230428132329](https://user-images.githubusercontent.com/70050375/235272353-700cab31-8866-4f03-b06a-ca4634695cc4.png)


Tunnelblick
![Pasted image 20230428132358](https://user-images.githubusercontent.com/70050375/235272377-6eef4378-9a88-4e45-aba4-7080d312a309.png)

## Step 16: Installing Wireguard and IPSec using AlgoVPN Scripts

Do a local install on our existing Custom Cloud VPN Server. For more information, click [here](https://github.com/trailofbits/algo).

* `ssh` into your VPN Server
* Get a copy of algo and install Algo’s core dependencies

`cd ~`
`git clone [https://github.com/trailofbits/algo.git](https://github.com/trailofbits/algo.git)`
`sudo apt install -y --no-install-recommends python3-virtualenv`

Install remaining Algo Dependencies 

`cd algo`

```
python3 -m virtualenv --python="$(command -v python3)" .env && 
source .env/bin/activate && 
python3 -m pip install -U pip virtualenv && 
python3 -m pip install -r requirements.txt
```

Edit config.cfg files

* `nano config.cfg`
* Change wireguard port to `53`
* Change ssh port `21999`
* Add/edit names of users for Wireguard and IPSec, if you desire
	* I chose not to as it would have gave me an error personally 

Run Algo install script

* While in the algo directory
	* `sudo ./algo`
	* Answer the questions provided
	* During the 'Cloud Prompt'
		* You need to choose 'Install to existing...' option 
		* It should be option 12
	* Continue answering the questions 
		* Put `localhost` and the DO server IP when prompted

![Pasted image 20230428131935](https://user-images.githubusercontent.com/70050375/235272390-b99505d8-7c52-40d5-b8f5-d5bcab3f0f2a.png)

## Step 17: Test & Fix your OpenVPN Server Configs changed by the Algo scripts

* Turn ufw back on the OpenVPN Server and add the necessary ports 
	* `sudo ufw status`
	* `sudo ufw enable`
		* vertify with `sudo ufw status`
	* `sudo ufw allow 53/udp`
	* `sudo ufw allow 500/udp`
	* `sudo ufw allow 4500/udp`

![Pasted image 20230428132044](https://user-images.githubusercontent.com/70050375/235272398-0b4fbab9-67d9-421c-bae5-03e59023d83f.png)

* Test OpenVPN to ensure that it works like before

## Step 18: Download Wireguard and IPSec Config files / credentials and test on Mac

In the terminal:
`ssh -p 21999 rose@146.190.78.68`

In the droplet, change the permissions of the file:
`sudo chmod 744 /algo/configs/146.190.78.68/wireguard/desktop.conf`

In the terminal, not in the server but terminal in general:
`sudo scp -P 21999 rose@146.190.78.68:~/algo/configs/146.190.78.68/wireguard/desktop.conf ~/Downloads/`

Wireguard:
![Pasted image 20230428134842](https://user-images.githubusercontent.com/70050375/235272413-67a73d0c-cadc-4946-a876-7ba8887bd3d3.png)

**SAVE 2 or more SNAPSHOTS on DIgitalOcean**

## Step 19: Creating Cron Jobs

Click [here](https://www.hostinger.com/tutorials/cron-job) for references.

In Terminal:
`ssh -p 21999 rose@146.190.78.68`
`crontab -e`
	Select editor 
	[1] <-- /bin/nano

See a list of active scheduled task:
`crontab -l`

Cronjobs I have chosen:

`0 0 * * * apt-get update && apt-get upgrade -y`
	Checks for all updates & install upgrades every 24 hours.

`0 1 * * 0 rm -rf /home/rose/.local/share/Trash/`
	Clearing the trash folder in my local system every Sunday at 1 am.

`0 * * * * echo "Hi, it's now $(date +%H:%M). Here’s a reminder to get up, stretch, take a break, and drink water"`
	Displays a message with the current time as well as some useful self care messages every hour on your terminal. This does not perform any actions or operations on your system, it simply displays a message on your terminal.

`0 0 * * * grep "Failed password" /var/log/auth.log > /path/to/logfile.txt`
	Sends all failed login attempts to a file once per day


## Step 20: Installing a different shell 

Installing zsh 
`sudo apt install zsh`

Switching to zsh
`zsh`

Switching to bash
`bash`

Displays current shell name
`ps -p $$`

## Step 21: Color Coding For Both Bash and Zsh 

### Coloring a bash prompt

`nano ~/.bashrc`

Paste the following:
```
bold=$(tput bold)
red=$(tput setaf 1)
green=$(tput setaf 2)
yellow=$(tput setaf 3)
blue=$(tput setaf 4)
purple=$(tput setaf 5)
reset=$(tput sgr0)

PS1="${bold}${red}\u${reset}@${bold}${green}\h${reset}:${bold}${yellow}\w${reset}\$ "

```

### Coloring a zsh prompt

`zsh`

`nano ~.zshrc`

Paste this into the nano file:
`PS1='%F{magenta}%1m%~%f '`

Save and exit.

`source ~.zshrc`
	Prompt should now be magenta

## Step 22: Creating Aliases

`bash`
`nano ~/.bashrc`

In the file, paste the following 
`alias c='clear'`
	clear command
`alias update='sudo apt-get update && sudo apt-get upgrade'`
	updates
`alias ports='netstat -tulanp'`
	show open ports
`alias h='history'`
	shortcut for history
`alias ..='cd ..'`
	quick way to get out of current directory

Save and exit and reload console.

Create snapshots on Digital Ocean.

Now you are all done!
