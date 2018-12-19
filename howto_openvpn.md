# OpenVPN
Be root or use sudo for all commands, and update your system. Following, install openvpn & easy-rsa

```
apt install openvpn easy-rsa
```
Create an easy-rsa directory

```
mkdir /etc/openvpn/easy-rsa/
```

Copy the files from easy-rsa into the openvpn folder

```
cp -r /usr/share/doc/openvpn/examples/easy-rsa/2.0/* /etc/openvpn/easy-rsa/
```

## OpenVPN conf.

Optionally you can edit the certificates settings

```
nano /etc/openvpn/easy-rsa/var

```

Refers to: <https://blog.ssdnodes.com/blog/tutorial-installing-openvpn-on-ubuntu-16-04/>

Note: If the file above is edited, change this line to point to the good top level folder

```
export EASY_RSA="/etc/openvpn/easy-rsa"

```

## Server public/private keys

Load vars

```
cd /etc/openvpn/easy-rsa/
source vars
./clean-all
```

Pointing to the good version

```
ln -s openssl-1.0.0.cnf openssl.cnf
```
Generate CA
```
./build-ca OpenVPN
```

Generate Server certificates
```
./build-key-server server
```
Generate Client certificates
```
./build-key-pass
```
Generate Diffie-Hellman key
```
./build-dh
```

All generated certificates and keys are in : ```/etc/openvpn/easy-rsa/keys```

# Server Configuration

Create the configuration file ```nano /etc/openvpn/openvpn.conf```

And paste

```
dev tun
proto udp
port 1194
ca /etc/openvpn/easy-rsa/keys/ca.crt
cert /etc/openvpn/easy-rsa/keys/server.crt
key /etc/openvpn/easy-rsa/keys/server.key
dh /etc/openvpn/easy-rsa/keys/dh2048.pem
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.4.4"
push "dhcp-option DNS 8.8.8.8"
log-append /var/log/openvpn
persist-key
persist-tun
user nobody
group nogroup
status /var/log/openvpn-status.log
verb 3
client-to-client
comp-lzo
```

Be careful with the paths. The relative path is ```/etc/openvpn/```. You can also use absolutes paths.

Check also if the certificates names are good.

## Start Server
```
openvpn --config /etc/openvpn/openvpn.conf
```

If it doesn't works, try stopping the service using ```service openvpn stop```

NAT the VPN client traffic to the Internet

Change the IP address mask according to your info of tun0 result while running the "ifconfig" command.

```
nano /etc/openvpn/natclientraffictotheinternet.sh
```

```
echo "1" > /proc/sys/net/ipv4/ip_forward
iptables -A INPUT -i tun0 -j ACCEPT
iptables -A FORWARD -i tun0 -j ACCEPT
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```
```
chmod +x /etc/openvpn/natclientraffictotheinternet.sh
```

Use this script to clean (disable) the NAT configuration

```
nano /etc/openvpn/stopnatclientraffictotheinternet.sh
```
```
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
```
```
chmod +x /etc/openvpn/stopnatclientraffictotheinternet.sh
```

Start the NAT

```
/etc/openvpn/natclientraffictotheinternet.sh
```
Stop the NAT
```
/etc/openvpn/stopnatclientraffictotheinternet.sh
```

# Client Configuration

First you need to get the following certificates from the server by using scp

``` scp user1@192.168.x.xx:/etc/openvpn/easy-rsa/keysclient* /etc/openvpn/ ```

``` scp user1@192.168.x.xx:/etc/openvpn/easy-rsa/ca.crt /etc/openvpn/ ```

``` nano client.ovpn ```

```
dev tun
client
proto udp
remote IP_ROUTEUR_ADSL 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
comp-lzo
verb 3
```

Again be careful with the paths: ```/etc/openvpn/```.

Check also if the certificates names are good.

Start the Client

Launch manually ```openvpn --config /etc/openvpn/client.ovpn```

If it doesn't work, try stopping the service using ```service openvpn stop```
