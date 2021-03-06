# DigitalOcean - FreeBSD
- [Initial setup](#initial-setup) - various steps after booting a fresh FreeBSD instance
  - [accounts](#1-accounts), [sshd](#2-sshd), [time](#3-time), [swap](#4-swap), [droplet.conf](#5-dropletconf), [packages](#6-packages), [update system](#7-update-system), [sources](#8-sources)
- [Floating IP](#floating-ip) - additional routing table for outgoing traffic 

## initial setup
##### 1. accounts
Add new user and copy ssh keys:
```
set _user=newuser
sudo adduser
sudo mkdir /home/${_user}/.ssh
sudo cp /home/freebsd/.ssh/authorized_keys /home/${_user}/.ssh/
sudo chown -R ${_user}:${_user} /home/${_user}/.ssh
unset _user
```
Add sudo access, i.e., run `sudo visudo` and add line:
```
newuser ALL=(ALL) ALL
```
Disconnect and reconnect with new user.  

Remove `freebsd` user:
```
sudo pw groupmod wheel -d freebsd
sudo rmuser freebsd
```
Change `root` password...
```
head -c 1024 /dev/urandom | sha256
sudo passwd root
```
...or lock `root` account:
```
sudo pw lock root
```
##### 2. sshd
Edit `/etc/ssh/sshd_config` configuration (e.g., `sudo vi /etc/ssh/sshd_config`):  

change these lines...
```
#VersionAddendum FreeBSD-20140420
PermitRootLogin without-password
#ChallengeResponseAuthentication yes
```
...to these lines
```
VersionAddendum none
PermitRootLogin no
ChallengeResponseAuthentication no
```
and remove these:
```
Match User root
        ForceCommand echo "Please use the freebsd@ user to access this droplet."
```
Restart sshd:
```
sudo service sshd restart
```

##### 3. time
Set time zone:
```
sudo tzsetup
```
Enable `ntpd`:
```
sudo sh -c 'echo ntpd_enable=\"YES\" >> /etc/rc.conf'
sudo sh -c 'echo ntpd_sync_on_start=\"YES\" >> /etc/rc.conf'
sudo service ntpd start
```

##### 4. swap
Add 2GB swap as file:
```
sudo truncate -s 2G /swap
sudo chmod 0600 /swap
sudo sh -c 'echo "md99 none swap sw,file=/swap,late 0 0" >> /etc/fstab'
sudo swapon -aqL
```

##### 5. droplet.conf
Merge configuration:
```
cat /etc/rc.digitalocean.d/droplet.conf > tmp.conf
cat /etc/rc.conf >> tmp.conf
sudo mv tmp.conf /etc/rc.conf
```
Remove remains of DigitalOcean's initial configuration:
```
sudo rm -rf /etc/rc.digitalocean.d
sudo rm /etc/rc.d/digitalocean
```
Edit `/etc/rc.subr` (e.g., `sudo vi /etc/rc.subr`) and remove these lines:
```
        if [ -L /etc/rc.digitalocean.d/droplet.conf -a -f /etc/rc.digitalocean.d/droplet.conf ]
        then
                . /etc/rc.digitalocean.d/droplet.conf
        fi
```

##### 6. packages
###### unused packages
Stop and remove `avahi`, used for DigitalOcean's initial configuration:
```
sudo pkill avahi-autoipd
sudo pkg delete avahi-app
sudo rmuser avahi
```
Remove other unused packages:
```
sudo pkg autoremove
```

###### update packages
Update ports tree:
```
sudo portsnap fetch update
```
Upgrade `pkg` utility and install `portmaster`:
```
cd /usr/ports/ports-mgmt/pkg
sudo make deinstall reinstall clean
cd /usr/ports/ports-mgmt/portmaster
sudo make install clean
rehash
```
Check and upgrade packages:
```
pkg version -vIL=
sudo portmaster -a
```

##### 7. update system
Disable `/usr/src` updating by editing `/etc/freebsd-update.conf`:
```
-Components src world kernel
+Components world kernel
```
Fetch and install updates:
```
sudo freebsd-update fetch
sudo freebsd-update install
```
Reboot:
```
sudo reboot
```

##### 8. sources
Install `svnup` utility:
```
cd /usr/ports/net/svnup
sudo make install clean
```
Edit `/usr/local/etc/svnup.conf` (e.g., `sudo vi /usr/local/etc/svnup.conf`)  

set appropriate `host`...
```
host=svn0.eu.freebsd.org
```
...and appropriate `branch` under `release` category:
```
branch=base/releng/10.2
```
Update sources:
```
sudo svnup release
```

## floating ip
DigitalOcean provides Floating IP for high availability. It is primarily intended for incoming traffic, but it can also be used for outgoing traffic.  

##### enable
Log into DigitialOcean and enable Floating IP in Networking menu. Afterwards, gather configuration:
```
curl -w '\n' -s http://169.254.169.254/metadata/v1/interfaces/public/0/anchor_ipv4/address
curl -w '\n' -s http://169.254.169.254/metadata/v1/interfaces/public/0/anchor_ipv4/gateway
```
It will output assigned Floating IP and gateway, e.g.:
```
10.50.0.9
10.50.0.1
```

##### configure
Edit `/boot/loader.conf.local` to enable additional routing table:
```
net.fibs=2
net.add_addr_allfibs=0
```

Edit `/etc/rc.conf` to add Floating IP and set up additional routing table configuration:
```
ifconfig_vtnet0_alias0="inet 10.50.0.9 netmask 255.255.0.0"
static_routes="float_if float_gw"
route_float_if="-net 10.50.0.0/24 -iface vtnet0 -fib 1"
route_float_gw="default 10.50.0.1 -fib 1"
```

Reboot.

##### usage
To use new Floating ip routing table, prefix each command with `setfib 1`, e.g.:
```
setfib 1 ping www.github.com
```
