
_Manual Installation/Setup for Custom Nodes, Raspibolt, Minibolt, etc..._

## Step 1: Configure Virtual Private Server (VPS)

Can obtain pleb-vpn.conf file from @allyourbankarebelongtous (easy), another pleb running their own VPS, or configure your own following the steps below (advanced).

*Manual VPS setup*

The first thing you need to do is spin up a VPS. This can be done with any cloud vendor that lets you install and manage your OS directly: Amazon Web Services (AWS), Google Cloud Platform (GCP), Linode, Digital Ocean, etc... It doesn't need to be very powerful, and you can even use the free tier on AWS for a year (t2.micro) or GCP forever (e2.micro following https://dev.to/phocks/how-to-get-a-free-google-server-forever-1fpf). If you are just running a single node off of it, the smallest size available on your vendor will probably be sufficient. If you want to run a BTCPayServer instance for a large store doing a lot of business, or offer it up for other plebs to use for their nodes too, you may need a larger instance but you can scale up when the need arises. Once you have the VPS procured, install your preferred flavor of linux, though Debian or Ubuntu will work best with the commands in this guide. You will also need to follow the instructions for your VPS provider to open ports 22, 80, 443, 1194, 9735 (lnd) or 9736 (cln), 9993, and possibly others depending on what services you activate below. When that is done, log in and proceed with setup.

