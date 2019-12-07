# roger-skyline_1
roger-skyline 1

# V1. Social stuff

Follow Slash16 on Facebook, Twitter and Linkedin.

# V2. VM Installation

## System specification
Debian 10

## Hardware specification
Disk size 8 GB
One 4.3 GB partition

## Network specification
Static IP 192.168.56.101/24
ssh listens on port 222
apache2 lesten to ports: 80 (HTTP) and 443 (HTTPS)

## User management
root - root user
aya42 - user, added to root and sudo

I used Virtual Box and downloaded the iso image of the latest [Debian10] <https://cdimage.debian.org/debian-cd/current/amd64/iso-cd>

During the installation process, made disc partition 4.3 GB mounted on / root and 4.3GB of swap area

<https://debian-handbook.info/browse/stable/sect.installation-steps.html>  
OR
<http://www.brianlinkletter.com/installing-debian-linux-in-a-virtualbox-virtual-machine/>


# V2. Network and security part

## Install Dependecies 

To have all the currencies updated run:

```
apt -y update
apt -y upgrade

apt-get install -y sudo net-tools iptables-persistent fail2ban sendmail apache2 cron vim 
```

## Configure SUDO

Login as root user
```
su

nano /etc/ssh/sshd_config
```

- add new lines OR change lines (uncomment them to make active) into: 

```
Port 222
PasswordAuthentification yes
PermitRootLogin no
PubkeyAuthentication yes
```

## Setup a static IP

