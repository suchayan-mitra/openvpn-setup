# OpenVPN Setup on Azure VM

## Step-by-Step Guide

### Initial Setup

1. **Create an Azure VM**:
   - Choose Ubuntu 20.04 or later as the operating system.
   - Open necessary ports in Azure Network Security Group (NSG): 22 (SSH), 1194 (UDP).

2. **SSH into the VM**:
   ```bash
   ssh your_username@your_vm_ip

3. **Update and install necessary packages**:
   ```bash
   sudo apt update
   sudo apt upgrade -y
   sudo apt install openvpn easy-rsa -y
   ```

### Configuration

1. **Setup Easy-RSA**:
   ```bash
   make-cadir ~/easy-rsa
   cd ~/easy-rsa
   ```

2. **Edit vars file**:
   Open `~/easy-rsa/vars` with a text editor and adjust the settings (optional).

3. **Generate keys and certificates**:
   ```bash
   ./easyrsa init-pki
   ./easyrsa build-ca nopass
   ./easyrsa gen-req server nopass
   ./easyrsa sign-req server server
   ./easyrsa gen-dh
   ./easyrsa gen-req client-us nopass
   ./easyrsa sign-req client client-us
   openvpn --genkey --secret ta.key
   ```

4. **Move keys and certificates to the appropriate location**:
   ```bash
   sudo cp pki/ca.crt pki/issued/server.crt pki/private/server.key pki/dh.pem /etc/openvpn/
   sudo cp ta.key /etc/openvpn/
   sudo cp pki/issued/client-us.crt pki/private/client-us.key /etc/openvpn/client/
   ```

### Server Configuration

1. **Create server configuration file**:
   ```bash
   sudo nano /etc/openvpn/server.conf
   ```

   Add the following configuration:
   ```conf
   port 1194
   proto udp
   dev tun
   cipher AES-256-GCM
   auth SHA256
   ca /etc/openvpn/ca.crt
   cert /etc/openvpn/server.crt
   key /etc/openvpn/server.key
   dh /etc/openvpn/dh.pem
   tls-auth /etc/openvpn/ta.key 0
   server 10.8.0.0 255.255.255.0
   ifconfig-pool-persist ipp.txt
   keepalive 10 120
   persist-key
   persist-tun
   status openvpn-status.log
   verb 3
   push "redirect-gateway def1 bypass-dhcp"
   push "dhcp-option DNS 208.67.222.222"
   push "dhcp-option DNS 208.67.220.220"
   log /var/log/openvpn.log
   topology subnet
   ```

2. **Enable IP forwarding**:
   ```bash
   sudo nano /etc/sysctl.conf
   ```
   Uncomment or add the line:
   ```conf
   net.ipv4.ip_forward = 1
   ```

   Apply the changes:
   ```bash
   sudo sysctl -p
   ```

3. **Configure firewall to allow traffic through VPN**:
   ```bash
   sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
   sudo apt install iptables-persistent
   sudo netfilter-persistent save
   ```

### Client Configuration

1. **Create client configuration file**:
   ```bash
   nano ~/client-us.ovpn
   ```

   Add the following configuration:
   ```conf
   client
   dev tun
   proto udp
   remote YOUR_SERVER_IP 1194
   resolv-retry infinite
   nobind
   persist-key
   persist-tun
   remote-cert-tls server
   auth SHA256
   verb 3
   key-direction 1
   keepalive 10 120
   data-ciphers AES-256-GCM:AES-128-GCM:CHACHA20-POLY1305

   <ca>
   -----BEGIN CERTIFICATE-----
   [Your CA Certificate]
   -----END CERTIFICATE-----
   </ca>

   <cert>
   -----BEGIN CERTIFICATE-----
   [Your Client Certificate]
   -----END CERTIFICATE-----
   </cert>

   <key>
   -----BEGIN PRIVATE KEY-----
   [Your Client Private Key]
   -----END PRIVATE KEY-----
   </key>

   <tls-auth>
   -----BEGIN OpenVPN Static key V1-----
   [Your Static Key]
   -----END OpenVPN Static key V1-----
   </tls-auth>

   cipher AES-256-GCM
   ncp-ciphers AES-256-GCM
   ```

2. **Start OpenVPN service**:
   ```bash
   sudo systemctl start openvpn@server
   sudo systemctl enable openvpn@server
   ```

3. **Test the connection**:
   Transfer `client-us.ovpn` to your client device and import it into your OpenVPN client software.

## Common Issues

1. **DNS Leaks**:
   Ensure the DNS settings are pushed correctly in the server config and check with [dnsleaktest.com](https://www.dnsleaktest.com).

2. **Connection Timeouts**:
   Verify port 1194 is open in the Azure NSG and the VM firewall.

3. **IP Forwarding**:
   Ensure IP forwarding is enabled and persistent.

4. **Client Can't Access the Internet**:
   Check the NAT rules and ensure the `iptables` MASQUERADE rule is correctly applied.

## Backup Configuration Files

### Zip the Configuration Files

```bash
sudo apt-get install zip
sudo zip -r vpn-config-backup.zip /etc/openvpn /usr/share/easy-rsa/pki
```

### Download the Configuration Files

Use `scp` to download the zipped configuration files to your local machine.

```bash
scp your_username@your_vm_ip:/path/to/vpn-config-backup.zip /local/directory/
```

Or, if you are using a Windows machine with PuTTY, you can use `pscp`:

```bash
pscp your_username@your_vm_ip:/path/to/vpn-config-backup.zip C:\path\to\local\directory\
```

## Updates for IPLEAK Fixes

### Final Configuration Files

#### `server.conf`
```conf
port 1194
proto udp
dev tun

cipher AES-256-GCM
auth SHA256