First, update apt packages:
```
sudo apt update
sudo apt upgrade
```
Now create a user 'pivpn', create a password for them, and add them to the sudo group:
```
sudo adduser pivpn
sudo usermod -aG sudo pivpn
```
Configure the firewall:
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default allow routed
sudo ufw allow ssh
sudo ufw allow 443
sudo ufw allow 1194/udp comment 'OpenVPN'
sudo ufw enable
```
Edit `/etc/sysctl.conf` to enable ip forwards.
```
sudo nano /etc/sysctl.conf
```
Uncomment the following line, then save and exit:
```
net.ipv4.ip_forward=1
```
Add fail2ban for additional hardening:
```
sudo apt install fail2ban
```
Install pivpn. Use user `pivpn`, choose `openvpn`, and accept all defaults. You can learn more about this and what it's doing here: https://pivpn.io
```
curl -L https://install.pivpn.io | bash
```
Check to see if nftables is installed:
```
nft --version
```
If not installed, install via:
```
sudo apt install nftables
```
Get the OpenVPN cert for plebvpn. Enter a name for the client (your node) when prompted.
```
sudo pivpn add nopass
```
Get the virtual IP address. For "clientname", use the name you chose in the previous step.
```
cd /etc/openvpn/ccd
sudo cat "clientname"
```
The virtual IP address is the first IP after the name of the client. Write this down or copy it to a text editor as you will need it for future steps.

Get the VPS internet controller:
```
ip route | grep default
```
This will output a line that looks like the following. In this example, `eth0` is what we're looking for.
```
default via 68.183.112.1 dev eth0 proto static ...
```
Now you need to forward ports coming into your VPS to your node via the VPN. If you want to learn more about what this is doing, check out https://jensd.be/1086/linux/forward-a-tcp-port-to-another-ip-or-port-using-nat-with-nftables. In all lines below, replace vip.ip.ip.ip with the virtual ip of your client from before, and `eth0` with your internet controller from the previous step.
There should be a way to do this with nftables commands, but I have had issues getting them to forward properly, so we are going to use the legacy iptables commands. These *should* still get automatically interpreted and set as nftables rules.

Forward port 9735 for lnd:
```
sudo iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 9735 -j DNAT --to vip.ip.ip.ip:9735
sudo iptables -A PREROUTING -t nat -i eth0 -p udp -m udp --dport 9735 -j DNAT --to vip.ip.ip.ip:9735
```
Forward port 9736 for cln:
```
sudo iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 9736 -j DNAT --to vip.ip.ip.ip:9736
sudo iptables -A PREROUTING -t nat -i eth0 -p udp -m udp --dport 9736 -j DNAT --to vip.ip.ip.ip:9736
```
Forward port 9993 for wireguard:
```
sudo iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 9993 -j DNAT --to vip.ip.ip.ip:9993
sudo iptables -A PREROUTING -t nat -i eth0 -p udp -m udp --dport 9993 -j DNAT --to vip.ip.ip.ip:9993
```
Forward port 443 to 5001 for LNBits. If you want to use BTCPayServer instead, skip these lines and execute the ones below. If you want to be able to forward to both, you need to set up nginx on your VPS and it's a bit more complicated but can be done.
```
sudo iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 443 -j DNAT --to vip.ip.ip.ip:5001
```
Forward port 443 to 23001 for BTCPayServer. Can't be done at the same time as LNBits above - you have to use nginx to configure both simultaneously.
```
sudo iptables -A PREROUTING -t nat -i eth0 -p tcp -m tcp --dport 443 -j DNAT --to vip.ip.ip.ip:23001
```
Adjust postrouting masquerade so all packets appear to be coming from the VPS:
```
sudo iptables -t nat -A POSTROUTING -d vip.ip.ip.0/24 -o tun0 -j MASQUERADE
```
Now you can logout of the VPS and download the configuration file to your laptop.
```
logout
```
On your home computer/laptop, execute the following command. If you want to save the file in a particular folder, navigate to that folder with `cd`, then execute the `scp` command. Don't forget the `.`!
```
scp -r root@vpsip.ip.ip.ip:/home/pivpn/ovpns .
```
You should have a directory on your home computer where you executed the previous command called `ovpns`. Rename the clientname.ovpn file in that folder to `plebvpn.conf`. You will upload this to your node in step 2.

## Step 2: Install and configure OpenVPN on node.

*This step mimics the actions of `vpn-install.sh` in pleb-vpn.*

First, before connecting to your node via ssh, upload the `plebvpn.conf` file you got from @allyourbankarebelongtous or that you created yourself to your node. To work as written, you must be in the same folder as where you saved the `plebvpn.conf` file.
```
scp plebvpn.conf admin@your.node.ip.ip:/home/admin/
```
SSH into your node, then...
Install OpenVPN
```
sudo apt install openvpn
```
Move openvpn config file and fix ownership/permissions
```
mv ~/plebvpn.conf /etc/openvpn
sudo chown admin:admin /etc/openvpn/plebvpn.conf
sudo chmod 640 /etc/openvpn/plebvpn.conf
```
Enable and start Open VPN
```
sudo systemctl enable openvpn@plebvpn
sudo systemctl start openvpn@plebvpn
```
Wait about 10 seconds, then check to see if it's working:
```
curl https://api.ipify.org
```
The resulting IP address should be your VPS IP. If not, something went wrong.

Configure your firewall:
```
# disable firewall 
sudo ufw disable
# allow local lan ssh
sudo ufw allow in to ${LAN}.0/24
sudo ufw allow out to ${LAN}.0/24
# set default policy (killswitch)
sudo ufw default deny outgoing
sudo ufw default deny incoming
# allow out on openvpn
sudo ufw allow out to ${vpnIP} port ${vpnPort} proto udp
# force traffic to use openvpn
sudo ufw allow out on tun0 from any to any
sudo ufw allow in on tun0 from any to any
# enable firewall
sudo ufw --force enable
```
Test firewall:
```
sudo systemctl stop openvpn@plebvpn
curl https://api.ipify.org
# curl step should take quite a while and ultimately be blank (cannot access clearnet without vpn). 
# Should get error message:
curl: (28) Failed to connect to api.ipify.org port 443: Connection timed out
sudo systemctl start openvpn@plebvpn
sleep 10
curl https://api.ipify.org
```
The resulting IP address should again be your VPS IP. If not, or if curl returned something with the VPN disabled, something went wrong.

## Steps 3-6 can be customized to only install the additional features you want, and can be done in any order you desire.

## Step 3: Enable LND TOR/Clearnet Hybrid Mode

*This step mimics the actions of `lnd-hybrid.sh` in pleb-vpn.*

Adjust firewall:
```
sudo ufw allow <lnPort> comment "LND Port"
```
Edit lnd.conf to enable hybrid mode via the VPS:
```
sudo nano /data/lnd/lnd.conf
```
Add the following lines to the `[Application Options]` section. (Replace "yourVPSIP.IP.IP.IP" with the IP address of your VPS and "lndPort" with the port forwarded from your VPS for LND.)
```
externalip=yourVPSIP.IP.IP.IP:lndPort
listen=0.0.0.0:lndPort
```
And change or the following lines to the `[tor]` section:
```
tor.streamisolation=false
tor.skip-proxy-for-clearnet-targets=true
```
When complete, restart lnd:
```
sudo systemctl restart lnd
```
If you don't have wallet auto-unlock enabled, unlock your wallet:
```
lncli unlock
```
Finally, verify your new clearnet IP:
```
lncli getinfo
```
Output should include `'your public key'@yourVPSIP.IP.IP.IP:lndPort` in the "uris" section.

## Step 4: Activate TOR split-tunneling.

*This step mimics the actions of `tor.split-tunnel.sh` in pleb-vpn.*

*This is by far the hardest and most complex part of setting up pleb-vpn. Take it slow and execute each command carefully.*

Install cgroup-tools:
```
sudo apt install cgroup-tools
```
Install nft and add ip nat table and POSTROUTING chain:
```
sudo apt install nftables
sudo nft add table ip nat
nft 'add chain ip nat POSTROUTING {type nat hook postrouting priority -100 ; policy accept; }'
```
Clean rules from failed install attempts:
```
OIFNAME=$(ip r | grep default | cut -d " " -f5)
GATEWAY=$(ip r | grep default | cut -d " " -f3)
ip_nat_handles=$(nft -a list table ip nat | grep "meta cgroup 1114129 counter" | sed "s/.*handle //")
while [ $(nft list table ip nat | grep -c "meta cgroup 1114129 counter") -gt 0 ]; \
do \
  ruleNumber=$(nft list table ip nat | grep -c "meta cgroup 1114129 counter") \
  && ip_nat_handle=$(echo "${ip_nat_handles}" | sed -n ${ruleNumber}p) \
  && nft delete rule ip nat POSTROUTING handle ${ip_nat_handle} ;\