- to find ip (https://vitux.com/find-debian-ip-network-address)

```
ip a
or
ip addr | grep 'inet'
```

- to enable [network_adapter] in VM (https://cs4118.github.io/dev-guides/host-only-network.html).

```
nano / etc / network / interfaces
```

# printscreen place


```
source /etc/network/interfaces.d/*
auto lo 
iface lo inet loopback
allow-hotplug enp0s3 
iface enp0s3 inet dhcp
allow-hotplug enp0s8 
iface enp0s8 inet static
    address 192.168.56.101/24
    gateway 192.168.56.102
    network 192.168.56.100 
    netmask 255.255.255.0
```

## How to add user?

```
adduser <username> 
```
Make it sudo & root
```
adduser <username> sudo
adduser <username> root
```
Also if you want to modify other users: check nano /etc/passwd & change user UID & GID to 0 


- to fix the problem with pub key [access] if appeared (<https://www.digitalocean.com/community/questions/error-permission-denied-publickey-when-i-try-to-ssh>)

Restart the network service to make changes effective

```
sudo service networking restart

ip addr
```

We can now login users with ssh

```
ssh <user>@debian -p 222 

OR

ssh <user>@192.168.56.101 -p 222
```

## Setup public keys acces to ssh

Generate the rsa key

```
ssh-keygen
```
This command will generate 2 files id_rsa and id_rsa.pub

id_rsa: Our private key, should be keep safely, it can be crypted with a password.
id_rsa.pub Our private key, you have to transfer this one to the server.

To copy id_rsa
```
ssh-copy-id -i id_rsa.pub <user>@debian/IP -p 222
OR
mkdir .ssh
~/.ssh/id_rsa.pub > .ssh/authorized_keys
```
Edit the sshd_config file /etc/ssh/sshd.config to remove root login permit, password authentification
```
sudo vim /etc/ssh/sshd.conf
Edit line like: PermitRootLogin no
Edit line like PasswordAuthentication no
```
Don't forget to delete de # at the beginning of each line. Now we can run ssh without password.
```
sudo service ssh restart
```
Try again:
```
ssh <user>@debian/IP -p 22
```
## Setup Firewall
```
sudo nano /etc/network/if-pre-up.d/iptables


//enter this:
#! / Bin / bash
iptables-restore </etc/iptables.test.rules
iptables -F iptables -X iptables -t nat -F iptables -t nat -X iptables -t mangle -F iptables -t mangle -X
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
iptables -A INPUT -m conntrack -ctstate ESTABLISHED, RELATED -j ACCEPT
iptables -A INPUT -p tcp -i enp0s8 -dport 22 -j ACCEPT
iptables -A INPUT -p tcp -i enp0s8 -dport 80 -j ACCEPT
iptables -A INPUT -p tcp -i enp0s8 -dport 443 -j ACCEPT
iptables -A OUTPUT -m conntrack! --ctstate INVALID -j ACCEPT
iptables -I INPUT -i lo -j ACCEPT
iptables -A INPUT -j LOG
iptables -A FORWARD -j LOG
iptables -I INPUT -p tcp -dport 80 -m connlimit -connlimit-above 10 -connlimit-mask 20 -j DROP

#port scan
iptables -N port-scanning
iptables -A port-scanning -p tcp --tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 60/s --limit-burst 2 -j RETURN
iptables -A port-scanning -j DROP
```
```
sudo chmod +x /etc/network/if-pre-up.d/iptables
```
The iptables rules are reset at each reboot. This file will allow the iptables-persistent package to load your rules every time you reboot. Modify port 22 by the port of your ssh


## ALTERNATIVE

Add in iptables
```
## Reset tables configurations
#
echo "📦  Reset tables configurations"

# Flush all rules
echo "   -- Flush iptable rules"
iptables -F
iptables -F -t nat
iptables -F -t mangle
iptables -F -t raw
# Erase all non-default chains
echo "   -- Erase all non-default chains"
iptables -X
iptables -X -t nat
iptables -X -t mangle
iptables -X -t raw


## Trafic configurations
#
echo "⚙️  Trafic configurations"
# Block all traffic by default
echo "   -- Block all traffic by default"
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
echo "   -- Allow all and everything on localhost"
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
# Allow established connections TCP
echo "   -- Allow established connections TCP"
iptables -A INPUT -p tcp -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -p tcp -m state --state ESTABLISHED,RELATED -j ACCEPT
# Allow established connections UDP
echo "   -- Allow established connections UDP"
iptables -A INPUT -p udp -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -p udp -m state --state ESTABLISHED,RELATED -j ACCEPT
# Allow pings
echo "   -- Allow output pings"
iptables -A INPUT -p icmp --icmp-type 0 -j ACCEPT
iptables -A OUTPUT -p icmp --icmp-type 8 -j ACCEPT
# Allow SSH connection on 69 port
echo "   -- Allow SSH connection on 69 port"
iptables -A INPUT -p tcp --dport 69 -j ACCEPT
# Allow HTTP
echo "   -- Allow HTTP"
iptables -t filter -A OUTPUT -p tcp --dport 80 -j ACCEPT
iptables -t filter -A INPUT -p tcp --dport 80 -j ACCEPT
# Allow HTTPS
echo "   -- Allow HTTPS"
iptables -t filter -A OUTPUT -p tcp --dport 443 -j ACCEPT
iptables -t filter -A INPUT -p tcp --dport 443 -j ACCEPT


## Mail configurations
#
echo "✉️  Mail configurations"
# Allow SMTP
echo "   -- Allow SMTP"
iptables -t filter -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -t filter -A OUTPUT -p tcp --dport 22 -j ACCEPT
# Allow POP3
echo "   -- Allow POP3"
iptables -t filter -A INPUT -p tcp --dport 110 -j ACCEPT
iptables -t filter -A OUTPUT -p tcp --dport 110 -j ACCEPT
# Allow IMAP
echo "   -- Allow IMAP"
iptables -t filter -A INPUT -p tcp --dport 143 -j ACCEPT
iptables -t filter -A OUTPUT -p tcp --dport 143 -j ACCEPT
# Allow POP3S
echo "   -- Allow POP3S"
iptables -t filter -A INPUT -p tcp --dport 995 -j ACCEPT
iptables -t filter -A OUTPUT -p tcp --dport 995 -j ACCEPT


## DDoS protections
#
echo "🛡  DDoS protetion"

# Synflood protection
echo "   -- Synflood protection"
/sbin/iptables -A INPUT -p tcp --syn -m limit --limit 2/s --limit-burst 30 -j ACCEPT
# Pingflood protection
echo "   -- Pingflood protection"
/sbin/iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
# Port scanning protection
echo "   -- Port scanning protection"
/sbin/iptables -A INPUT -p tcp --tcp-flags ALL NONE -m limit --limit 1/h -j ACCEPT
/sbin/iptables -A INPUT -p tcp --tcp-flags ALL ALL -m limit --limit 1/h -j ACCEPT


## Save the tables configuration
#
iptables-save > /etc/iptables/rules.v4
```

## Setup logs

Creates log and send info if files were changed and protects from ddos
```
sudo touch /var/log/apache2/server.log
sudo nano /etc/fail2ban/jail.local
```
[DEFAULT] destemail = USER@student.42.us.org sender = root@debian

#just to make sure do it again but now in crontab:
```
sudo nano/etc/cron.d/packages.sh
```
```
sudo apt-get -y update > /var/log/update_script.log && sudo apt-get -y upgrade >> /var/log/update_script.log

sudo nano /etc/cron.d/survey.sh
```

```
if [[ $(($(date +%s) - $(date +%s -r /etc/crontab))) -lt 86400 ]]
then
	echo "Crontab file has been modified" | sudo /usr/sbin/sendmail root
fi
```

crontab -e

```
0 4 * * 1 /etc/cron.d/packages.sh
@reboot /etc/cron.d/packages.sh
0 0 * * * /etc/cron.d/survey.sh
```

{946e5835-c955-400e-b287-65eeb2a7a4a8}.vhd