# OPEN VPN Server
[Source link](https://www.webhi.com/how-to/how-to-install-openvpn-server-on-ubuntu/)
- Install openvpn server and easy-rsa
```bash
sudo apt install openvpn easy-rsa
```
- Make a directory for the certificates using `make-cadir` command
```bash
make-cadir ~/openvpn-ca && cd ~/openvpn-ca
```
- Edit the `vars` file
```bash
nano vars
```
- Put the following configs, *this is an example*
```bash
set_var EASYRSA_REQ_COUNTRY    "IQ"
set_var EASYRSA_REQ_PROVINCE   "Nineveh"
set_var EASYRSA_REQ_CITY       "Mosul"
set_var EASYRSA_REQ_ORG        "QAF Lab"
set_var EASYRSA_REQ_EMAIL      "researchlab@qaflab.net"
set_var EASYRSA_REQ_OU         "Research Lab"
```
- Run the following commands
```bash
./easyrsa init-pki
./easyrsa build-ca
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey --secret pki/ta.key
```
- Copy the certificate files to `/etc/openvpn/` directory
```bash
sudo cp pki/{ca.crt,dh.pem,ta.key} /etc/openvpn
sudo cp pki/issued/server.crt /etc/openvpn
sudo cp pki/private/server.key /etc/openvpn
```
- Edit `/etc/openvpn/server.conf` file
```bash
sudo nano /etc/openvpn/server.conf
```
- Edit the configurations to match these settings, **if there are other configurations leave them as they are**
```bash
port 1194
proto udp6
dev tun
user nobody
group nogroup
persist-key
persist-tun
keepalive 10 120
topology subnet
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /etc/openvpn/ipp.txt
tun-ipv6
push tun-ipv6
push "route-ipv6 2000::/3"
push "redirect-gateway ipv6"
dh none
ecdh-curve prime256v1
tls-crypt ta.key
crl-verify crl.pem
ca ca.crt
cert server.crt
key server.key
auth SHA256
cipher AES-128-GCM
ncp-ciphers AES-128-GCM
tls-server
tls-version-min 1.2
tls-cipher TLS-ECDHE-ECDSA-WITH-AES-128-GCM-SHA256
client-config-dir /etc/openvpn/ccd
status /var/log/openvpn/status.log
verb 3
explicit-exit-notify
duplicate-cn
push "redirect-gateway def1"
push "dhcp-option DNS 10.8.0.1"
```
- Enable IP forwarding
```bash
sudo nano /etc/sysctl.conf
```
- Uncomment the following line:
```bash
net.ipv4.ip_forward=1
```
- Apply the change
```bash
sudo sysctl -p
```
- Start and enable OPEN VPN server
```bash
sudo systemctl start openvpn@server 
sudo systemctl enable openvpn@server
```
## Client Setup
- The following commands are also executed on the server
- Enter the `openvpn-ca` directory
```bash
cd ~/openvpn-ca
```
- Make certificates to be used for clients later
```bash
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
cp pki/private/client1.key /etc/openvpn/client/
cp pki/issued/client1.crt /etc/openvpn/client/
cp pki/{ca.crt,ta.key} /etc/openvpn/client/
```
- Create and edit client.conf file
```bash
nano client.conf
```
- Puth the following configurations, **replace the SERVER_IP_ADDRESS with the IP address of the server**
```bash
client
dev tun
proto udp
remote SERVER_IP_ADDRESS 1194
resolv-retry infinite
nobind
user nobody
group nobody
persist-key
persist-tun
key-direction 1
remote-cert-tls server
tls-auth ta.key 1
cipher AES-256-CBC
verb 3
```
- Make a config generator script to be used to generate clients
```bash
nano config_gen.sh
```
- Write the following configuration
```bash
#!/bin/bash
# First argument: Client identifier
KEY_DIR=/etc/openvpn/client
OUTPUT_DIR=.
BASE_CONFIG=client.conf
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
- Change the file to be executable
```bash
chmod +x config_gen.sh
```
- You can use it now to make new clients
```bash
./config_gen.sh client1
```
- After generating a new client, a file named `client1.ovpn` will be generated, use this file to connect to the server from any client
- If you want to give a static IP for a certain client, create a file in `/etc/openvpn/ccd`
```bash
nano /etc/openvpn/ccd/client1
```
- Write the following configurations
```bash
#ifconfig-push clientIP Netmask
ifconfig-push 10.8.0.200 255.255.255.0
```
replace `10.8.0.200` with the IP address you want to give to this client