done

while [ $(nft list tables | grep -c mangle) -gt 0 ]; \
do
  nft delete table ip mangle
done
ip_filter_input_handles=$(nft -a list chain ip filter ufw-user-input | grep "meta cgroup 1114129 counter" | sed "s/.*handle //")
while [ $(nft list chain ip filter ufw-user-input | grep -c "meta cgroup 1114129 counter") -gt 0 ]
do
  ruleNumber=$(nft list chain ip filter ufw-user-input | grep -c "meta cgroup 1114129 counter")
  ip_filter_input_handle=$(echo "${ip_filter_input_handles}" | sed -n ${ruleNumber}p)
  nft delete rule ip filter ufw-user-input handle ${ip_filter_input_handle}
done
ip_filter_output_handles=$(nft -a list chain ip filter ufw-user-output | grep "meta cgroup 1114129 counter" | sed "s/.*handle //")
while [ $(nft list chain ip filter ufw-user-output | grep -c "meta cgroup 1114129 counter") -gt 0 ]
do
  ruleNumber=$(nft list chain ip filter ufw-user-output | grep -c "meta cgroup 1114129 counter")
  ip_filter_output_handle=$(echo "${ip_filter_output_handles}" | sed -n ${ruleNumber}p)
  nft delete rule ip filter ufw-user-output handle ${ip_filter_output_handle}
done
while [ $(ip rule | grep -c "fwmark 0xb lookup novpn") -gt 0 ]
do
  ip rule del from all table novpn fwmark 11
done
while [ $(ip rule | grep -c novpn) -gt 0 ]
do
  ip rou del from all table novpn default via ${GATEWAY}
done
```
Create group novpn
```
sudo groupadd novpn
```
Create routing table
```
echo "1000 novpn" | sudo tee /etc/iproute2/rt_tables.d/novpn-route.conf
```
Create create-cgroup.sh (this is all one big command)
```
mkdir /home/admin/pleb-vpn/split-tunnel
chmod 755 -R /home/admin/pleb-vpn/split-tunnel

