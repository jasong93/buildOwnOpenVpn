# buildOwnOpenVpn

### 此文档目的是指导如何创建自己的open vpn

搭建OpenVPN服务器需要以下几个步骤：

1. **安装OpenVPN和Easy-RSA**
2. **配置Easy-RSA生成证书和密钥**
3. **配置OpenVPN服务器**
4. **配置防火墙和路由**
5. **生成客户端配置文件**

下面是具体步骤，以Ubuntu为例：

## 1. 安装OpenVPN和Easy-RSA

首先，更新软件包列表并安装OpenVPN和Easy-RSA：

```sh
sudo apt-get update
sudo apt-get install openvpn easy-rsa -y
```

## 2. 配置Easy-RSA生成证书和密钥

创建一个目录来存放Easy-RSA相关文件：

```sh
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
```

编辑`vars`文件，配置一些基本信息：

```sh
nano vars
```

在`vars`文件中，修改以下内容（根据实际情况）：

```sh
set_var EASYRSA_REQ_COUNTRY    "US"
set_var EASYRSA_REQ_PROVINCE   "California"
set_var EASYRSA_REQ_CITY       "San Francisco"
set_var EASYRSA_REQ_ORG        "MyOrg"
set_var EASYRSA_REQ_EMAIL      "email@example.com"
set_var EASYRSA_REQ_OU         "MyOrgUnit"
```

初始化PKI并生成CA证书和密钥：

```sh
./easyrsa init-pki
./easyrsa build-ca
```

生成服务器证书和密钥：

```sh
./easyrsa gen-req server nopass
./easyrsa sign-req server server
```

生成Diffie-Hellman参数：

```sh
./easyrsa gen-dh
```

生成TLS密钥：

```sh
openvpn --genkey --secret ta.key
```

## 3. 配置OpenVPN服务器

复制生成的证书和密钥到OpenVPN目录：

```sh
sudo cp pki/ca.crt pki/private/server.key pki/issued/server.crt pki/dh.pem /etc/openvpn
sudo cp ta.key /etc/openvpn
```

创建并编辑OpenVPN服务器配置文件：

```sh
sudo nano /etc/openvpn/server.conf
```

填入以下内容：

```conf
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
tls-auth ta.key 0
cipher AES-256-CBC
auth SHA256
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
keepalive 10 120
persist-key
persist-tun
status openvpn-status.log
verb 3
```

## 4. 配置防火墙和路由

启用IP转发：

```sh
sudo nano /etc/sysctl.conf
```

取消以下行的注释：

```conf
net.ipv4.ip_forward=1
```

使更改生效：

```sh
sudo sysctl -p
```

配置防火墙规则：

```sh
sudo ufw allow 1194/udp
sudo ufw allow OpenSSH
sudo ufw disable
sudo ufw enable
```

添加NAT规则：

```sh
sudo nano /etc/ufw/before.rules
```

在`*filter`之前添加以下内容：

```conf
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
```

## 5. 生成客户端配置文件

生成客户端证书和密钥：

```sh
cd ~/openvpn-ca
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
```

创建客户端配置文件：

```sh
nano client1.ovpn
```

填入以下内容：

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
tls-auth ta.key 1
cipher AES-256-CBC
auth SHA256
verb 3

<ca>
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
</ca>

<cert>
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
</key>

<tls-auth>
-----BEGIN OpenVPN Static key V1-----
...
-----END OpenVPN Static key V1-----
</tls-auth>
```

将`ca.crt`、`client1.crt`、`client1.key`和`ta.key`的内容分别填入相应的部分。

## 6. 启动OpenVPN服务

启动并启用OpenVPN服务：

```sh
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```

## 7. 连接客户端

将生成的`client1.ovpn`文件复制到客户端设备上，并使用OpenVPN客户端软件导入并连接。

这样，OpenVPN服务器就搭建完成了。如果有其他客户端需要连接，重复第5步生成新的客户端配置文件即可。
