# Microsoft Azure 搭建 Hysteria 2 实操指南

本文说明如何在 **Microsoft Azure** 上创建东京（`Japan East`）与新加坡（`Southeast Asia`）节点，并复用本仓库的 [Hysteria 2 服务端模板](../examples/hysteria-server.yaml) 和 [FlClash 双节点模板](../examples/flclash.yaml)。

> [!IMPORTANT]
> 开始前请先阅读 [云厂商方案对比](云厂商方案对比.md)，明确 Azure 的两个核心费用边界：
> 1. 新户 **$200 抵用金仅限前 30 天**，到期清零；之后想使用 12 个月免费机型必须升级为 Pay-As-You-Go。
> 2. 12 个月免费额度仅够 **1 台** 符合条件的机型（`B1s` / `B2ats v2`）24/7 运行；**静态 Standard IPv4 不在免费额度内**（约 $3.65/月），且亚太区（Zone 2）超出免费流量后按约 **$0.12/GB** 计费。

## 1. 准备 Azure 账户与确认计费模式

1. 进入 [Azure 免费账户注册页](https://azure.microsoft.com/free/)，使用 Microsoft 账号完成身份与信用卡验证，并开启账号的双重验证（MFA）。
2. 登录 [Azure Portal](https://portal.azure.com/)，在 **Cost Management + Billing** 确认当前订阅类型：
   - **前 30 天免费试用（Free Trial）**：自带 $200 抵用金和默认消费上限（Spending Limit），额度用尽或满 30 天会自动暂停服务，不会悄悄扣卡。此阶段可以放心开东京 + 新加坡两台机器。
   - **第 31 天起（12 个月免费阶段）**：升级为 Pay-As-You-Go 后，消费上限解除。若追求零成本或极低成本，建议**只保留 1 台** 免费机型（`Standard_B1s` 或 `Standard_B2ats_v2`），并立即在 **Cost Management → Budgets** 设置月度预算告警（如 $5 和 $10）。

## 2. 创建虚拟机（VM）

先以新加坡为例，部署验证通过后，如需双节点再按相同步骤创建东京节点。

1. 在 Azure Portal 搜索并进入 **Virtual machines → Create → Azure virtual machine**。
2. **Basics（基础配置）**：
   - **Resource group**：新建独立资源组（例如 `rg-proxy-sg`），后续一键删除最干净，避免遗留磁盘或公网 IP 继续扣费。
   - **Region**：新加坡选 **Southeast Asia**；东京选 **Japan East**。
   - **Image**：选择 **Ubuntu Server 24.04 LTS - x64 Gen2**。
   - **Size**：
     - 走 12 个月免费额度：选 **`Standard_B1s`**（1 vCPU, 1 GiB RAM）或 **`Standard_B2ats_v2`**（AMD 突发机型，性能更好，以创建页标注 "Free services eligible" 为准）。
     - 30 天 $200 体验期：也可直接选 `B1s` / `B2ats_v2`，足够个人 Hysteria 2 使用。
   - **Authentication type**：选 **SSH public key**，生成或使用自己的密钥对，妥善下载 `.pem` 私钥。
3. **Disks（磁盘）**：
   - 12 个月免费额度通常包含 **64 GiB Standard SSD（E6）** 或 Standard HDD，**不要选默认的 Premium SSD**，否则会产生额外磁盘费。将 OS disk type 改为 **Standard SSD**（或确认符合免费标注）。
4. **Networking（网络与公网 IP 避坑）**：
   - 确认自动创建一个 **Public IP**。
   - 注意：**Standard SKU 静态公网 IPv4** 在 Pay-As-You-Go 下约 **$0.005/小时（约 $3.65/月）**。如果你在 30 天试用期内，费用会从 $200 抵用金扣除；如果后续进入 12 个月免费期，需留意账单中的 Public IP 项。
   - **NIC network security group**：保持默认 Basic 或创建独立 NSG，先放行 SSH（TCP 22，建议在 Source 限制为自己的公网 IP）。
5. 核对下方预估价格后点击 **Review + create → Create**。

## 3. 在网络安全组（NSG）放行 UDP 端口

Azure 的防火墙规则配置在虚拟机关联的 **Network Security Group（NSG）** 中。

| 地区 | Azure Region | 建议监听端口 | 证书名称（SNI） |
| --- | --- | --- | --- |
| 东京 | `Japan East` | UDP 443 | `jp.vpn.invalid` |
| 新加坡 | `Southeast Asia` | UDP 8443 | `sg.vpn.invalid` |

操作步骤：

1. 进入已创建的虚拟机页面，左侧点击 **Networking → Network settings**。
2. 点击 **Create port rule → Inbound port rule**，填写：
   - **Source**：`Any`（若仅在固定宽带使用，可改为 `IP Addresses` 填入自家公网 IP）。
   - **Source port ranges**：`*`。
   - **Destination**：`Any`。
   - **Destination port ranges**：新加坡填 `8443`；东京填 `443`。
   - **Protocol**：选择 **UDP**（不要选 TCP）。
   - **Action**：`Allow`。
   - **Name**：如 `Allow-Hysteria2-UDP-8443`。
3. 点击 **Add** 保存，并在实例 **Overview** 页记下 **Public IP address**。

## 4. 安装 Hysteria 2 与导入客户端

从这一步开始，服务器内部操作与 AWS Lightsail **完全一致**：

1. 使用 SSH 登录 Azure 虚拟机：
   ```bash
   chmod 600 /path/to/your-azure-key.pem
   ssh -i /path/to/your-azure-key.pem azureuser@YOUR_AZURE_PUBLIC_IP
   ```
2. 按 [中文实操指南 第 3 节](实操指南.md#3-在第一台空白-ubuntu-上安装服务端) 安装 Hysteria 2 官方服务端、生成本节点独立自签证书与 SHA-256 指纹、写入 [服务端模板](../examples/hysteria-server.yaml) 并启动服务。
   - 若系统启用了 `ufw`，记得同步执行 `sudo ufw allow "$VPN_PORT/udp"`。
3. 按 [中文实操指南 第 4 节](实操指南.md#4-导入设备一份资料包含两条线) 将 Azure 节点的公网 IP、端口、口令、SNI 和指纹填入私有 `flclash.yaml` 或 Shadowrocket 链接，并按 [第 5 节](实操指南.md#5-验收两地出口和故障切换) 完成真实出口、错误指纹拦截与切换验收。

## 5. 查账、控流量与彻底停用

- **查看余额与流量**：在 **Cost Management + Billing → Cost analysis** 中按资源组和计费项（Meter）查看，重点关注 **Bandwidth（Data Transfer Out）**、**Virtual Machines**、**Managed Disks** 和 **IP Addresses** 四项。
- **防流量超额**：升级 Pay-As-You-Go 后，单节点日常网页、文字 AI、社媒通常在 10～30 GB/月；尽量避免通过 Azure 节点长期挂 4K 视频或大文件下载，防止触发 $0.12/GB 的亚太区流量费。
- **彻底停用（避免隐形扣费）**：
  - 仅在 Azure 关机（Stop/Deallocate）只会停止计算计费，**磁盘和公网 IP 仍会继续计费**。
  - 确认不再使用时，直接进入 **Resource groups**，选中当初新建的资源组（如 `rg-proxy-sg`），点击 **Delete resource group**，可一次性把 VM、磁盘、网卡、公网 IP 和 NSG 全部清理干净。