sudo modprobe cls_cgroup
sudo mkdir /sys/fs/cgroup/net_cls

sudo mount -t cgroup -o net_cls novpn /sys/fs/cgroup/net_cls
sudo cgcreate -t debian-tor:novpn -a debian-tor:novpn -d 775 -f 664 -s 664 -g net_cls:novpn
echo 0x00110011 > /sys/fs/cgroup/net_cls/novpn/net_cls.classid # must be done as root...need to figure out how to change that
```
Make it permanent:
```
sudo nano /etc/systemd/system/pleb-vpn-create-cgroup.service
```
Fill with:
```
[Unit]
Description=Creates cgroup for split-tunneling tor from vpn
StartLimitInterval=200
StartLimitBurst=5
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash /home/admin/pleb-vpn/split-tunnel/create-cgroup.sh
[Install]
WantedBy=multi-user.target
```
Add pleb-vpn-create-cgroup.service as a requirement to start tor:
```
sudo mkdir /etc/systemd/system/tor@default.service.d
sudo nano /etc/systemd/system/tor@default.service.d/tor-cgroup.conf
```
Fill with the following, then save and exit.
```
#Don't edit this file, it is created by pleb-vpn split-tunnel
[Unit]
Requires=pleb-vpn-create-cgroup.service
After=pleb-vpn-create-cgroup.service
```
Create tor-split-tunnel.sh in pleb-vpn/split-tunnel to add tor to cgroup for split-tunnel
```
echo '#!/bin/bash

# adds tor to cgroup for split-tunneling
tor_pid=$(pgrep -x tor)
cgclassify -g net_cls:novpn $tor_pid
' | sudo tee /home/admin/pleb-vpn/split-tunnel/tor-split-tunnel.sh

sudo chmod 755 -R /home/admin/pleb-vpn/split-tunnel
```
Run tor-split-tunnel.sh
```
sudo /home/admin/pleb-vpn/split-tunnel/tor-split-tunnel.sh
```
create pleb-vpn-tor-split-tunnel.service
```
echo "[Unit]
Description=Adding tor process to cgroup novpn
Requires=tor@default.service
After=tor@default.service
[Service]
Type=oneshot
ExecStart=/bin/bash /home/admin/pleb-vpn/split-tunnel/tor-split-tunnel.sh
[Install]
WantedBy=multi-user.target
" | sudo tee /etc/systemd/system/pleb-vpn-tor-split-tunnel.service
```
create pleb-vpn-tor-split-tunnel.timer
```
echo "[Unit]
Description=1 min timer to add tor process to cgroup novpn
[Timer]
OnBootSec=10
OnUnitActiveSec=10
Persistent=true
[Install]
WantedBy=timers.target
" | sudo tee /etc/systemd/system/pleb-vpn-tor-split-tunnel.timer
```
Create nftables-config.sh in pleb-vpn/split-tunnel...
```
echo '#!/bin/bash

# adds route and rules to allow novpn cgroup past firewall

# first clean tables from existing duplicate rules
OIFNAME=$(ip r | grep default | cut -d " " -f5)
GATEWAY=$(ip r | grep default | cut -d " " -f3)

# first check for and remove old names from prior starts
ip_nat_handles=$(nft -a list table ip nat | grep "meta cgroup 1114129 counter" | sed "s/.*handle //")
while [ $(nft list table ip nat | grep -c "meta cgroup 1114129 counter") -gt 0 ]
do
  ruleNumber=$(nft list table ip nat | grep -c "meta cgroup 1114129 counter")
  ip_nat_handle=$(echo "${ip_nat_handles}" | sed -n ${ruleNumber}p)
  nft delete rule ip nat POSTROUTING handle ${ip_nat_handle}
done
while [ $(nft list tables | grep -c mangle) -gt 0 ]
do
  nft delete table ip mangle