ca /usr/share/easy-rsa/pki/ca.crt
cert /usr/share/easy-rsa/pki/issued/server.crt
key /usr/share/easy-rsa/pki/private/server.key
dh /usr/share/easy-rsa/pki/dh.pem
tls-auth /etc/openvpn/ta.key 0

server 10.8.0.0 255.255.255.0
server-ipv6 fd00:1234:5678::/64  # Replace with your desired IPv6 subnet

ifconfig-pool-persist ipp.txt

keepalive 10 120
persist-key
persist-tun

status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3

# Push DNS changes to redirect all traffic through the VPN
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"

# Ensure IPv6 traffic is also redirected through the VPN
push "redirect-gateway ipv6"
push "route-ipv6 2000::/3"
push "dhcp-option DNS6 2620:0:ccc::2"
push "dhcp-option DNS6 2620:0:ccd::2"

topology subnet

# Enable IP forwarding and NAT (Network Address Translation)
script-security 2
up /etc/openvpn/scripts/up.sh
down /etc/openvpn/scripts/down.sh
```

#### `client.ovpn`
```conf
client
dev tun
proto udp
remote YOUR_SERVER_IP 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
auth SHA256
verb 3
key-direction 1
keepalive 10 120
data-ciphers AES-256-GCM:AES-128-GCM:CHACHA20-POLY1305

<ca>
-----BEGIN CERTIFICATE-----
[Your CA certificate here]
-----END CERTIFICATE-----
</ca>

<cert>
[Your client certificate here]
-----END CERTIFICATE-----
</cert>

<key>
[Your client private key here]
-----END PRIVATE KEY-----
</key>

<tls-auth>
[Your TLS auth key here]
-----END OpenVPN Static key V1-----
</tls-auth>

cipher AES-256-GCM
ncp-ciphers AES-256-GCM
verb 3

block-outside-dns

tun-ipv6
redirect-gateway ipv6

# Disable IPv6 to prevent leaks
pull-filter ignore "route-ipv6"
```

#### `up.sh`
```sh
#!/bin/sh
# Enable IPv4 forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward
# Enable IPv6 forwarding
echo 1 > /proc/sys/net/ipv6/conf/all/forwarding

# Enable NAT for IPv4
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

# Enable NAT for IPv6
ip6tables -t nat -A POSTROUTING -s 2001:db8:0:123::/64 -o eth0 -j MASQUERADE

# Ensure the firewall is allowing traffic
iptables -A FORWARD -i tun0 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o tun0 -j ACCEPT
ip6tables -A FORWARD -i tun0 -o eth0 -j ACCEPT
ip6tables -A FORWARD -i eth0 -o tun0 -j ACCEPT
```

#### `down.sh`
```sh
#!/bin/sh
# Disable IPv4 forwarding
echo 0 > /proc/sys/net/ipv4/ip_forward
# Disable IPv6 forwarding
echo 0 > /proc/sys/net/ipv6/conf/all/forwarding

# Remove NAT for IPv4
iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

# Remove NAT for IPv6
ip6tables -t nat -D POSTROUTING -s 2001:db8:0:123::/64 -o eth0 -j MASQUERADE

# Remove forwarding rules
iptables -D FORWARD -i tun0 -o eth0 -j ACCEPT
iptables -D FORWARD -i eth0 -o tun0 -j ACCEPT
ip6tables -D FORWARD -i tun0 -o eth0 -j ACCEPT
ip6tables -D FORWARD -i eth0 -o tun0 -j ACCEPT
```

### Commands to Set Up and Download Configuration Files

1. **Create Directory for Scripts:**
    ```sh
    sudo mkdir -p /etc/openvpn/scripts
    ```

2. **Create and Set Permissions for `up.sh` Script:**
    ```sh
    sudo nano /etc/openvpn/scripts/up.sh
    sudo chmod +x /etc/openvpn/scripts/up.sh
    ```

3. **Create and Set Permissions for `down.sh` Script:**
    ```sh
    sudo nano /etc/openvpn/scripts/down.sh
    sudo chmod +x /etc/openvpn/scripts/down.sh
    ```

4. **Restart OpenVPN Service:**
    ```sh
    sudo systemctl restart openvpn@server
    ```

5. **Enable Firewall Rules (if `ufw` is being used):**
    ```sh
    sudo ufw allow 1194/udp
    sudo ufw enable
    ```

### Commands to Download Configuration Files for Future Use

You can use the following commands to download the configuration files to your local machine for backup purposes.

1. **Download `server.conf`:**
    ```sh
    scp user@your_server_ip:/etc/openvpn/server.conf /path/to/local/directory/server.conf
    ```

2. **Download `client.ovpn`:**
    ```sh
    scp user@your_server_ip:/path/to/your/client.ovpn /path/to/local/directory/client.ovpn
    ```

3. **Download `up.sh`:**
    ```sh
    scp user@your_server_ip:/etc/openvpn/scripts/up.sh /path/to/local/directory/up.sh
    ```

4. **Download `down.sh`:**
    ```sh
    scp user@your_server_ip:/etc/openvpn/scripts/down.sh /path/to/local/directory/down.sh
    ```

Replace `user` with your username, `your_server_ip` with the IP address of your server, and `/path/to/local/directory` with the directory on your local machine where you want to save the files.

With these configurations and commands, you should have a fully functioning OpenVPN setup with IPv4, and you'll be able to download and back up your configuration files for future use.

## License

[MIT](LICENSE)
```

This README file includes all necessary steps, configuration details, common issues, and backup instructions for setting up OpenVPN on an Azure VM. You can copy and paste this content into a README.md file for your GitHub repository. If you need any more customizations, feel free to ask!
