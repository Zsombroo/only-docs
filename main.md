# Fedora Server

## SSH key setup on a remote system

```
ssh-copy-id -i ~/.ssh/id_ed25519.pub username@remote_server_ip
```

## Extend the filesystem to the fill disk

Find the root filesystem's path. Look for whatever is mounted on /

```
df -h

Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/fedora-root        157G   57G  100G   3% /
```

The first name is the one we need to use to extend the FS with the following two commands:

```
lvextend -l +100%FREE /dev/mapper/fedora-root
xfs_growfs /
```

Next time you check the size, you will see it occupies the full available space.

```
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/fedora-root        2.8T   57G  2.7T   3% /
```

## WireGuard

### As a Hub

These steps are used to set up a central hub / main server for your VPN. For each new peer joining the network you have to add them to the host's config on top of configuring the new peer.

```
sudo su
dnf install wireguard-tools -y

# Allow forwarding between interfaces
echo "net.ipv4.ip_forward=1" | tee -a /etc/sysctl.conf
sysctl -p

# Generate the keys. The files themselves won't be needed, just their content.
cd /etc/wireguard/
wg genkey | tee private.key | wg pubkey > public.key

vi /etc/wireguard/wg0.conf

### Paste this into wg0.conf

[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = <private_key>

[Peer]
PublicKey = <peer_public_key>
AllowedIPs = 10.10.0.2/32

### End Paste

systemctl enable wg-quick@wg0 --now
systemctl restart wg-quick@wg0

firewall-cmd --permanent --add-port=51820/udp
firewall-cmd --zone=trusted --add-interface=wg0 --permanent
firewall-cmd --reload
```

### As a Peer

```
sudo su
dnf install wireguard-tools -y
cd /etc/wireguard/
wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key

vi wg0.conf

### Paste into wg0.conf

[Interface]
Address = 10.10.0.2/24
PrivateKey = <generated_private.key>

[Peer]
PublicKey = <server_public_key>
Endpoint = <domain_name_in_cloudflare>:51820
AllowedIPs = 10.10.0.0/24
PersistentKeepalive = 25

### End Pase

systemctl enable wg-quick@wg0 --now
sudo systemctl restart wg-quick@wg0
```

Allow the client to join the network on the WireGuard server

```
sudo su
vi /etc/wireguard/wg0.conf

### Add this section

PublicKey = <generated_public_key>
AllowedIPs = 10.10.0.2/32

### End Add

sudo systemctl restart wg-quick@wg0
```