done
ip_filter_input_handles=$(nft -a list chain ip filter ufw-user-input | grep "meta cgroup 1114129 counter" | sed "s/.*handle //")
while [ $(nft list chain ip filter ufw-user-input | grep -c "meta cgroup 1114129 counter") -gt 0 ]
do
  ruleNumber=$(nft list chain ip filter ufw-user-input | grep -c "meta cgroup 1114129 counter")
  ip_filter_input_handle=$(echo "${ip_filter_input_handles}" | sed -n ${ruleNumber}p)
  nft delete rule ip filter ufw-user-input handle ${ip_filter_input_handle}
done
ip_filter_output_handles=$(nft -a list chain ip filter ufw-user-output | grep "meta cgroup 1114129 counter" | sed "s/.*handle //")
while [ $(nft list chain ip filter ufw-user-output | grep -c "meta cgroup 1114129 counter") -gt 0 ]
do
  ruleNumber=$(nft list chain ip filter ufw-user-output | grep -c "meta cgroup 1114129 counter")
  ip_filter_output_handle=$(echo "${ip_filter_output_handles}" | sed -n ${ruleNumber}p)
  nft delete rule ip filter ufw-user-output handle ${ip_filter_output_handle}
done
while [ $(ip rule | grep -c "fwmark 0xb lookup novpn") -gt 0 ]
do
  ip rule del from all table novpn fwmark 11
done
while [ $(ip rule | grep -c novpn) -gt 0 ]
do
  ip rou del from all table novpn default via ${GATEWAY}
done

