# Google Cloud（GCP）搭建 Hysteria 2 实操指南

本文说明如何在 **Google Cloud Platform（GCP）** 上创建东京（`asia-northeast1`）与新加坡（`asia-southeast1`）节点，并复用本仓库的 [Hysteria 2 服务端模板](../examples/hysteria-server.yaml) 和 [FlClash 双节点模板](../examples/flclash.yaml)。

> [!CAUTION]
> **GCP 最大的坑是网络流量计费，建机前务必看清以下两点：**
> 1. 创建实例时，**必须把网络服务层级（Network Service Tier）从默认的 Premium 改为 Standard**。默认 Premium Tier 发往中国大陆的流量**无任何免费额度**，从第 1 字节起按 **$0.23/GB** 计费；切换为 Standard Tier 后可享受每月 200 GB 标准层出站免费额度，超额单价也大幅降低。
> 2. 订阅 Google AI Pro / Ultra（Gemini 会员）通过 Google Developer Program 获得的月度抵用金，**主要适用于 Gemini API / Vertex AI 等 GenAI 服务**，不能默认抵扣 Compute Engine 和公网流量，请以你账户 **Billing → Credits** 实际适用范围为准。

## 1. 准备 GCP 账户与确认额度

1. 进入 [Google Cloud 免费试用页](https://cloud.google.com/free)，登录你的 Google 账号（建议开启两步验证），完成账单资料绑定。
   - 符合条件的新户可获得 **$300 体验金（有效期 90 天）**，试用期内默认不会自动转为正式扣费账户，到期或用尽后服务会暂停，除非你手动点击升级。
2. 如果你已升级为正式结算账户，或准备使用其他抵用金，请先在 **Billing → Credits** 核对抵用金的有效期与适用服务范围，并在 **Billing → Budgets & alerts** 设置预算提醒（例如 $5、$15、$30）。
3. 创建一个独立项目（例如 `personal-proxy`），方便集中管理防火墙、实例和账单。

## 2. 配置 VPC 防火墙规则（放行 UDP）

GCP 的防火墙是项目级（VPC 级）的，推荐通过 **网络标签（Target tags）** 精准绑定到代理实例。

| 地区 | GCP Region | 建议监听端口 | 网络标签示例 | 证书名称（SNI） |
| --- | --- | --- | --- | --- |
| 东京 | `asia-northeast1` | UDP 443 | `hy2-jp` | `jp.vpn.invalid` |
| 新加坡 | `asia-southeast1` | UDP 8443 | `hy2-sg` | `sg.vpn.invalid` |

操作步骤：

1. 进入 **VPC network → Firewall → Create firewall rule**。
2. 先为新加坡建一条规则：
   - **Name**：`allow-hy2-sg-udp-8443`
   - **Direction of traffic**：`Ingress`（入站）
   - **Action on match**：`Allow`
   - **Targets**：`Specified target tags`
   - **Target tags**：输入 `hy2-sg`
   - **Source IPv4 ranges**：`0.0.0.0/0`（若只从固定公网 IP 连入，可改为对应网段）
   - **Protocols and ports**：勾选 **UDP**，填写 `8443`
3. 点击 **Create**。部署东京节点时，再建一条规则 `allow-hy2-jp-udp-443`，Target tags 填 `hy2-jp`，UDP 填 `443`。

## 3. 创建 Compute Engine 实例（关键：选 Standard Tier）

以新加坡为例，验证通过后再按同样步骤建东京：

1. 进入 **Compute Engine → VM instances → Create instance**。
2. **Name & Region**：
   - **Region**：新加坡选 `asia-southeast1`；东京选 `asia-northeast1`（也可选台湾 `asia-east1`）。
   - *说明：若想使用永久免费 Always Free `e2-micro`，则只能选美国 `us-west1`、`us-central1` 或 `us-east1`，且延迟明显高于亚洲区。*
3. **Machine configuration**：
   - 选 **E2** 系列 → **`e2-micro`**（2 vCPU 共享核心, 1 GB RAM），个人 Hysteria 2 绰绰有余。
4. **OS and storage（启动盘）**：
   - 点击 **Change**，Operating system 选 **Ubuntu**，Version 选 **Ubuntu 24.04 LTS（x86/64）**。
   - Boot disk type 选 **Standard persistent disk**（标准永久性磁盘，费用最低，Always Free 下 30 GB 内免费），大小保持 10～20 GB 即可。
5. **Networking（最关键一步，防止流量刺客！）**：
   - 展开 **Networking** → **Network tags**，填入前面设置的标签（新加坡填 `hy2-sg`，东京填 `hy2-jp`）。
   - 展开 **Network interfaces**，点击默认网卡（`default`）进行编辑：
     - **External IPv4 address**：若在 $300 体验期内，可直接保留 Ephemeral（临时 IP，只要不关机通常不变）或预留 Static IP（绑定运行中的实例一般不额外收费，但闲置未绑定会收费）。
     - **Network Service Tier**：**务必从 `Premium` 切换为 `Standard`**！
6. 核对右侧月度预估费用后，点击 **Create**。

## 4. 安装 Hysteria 2 与导入客户端

1. 在 **VM instances** 列表找到新建的实例，记下它的 **External IP**，点击右侧 **SSH** 按钮在浏览器打开终端（或通过本地 `gcloud compute ssh` / 自行配置的 SSH 密钥登录）。
2. 按 [中文实操指南 第 3 节](实操指南.md#3-在第一台空白-ubuntu-上安装服务端) 安装 Hysteria 2 官方服务端、生成本节点独立自签证书与 SHA-256 指纹、写入 [服务端模板](../examples/hysteria-server.yaml) 并启动服务。
3. 按 [中文实操指南 第 4 节](实操指南.md#4-导入设备一份资料包含两条线) 将 GCP 节点的公网 IP、端口、口令、SNI 和指纹填入私有 `flclash.yaml` 或 Shadowrocket 链接，并按 [第 5 节](实操指南.md#5-验收两地出口和故障切换) 完成真实出口、错误指纹拦截与切换验收。

## 5. 查账、核对网络层级与彻底停用

- **上线后复核网络层级**：跑完测试后，进入实例详情页 → **Network interfaces**，再次确认 **Network tier** 显示为 `Standard`。第二天去 **Billing → Reports** 按 SKU 分组看一眼，确认没有出现 `Network Internet Egress ... to China` 的 Premium Tier 计费项。
- **90 天体验金到期前**：提前 5～7 天决定是否迁移到 AWS Lightsail 或 Azure。如果不想续费，不要点击升级正式账户。
- **彻底停用（避免残留扣费）**：
  - 仅 Stop 实例不会停止磁盘计费，且**静态 IP 与关机实例解绑后会按闲置 IP 计费**。
  - 确认不再使用时，在 **Compute Engine → VM instances** 删除实例，到 **VPC network → IP addresses** 释放不再使用的静态 IP，并在 **Compute Engine → Disks / Snapshots** 确认没有残留磁盘；若该 GCP 项目专用于此，也可直接在 **IAM & Admin → Settings** 关闭（Shut down）整个项目。
