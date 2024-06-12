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

## License

[MIT](LICENSE)
```

This README file includes all necessary steps, configuration details, common issues, and backup instructions for setting up OpenVPN on an Azure VM. You can copy and paste this content into a README.md file for your GitHub repository. If you need any more customizations, feel free to ask!