# add/refresh rules
nft add rule ip nat POSTROUTING oifname ${OIFNAME} meta cgroup 1114129 counter masquerade
nft add table ip mangle
nft add chain ip mangle markit "{type route hook output priority filter; policy accept;}"
nft add rule ip mangle markit meta cgroup 1114129 counter meta mark set 0xb
nft add rule ip filter ufw-user-input meta cgroup 1114129 counter accept
nft add rule ip filter ufw-user-output meta cgroup 1114129 counter accept
ip route add default via ${GATEWAY} table novpn
ip rule add fwmark 11 table novpn
' | tee /home/admin/pleb-vpn/split-tunnel/nftables-config.sh
```
Update permissions:
```
chmod 755 -R /home/admin/pleb-vpn/split-tunnel
```
Run it once:
```
sudo /home/admin/pleb-vpn/split-tunnel/nftables-config.sh
```
Create nftables-config systemd service...
```
echo "[Unit]
Description=Configure nftables for split-tunnel process
Requires=pleb-vpn-tor-split-tunnel.service
After=pleb-vpn-tor-split-tunnel.service
[Service]
Type=oneshot
ExecStart=/bin/bash /home/admin/pleb-vpn/split-tunnel/nftables-config.sh
[Install]
WantedBy=multi-user.target
" | sudo tee /etc/systemd/system/pleb-vpn-nftables-config.service
```
Now enable all services:
```
sudo systemctl daemon-reload
sudo systemctl enable pleb-vpn-create-cgroup.service
sudo systemctl enable pleb-vpn-tor-split-tunnel.service
sudo systemctl enable pleb-vpn-nftables-config.service
sudo systemctl enable pleb-vpn-tor-split-tunnel.timer
```
Restart tor to pick up new split-tunnel configuration:
```
sudo systemctl stop tor@default.service
sudo systemctl start pleb-vpn-create-cgroup.service
sudo systemctl start tor@default.service
sudo systemctl start pleb-vpn-tor-split-tunnel.service
sudo systemctl start pleb-vpn-nftables-config.service
sudo systemctl start pleb-vpn-tor-split-tunnel.timer
```
If you want to be able to test your split-tunneling configuration, create a test script in user `admin`'s home directory as follows:
```
cd ~
nano split-tunnel-test.sh
```
Fill with the following, then save and exit.
```
#!/bin/bash

  # check configuration
  echo "OK...tor is configured. Wait 2 minutes for tor to start..."
  sleep 60
  echo "wait 1 minute for tor to start..."
  sleep 60
  echo "checking configuration"
  echo "stop vpn"
  systemctl stop openvpn@plebvpn
  echo "vpn stopped"
  echo "checking firewall"
  currentIP=$(curl https://api.ipify.org)
  echo "current IP = (${currentIP})...should be blank"
  if [ "${currentIP}" = "" ]; then
    echo "firewall config ok"
  else
    echo "error...firewall not configured. Clearnet accessible when VPN is off. Re-attempt pleb-vpn install steps, especially firewall configuration."
    systemctl start openvpn@plebvpn
    exit 1
  fi
  echo "Checking connection over tor with VPN off (takes some time, likely multiple tries)..."
  echo "Will attempt a connection up to 10 times before giving up..."
  inc=1
  while [ $inc -le 10 ]
  do
    echo "attempt number ${inc}"
    torIP=$(torify curl http://api.ipify.org)
    echo "tor IP = (${torIP})...should not be blank, should not be your home IP, and should not be your VPS IP."
    if [ ! "${torIP}" = "" ]; then
      inc=11
    else
      ((inc++))
    fi
  done
  if [ ! "${torIP}" = "" ]; then
    echo "tor split-tunnel successful"
  else
    echo "error...unable to connect over tor when VPN is down. It's possible that it needs more time to establish a connection. Try checking the status using STATUS menu later. If unable to connect, re-attempt tor split-tunneling install instructions."
  fi
  sleep 2
  echo "restarting vpn"
  systemctl start openvpn@plebvpn
  sleep 2
  echo "checking vpn IP"
  currentIP=$(curl https://api.ipify.org)
  echo "current IP = (${currentIP})...should be your VPS IP address"
  echo "tor split-tunneling enabled!"
  sleep 2
```
Make it executable:
```
sudo chmod +x split-tunnel-test.sh
```
To test your VPN and split-tunnel configuration, run the script with `sudo bash split-tunnel-test.sh`. This will first test your firewall settings by stopping your openvpn VPN, then try to check your IP address using `curl https://api.ipify.org`. It should timeout without a connection and show a blank IP. Then it will try to check your IP using the same command, but with `torify` to go over tor. It should return an IP different from your home IP, and different from your VPS IP, but should not be blank. This step can take several times to succeed due to the nature of tor. The script will try up to 10 times. Finally, it will re-enable your openvpn VPN and try the `curl` command one more time. This time, it should return the IP address of your VPS.

Congratulations, you have successfully configured your node with clearnet/tor hybrid connectivity and split-tunneling of tor traffic!

## Step 5: Install WireGuard for private remote access

*This step mimics the actions of `wg-install.sh` in pleb-vpn.*

Install WireGuard
```
sudo apt update
sudo apt install wireguard
```
Generate private key, public key, and write to WireGuard configuration file
```
(umask 077 && printf "[Interface]\nPrivateKey = " | sudo tee /etc/wireguard/wg0.conf > /dev/null)
wg genkey | sudo tee -a /etc/wireguard/wg0.conf | wg pubkey | sudo tee /etc/wireguard/server_public_key
sudo nano /etc/wireguard/wg0.conf
```
The first two lines should already be in your wg0.conf file. Add the rest, taking care to input *your* port (forwarded from your VPS). For address, can enter any private IP address *that is not in your local LAN range* you like.
```
[Interface]
PrivateKey = (generated_private_key)
ListenPort = 5555
SaveConfig = true
Address = 10.0.0.0/24

PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```
Generate client keys and config files - can customize names and addresses as you see fit.
```
serverPublicKey=$(sudo cat /etc/wireguard/server_public_key)
sudo wg genkey | sudo tee /etc/wireguard/client1_private_key | wg pubkey > /etc/wireguard/client1_public_key
client1PrivateKey=$(sudo cat /etc/wireguard/client1_private_key)
client1PublicKey=$(sudo cat /etc/wireguard/client1_public_key)
sudo wg genkey | sudo tee /etc/wireguard/client2_private_key | wg pubkey > /etc/wireguard/client2_public_key
client2PrivateKey=$(sudo cat /etc/wireguard/client2_private_key)
client2PublicKey=$(sudo cat /etc/wireguard/client2_public_key)
sudo wg genkey | sudo tee /etc/wireguard/client3_private_key | wg pubkey > /etc/wireguard/client3_public_key
client3PrivateKey=$(sudo cat /etc/wireguard/client3_private_key)
client3PublicKey=$(sudo cat /etc/wireguard/client3_public_key)
wgIP= # enter your desired WireGuard IP address here, from 10.0.0.0 to 10.255.255.255. Must be different from your home LAN.
wgLAN=$(echo "${wgIP}" | sed 's/^\(.*\)\.\(.*\)\.\(.*\)\.\(.*\)$/\1\.\2\.\3/')
serverHost=$(echo "${wgIP}" | cut -d "." -f4)
client1Host=$(expr $serverHost + 1)
client2Host=$(expr $serverHost + 2)
client3Host=$(expr $serverHost + 3)
client1ip=$(echo "${wgLAN}.${client1Host}")
client2ip=$(echo "${wgLAN}.${client2Host}")
client3ip=$(echo "${wgLAN}.${client3Host}")

echo "[Peer]
PublicKey = ${client1PublicKey}
AllowedIPs = ${client1ip}/32

[Peer]
PublicKey = ${client2PublicKey}
AllowedIPs = ${client2ip}/32

[Peer]
PublicKey = ${client3PublicKey}
AllowedIPs = ${client3ip}/32
" >> /etc/wireguard/wg0.conf

sudo mkdir /etc/wireguard/clients

echo "[Interface]
Address = ${client1ip}/32
PrivateKey = ${client1PrivateKey}

[Peer]
PublicKey = ${serverPublicKey}
Endpoint = ${vpnIP}:${wgPort}
AllowedIPs = ${wgLAN}.0/24, ${LAN}.0/24
" | sudo tee /etc/wireguard/clients/mobile.conf
    echo "[Interface]
Address = ${client2ip}/32
PrivateKey = ${client2PrivateKey}

[Peer]
PublicKey = ${serverPublicKey}
Endpoint = ${vpnIP}:${wgPort}
AllowedIPs = ${wgLAN}.0/24, ${LAN}.0/24
" | sudo tee /etc/wireguard/clients/laptop.conf
    echo "[Interface]
Address = ${client3ip}/32
PrivateKey = ${client3PrivateKey}

[Peer]
PublicKey = ${serverPublicKey}
Endpoint = ${vpnIP}:${wgPort}
AllowedIPs = ${wgLAN}.0/24, ${LAN}.0/24
" | sudo tee /etc/wireguard/clients/desktop.conf
```
Open firewall ports
```
sudo ufw allow ${wgPort}/udp comment "wireguard port"
sudo ufw allow out on wg0 from any to any
sudo ufw allow in on wg0 from any to any
sudo ufw allow in to ${wgLAN}.0/24
sudo ufw allow out to ${wgLAN}.0/24
```
Enable ip forward
```
sudo sed -i '/net.ipv4.ip_forward/ s/#//' /etc/sysctl.conf
```
Enable systemd and fix permissions
```
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```
Wait 10 seconds, then start wireguard
```
sleep 10
sudo systemctl restart wg-quick@wg0
```
If running LND, add tlsextraip and refersh certs:
```
sudo nano /data/lnd/lnd.conf
```
Add the following line under "[Application Options]". Be careful to add your node's WireGuard IP.
```
tlsextraip=<node wgIP>
```
Save and exit. Then:
```
sudo rm /mnt/hdd/lnd/*.old # deletes existing old certs if you've already done this once.
sudo mv /mnt/hdd/lnd/tls.cert /mnt/hdd/lnd/tls.cert.old
sudo mv /mnt/hdd/lnd/tls.key /mnt/hdd/lnd/tls.key.old

sudo systemctl restart lnd
```
If wallet not set to auto-unlock, unlock wallet, then restart nginx:
```
lncli unlock
sudo systemctl restart nginx
```

## Step 6: Set up LetsEncrypt certificates for BTCPayServer and/or LNBits

[TODO: Complete]
