# 1.Tailscale-Derp
如何自部署Tailscale中继Derp节点

### 安装tailscale
- 安装`curl -fsSL https://tailscale.com/install.sh | sh`
- 启动`tailscale up`
## Golang
1. `https://go.dev/dl/`下载golang,
2. 解压到 `/usr/local`
```shell
sudo rm -rf /usr/local/go  # 删除旧版本（如有）
sudo tar -C /usr/local -xzf go1.22.4.linux-amd64.tar.gz  # 替换成你下载的版本
```
3. 设置环境变量:编辑 `~/.bashrc`（或 `~/.zshrc`、`~/.profile`）：
```shell
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go  # 可选：设置Go工作目录
export PATH=$PATH:$GOPATH/bin
```
4. 使配置生效：
```shell
source ~/.bashrc
```
### 安装DERP服务
1. `go install tailscale.com/cmd/derper@latest` 安装DERP
2. 安装完成后，DERP 服务器的可执行文件将位于 `~/go/bin/derper`，为了方便使用，可以将其移动到` /usr/bin/ `目录：`cp ~/go/bin/derper /usr/bin/`
### 生成 SSL 证书
1. 由于 Tailscale 使用 SSL 加密，DERP 服务器需要一个证书。我们将生成自签名证书（有效期 10 年），并包含 VPS 的 IP 地址。（如果你有域名，可以不用自签名证书，稍后改为letsencrypt模式）
例如你的VPS的IP地址为：154.xxx.xxx.xxx
```bash
mkdir /home/user/derp
cd /home/user/derp
DERP_IP="154.xxx.xxx.xxx"  # 替换为你的 VPS 的 IP 地址
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes -keyout ${DERP_IP}.key -out ${DERP_IP}.crt -subj "/CN=${DERP_IP}" -addext "subjectAltName=IP:${DERP_IP}"
```
此命令将在当前目录（_/home/user/derp_）下生成两个文件：`${DERP_IP}.key` 和 `${DERP_IP}.crt`。这将在后续步骤中使用。
### 启动DERP服务
例如HTTPS服务的**TCP**端口为`10001`，STUN服务的**UDP**端口为`10002`，IP为`154.xxx.xxx.xxx`，自签名证书目录为`/home/user/derp`，防火墙打开相应端口后，执行：
```bash
derper -a :10001 -http-port -1 -stun -stun-port 10002 -hostname 154.xxx.xxx.xxx -certmode manual -certdir /home/user/derp -verify-client
```
您可以通过添加额外的命令参数来自定义 DERP 服务器的配置。以下是常用的参数说明及其用途：
-a string  
指定服务器的 HTTP/HTTPS 监听地址。格式为 “:端口”、”ip:端口” 或 IPv6 地址 “[ip]:端口”。如果未指定 IP，默认监听所有接口。如果端口为 443 和/或 -certmode 为 manual，则提供 HTTPS 服务；否则，提供 HTTP 服务。默认值为 “:443″。
-accept-connection-burst int  
设置接受新连接的突发限制。默认值为 9223372036854775807，表示没有限制。
-accept-connection-limit float  
设置接受新连接的速率限制。默认值为 +Inf，即无限制。
-bootstrap-dns-names string  
可选参数，指定以逗号分隔的主机名列表，这些主机名会在 /bootstrap-dns 下可用。
-c string  
配置文件的路径。
-certdir string  
存储证书的目录。如果监听地址使用端口 443，默认目录为 /root/.cache/tailscale/derper-certs。
-certmode string  
获取证书的模式。可选的模式为 manual 或 letsencrypt。默认值为 letsencrypt。
-derp  
设置是否运行 DERP 服务器。默认值为 true。将其设置为 false，仅运行 bootstrap DNS 功能。
-dev  
以本地开发模式运行，覆盖 -a 参数设置。
-hostname string  
指定 Let’s Encrypt 主机名。如果端口为 443，默认值为 derp.tailscale.com。
-http-port int  
设置 HTTP 服务的端口。如果设置为 -1，禁用 HTTP 服务。监听器绑定到 -a 标志指定的地址。
-mesh-psk-file string  
如果指定，该路径包含 mesh 预共享密钥文件，文件内容应为十六进制字符串。空白表示不使用预共享密钥。
-mesh-with string  
可选参数，以逗号分隔的主机名列表，指定与哪些主机进行 mesh 网络连接。服务器自身的主机名也可以在列表中。
-stun  
设置是否运行 STUN 服务器。默认值为 true，它会绑定到与 -a 标志指定的相同 IP。
-stun-port int  
设置运行 STUN 服务的 UDP 端口。默认值为 3478。
-unpublished-bootstrap-dns-names string  
可选参数，以逗号分隔的主机名列表，这些主机名会在 /bootstrap-dns 下可用，但不会公开。
-verify-clients  
通过本地 tailscaled 实例验证客户端的身份。
### 在Tailscale 控制台添加DERP服务
访问官网的[Access controls](https://login.tailscale.com/admin/acls/file)页面，增加以下字段：
```json
"randomizeClientPort": true, //让Tailscale客户端在每次连接时使用随机 UDP 端口
"derpMap": {
    "OmitDefaultRegions": false, // 是否忽略官方DERP服务器。如果不特别自信，最好不要设置为true，因为在自己搭建的 DERP 服务器出现异常时，官方 DERP 服务器可以作为备选，确保连接正常。
    "Regions": {
         "900": { // DERP服务器的区域ID，必须从不小于900开始以免与官方冲突。
            "RegionID": 900, // 唯一标识区域的ID。这个值需要与您所选择的区域一致。
            "RegionCode": "自定义区域代码", // 区域代码，可以是您选择的区域的代号或标识符。
            "RegionName": "自定义区域名称", // 区域名称，可以是描述该区域的名称，便于理解。
            "Nodes": [
                {
                    "Name": "自定义节点名称", // 节点名称，表示这个DERP服务器的名称。
                    "RegionID": 900, // 该节点所属的区域ID，应该与父级区域ID一致。
                    "HostName": "154.xxx.xxx.xxx", // 指定该DERP服务器使用的主机名。通常是自签名的域名，用于TLS证书验证。如果选择了自签名证书，推荐使用域名；如果使用的是普通IP地址，也可以填IP地址。
                    "DERPPort": 10001, // 该DERP服务器使用的HTTPS端口。需要与实际运行的DERP服务的端口一致，默认值为443。
                    "IPv4": "154.xxx.xxx.xxx", // 指定主机名对应的IPv4地址。如果使用域名，必须提供IPv4地址，否则DNS解析可能无法正常工作。
                // "IPv6": "x.x.x.x", // 可选，指定主机名对应的IPv6地址。如果有IPv6支持，可以填写此项。
                    "InsecureForTests": true, // 当HostName没有使用域名，而是使用IP时设置为true，这会降低连接的安全性。
                    "STUNPort": 10002 // 用于STUN协议的端口，默认值是3478。STUN用于NAT穿透，如果需要修改端口，确保与DERP服务的配置一致。
                }
            ]
        }
    }
},
```
Save后，可以运行以下命令来检查 DERP 服务器的状态：
```bash
tailscale netcheck
```
该命令将会检测你的网络配置，并验证是否能够成功连接到 DERP 服务器。它会提供延迟信息、STUN 连接状态以及连接到 Tailscale 网络的其他详细信息。确保输出中没有错误，并且 DERP 服务器的连接状况良好。
### 创建系统服务
为了让DERP服务在系统启动时自动启动，我们将其配置为系统服务，创建并编辑文件`/etc/systemd/system/derp.service`
```apacheconf
[Unit]
Description=Tailscale DERP
After=network.target
Wants=network.target

[Service]
User=root
Restart=always
ExecStart=/root/go/bin/derper -a :10001 -http-port -1 -stun -stun-port 10002 -hostname 100.xxx.xxx.xxx -certmode manual -certdir /home/user/derp -verify-clients
RestartSec=5
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
```




