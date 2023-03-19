# ubutu-openvpn-tutorial
在本教程中，我们将学习如何在Ubuntu(20.04 lts)服务器上使用OpenVPN搭建VPN服务器。请按照以下步骤操作：

# 第1步：更新系统
在开始之前，请确保系统已更新到最新版本。
```
sudo apt update
sudo apt upgrade
```
# 第2步：安装OpenVPN和Easy-RSA
安装OpenVPN和用于创建证书的Easy-RSA。
```
sudo apt install openvpn easy-rsa
```
# 第3步：初始化PKI (Public Key Infrastructure)

返回到您的主目录并创建一个新的easy-rsa目录：

```
cd ~
mkdir easy-rsa
```
将示例Easy-RSA配置文件复制到该目录：

```
cp -R /usr/share/easy-rsa/* ~/easy-rsa/
```
进入easy-rsa目录：
```
cd ~/easy-rsa
```
初始化PKI：
```
./easyrsa init-pki
```
# 第4步：创建CA并生成服务器和客户端证书

建立CA：
```
./easyrsa build-ca
```
按提示操作，为CA创建一个密码并输入相关信息。

接下来，生成服务器证书和密钥：

```
./easyrsa build-server-full server nopass
```
为客户端生成证书和密钥：

```
./easyrsa build-client-full client1 nopass
```
# 第5步：生成Diffie-Hellman密钥交换文件

执行以下命令生成Diffie-Hellman密钥交换文件：

```
./easyrsa gen-dh
```

# 第6步：配置OpenVPN服务器

将证书和密钥文件复制到OpenVPN目录：

```
sudo cp ~/easy-rsa/pki/{ca.crt,issued/server.crt,private/server.key,dh.pem} /etc/openvpn/
```
创建一个新的OpenVPN配置文件：

```
sudo nano /etc/openvpn/server.conf
```
将以下内容粘贴到文件中，并保存退出：
```
vbnet
Copy code
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"
keepalive 10 120
cipher AES-256-CBC
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
verb 3
```

# 第7步：启用IPv4转发

编辑sysctl.conf文件：

```
sudo nano /etc/sysctl.conf
```
取消以下行的注释：

```
net.ipv4.ip_forward=1
```
保存并退出。然后运行以下命令使更改生效：
如果/etc/sysctl.conf文件中没有net.ipv4.ip_forward=1这一行，您可以直接在文件末尾添加它。
```
sudo sysctl -p
```

# 第8步：配置防火墙

首先，安装ufw（如果尚未安装）：

```
sudo apt install ufw
```
配置防火墙规则以允许OpenVPN流量：

```
sudo nano /etc/default/ufw
```
找到DEFAULT_FORWARD_POLICY，将其值更改为ACCEPT：

```
DEFAULT_FORWARD_POLICY="ACCEPT"
```
保存并退出。

现在，创建一个新的UFW规则文件：

```
sudo nano /etc/ufw/before.rules
```
在文件顶部添加以下内容：

```
# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
# Allow traffic from OpenVPN client to eth0 (change to your network interface)
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES
```
将eth0更改为您的服务器网络接口名称。

接下来，允许OpenVPN和SSH流量：
```
sudo ufw allow 1194/udp
sudo ufw allow ssh
```
启用防火墙：

```
sudo ufw enable
```

### 如何查找服务器网络接口名称
#### 使用ip命令：

在终端中输入以下命令：

```
ip addr show
```
您将看到类似于以下内容的输出：
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:7e:37:4b brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global eth0
       valid_lft forever preferred_lft forever
```
在这个例子中，eth0是网络接口名称。根据您的系统，它可能是ens160，enp0s3或其他类似名称。


# 第9步：启动OpenVPN服务器

现在启动OpenVPN服务器并设置为开机自启：

```
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```
您可以通过以下命令检查OpenVPN服务器的状态：

```
sudo systemctl status openvpn@server
```
# 第10步：创建客户端配置文件

首先，在您的主目录中创建一个名为client-configs的文件夹：

```
mkdir -p ~/client-configs
```
接下来，进入client-configs目录并创建一个名为client.ovpn的文件：

```
cd ~/client-configs
nano client.ovpn
```
将以下内容粘贴到文件中：
```
vbnet
Copy code
client
dev tun
proto udp
remote your_server_ip 1194
resolv-retry infinite
nobind
persist-key
persist-tun
mute-replay-warnings
cipher AES-256-CBC
verb 3
```
用实际的服务器IP地址替换your_server_ip。

将以下内容替换为行ca ca.crt：
ca.crt文件内容确认
```
cat ~/easy-rsa/pki/ca.crt
```
```
<ca>
-----BEGIN CERTIFICATE-----
(将CA证书的内容粘贴在这里)
-----END CERTIFICATE-----
</ca>
```
在这里粘贴ca.crt文件的内容。确保包括-----BEGIN CERTIFICATE-----和-----END CERTIFICATE-----。
将以下内容替换为行cert client1.crt：
client1.crt文件内容确认
```
cat ~/easy-rsa/pki/issued/client1.crt
```
```
<cert>
-----BEGIN CERTIFICATE-----
(将客户端证书的内容粘贴在这里)
-----END CERTIFICATE-----
</cert>
```
在这里粘贴client1.crt文件的内容。确保包括-----BEGIN CERTIFICATE-----和-----END CERTIFICATE-----。

将以下内容替换为行key client1.key：
client1.key文件内容确认
```
cat ~/easy-rsa/pki/private/client1.key
```
```
<key>
-----BEGIN PRIVATE KEY-----
(将客户端密钥的内容粘贴在这里)
-----END PRIVATE KEY-----
</key>
```
在这里粘贴client1.key文件的内容。确保包括-----BEGIN PRIVATE KEY-----和-----END PRIVATE KEY-----。

保存并退出。

现在，您的.ovpn文件应该包含所有必要的证书和密钥。将此文件传输到客户端设备并导入到支持OpenVPN的客户端应用程序中。这将允许客户端通过VPN服务器连接。无需单独导入其他证书和密钥文件。

