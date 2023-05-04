
_Manual Installation/Setup for Custom Nodes, Raspibolt, Minibolt, etc..._

## Step 1: Configure Virtual Private Server (VPS)

Can obtain pleb-vpn.conf file from @allyourbankarebelongtous (easy) or configure your own VPS following the steps below.

[TODO: Add manual VPS setup steps.]

## Step 2: Install and configure OpenVPN on node.

*This step mimics the actions of `vpn-install.sh` in pleb-vpn.*

Install OpenVPN
```
sudo apt install openvpn
```
Upload openvpn config file and rename to plebvpn.conf
```
[Add code]
sudo mv [yournode].ovpn plebvpn.conf
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

## Steps 3-X can be customized to only install the additional features you want, and can be done in any order you desire.

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

Install cgroup-tools:
```
sudo apt install cgroup-tools
```
Install nft and add ip nat table and POSTROUTING chain:
```
sudo apt install nftables
sudo nft add table ip nat
nft 'add chain ip nat POSTROUTING {type nat hook postrouting priority 100 ; policy accept; }'
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

```
Add pleb-vpn-create-cgroup.service as a requirement to start tor:
```
sudo mkdir /etc/systemd/system/tor@default.service.d
sudo nano /etc/systemd/system/tor@default.service.d/tor-cgroup.conf
```
Fill with:
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
[NEED TO ADD STEPS TO ACTUALLY FINISH ACTIVATING]

## Step 5: Install WireGuard for private remote access

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
