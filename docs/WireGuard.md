# 新加坡单节点：从一部手机开始

如果目标是让一部手机通过自己控制的出口上网，可以先完成这一条路线。这里的规则只用于自己信任的设备；如向外部分发，需另行隔离云内网和元数据服务，并给每位使用者独立授权。安装 WireGuard 的每台终端需要自己的密钥。双节点 Hysteria 2 使用不同的客户端配置，两个方案可以独立部署。

## 准备一台新实例

按[主指南](实操指南.md)核对 AWS 账户计划、创建 Ubuntu Lightsail 实例并绑定同区域的静态 IPv4。以下操作面向新建、由自己独占的 Ubuntu 实例，使用 IPv4 转发。把公网 SSH 限制在自己的管理来源，确认控制台的 SSH 恢复通道可用。在 Lightsail 网络防火墙为客户端开放 UDP 51820，如系统启用了防火墙，也要放行相同端口。新加坡原案例后来换过端口，读者可以从标准端口开始。新部署用 51820 时可与主指南的 8443 并存。

在服务器的终端执行以下程序和状态检查。如果已有 wg0、同名配置、同一端口的服务或与示例重叠的网段，先选其他接口名、端口或隧道网段，再相应调整全部步骤。

```bash
sudo -i
apt-get update && apt-get install -y wireguard iptables
ip -4 route show default
ip -4 route
ip link show type wireguard
ss -lun
ls -l /etc/wireguard
```

## 在两端准备密钥

在手机或电脑的 WireGuard 中选择新建空白隧道，生成密钥，把**客户端公钥**复制给服务器管理员。客户端私钥留在该设备。在服务器的 root 终端执行以下命令。它们为新接口生成单独的服务器密钥，并打印需要交给客户端的公钥。

```bash
install -d -m 700 /etc/wireguard
cd /etc/wireguard
umask 077
test ! -e wg0.key && wg genkey > wg0.key
wg pubkey < wg0.key
```

## 配置 IPv4 转发

以下示例使用 10.77.0.0/24。记录 `sysctl -n net.ipv4.ip_forward` 的原值。在干净实例上创建 `/etc/sysctl.d/70-personal-wg.conf`，内容为 `net.ipv4.ip_forward=1`。运行 `sysctl -p /etc/sysctl.d/70-personal-wg.conf`。用 `ip -4 route show default` 查公网出口网卡名，替换下方所有 `WAN_IF`。

用 `test ! -e /etc/wireguard/wg0.conf && install -m 600 /dev/null /etc/wireguard/wg0.conf` 创建权限仅限 root 的新配置文件。如果已有配置，请先停止本步骤并保留原文件。用 root 编辑器在服务器本机把 `SERVER_PRIVATE_KEY` 替换为 wg0.key 中的值。将 `DEVICE_PUBLIC_KEY` 替换为第一台设备的公钥。示例在接口启动时添加对应转发与地址转换规则，在关闭时精确撤销。已有防火墙规则集的主机应在原规则体系中配置。

```ini
[Interface]
Address = 10.77.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY
PostUp = iptables -A FORWARD -i %i -o WAN_IF -s 10.77.0.0/24 -j ACCEPT
PostUp = iptables -A FORWARD -i WAN_IF -o %i -d 10.77.0.0/24 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -s 10.77.0.0/24 -o WAN_IF -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -o WAN_IF -s 10.77.0.0/24 -j ACCEPT
PostDown = iptables -D FORWARD -i WAN_IF -o %i -d 10.77.0.0/24 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.77.0.0/24 -o WAN_IF -j MASQUERADE

[Peer]
PublicKey = DEVICE_PUBLIC_KEY
AllowedIPs = 10.77.0.2/32
```

运行 `systemctl start wg-quick@wg0`，再执行 `wg show wg0`。应看到刚配置的监听端口和客户端公钥。

## 在备用设备上验证

把下表填入刚生成密钥的客户端隧道。使用配置文件导入时应在设备私有目录内制作，每台设备单独一份。路由器提供的私网 DNS 从外面不一定能访问，示例使用公共 DNS。

| 界面中的项目 | 第一个客户端的值 |
| --- | --- |
| 本机地址 / Address | 10.77.0.2/32 |
| DNS | 1.1.1.1 |
| 对端公钥 / Peer public key | 上一步取得的服务器公钥 |
| 端点 / Endpoint | 自己的静态 IPv4 加上 `:51820` |
| 路由 / Allowed IPs | 0.0.0.0/0 |
| 保活 / Persistent keepalive | 25 秒，适用于位于 NAT 后且需保持映射的设备 |

启用备用设备的隧道，打开 [AWS 出口查询](https://checkip.amazonaws.com)。结果应等于控制台中的新加坡静态 IPv4。再到服务器执行 `wg show wg0`，确认最近握手时间及接收、发送字节都已更新。Wi-Fi 和蜂窝数据可各测一次。示例只接管 IPv4；需要 IPv6 也经过隧道时，须另行配置 IPv6 转发、路由和相应测试。

确认有效后，可用 `systemctl enable wg-quick@wg0` 设置该云实例启动时恢复服务。第二台设备使用新密钥及 10.77.0.3/32，在服务端增加另一段 Peer 并用 `wg syncconf wg0 <(wg-quick strip wg0)` 在 root 的 Bash 中应用，保留已有会话。若撤销本次实验，先用 `systemctl disable --now wg-quick@wg0` 撤掉本接口的规则，关闭本次新增的 UDP 入站规则。仅当本机其他服务均无需 IPv4 转发时才恢复原转发值。若要结束云资源计费，再按主指南处理实例及静态 IP。

[WireGuard 官方快速入门](https://www.wireguard.com/quickstart/) · [Ubuntu 的默认网关配置说明](https://ubuntu.com/server/docs/how-to/wireguard-vpn/vpn-as-the-default-gateway/)
