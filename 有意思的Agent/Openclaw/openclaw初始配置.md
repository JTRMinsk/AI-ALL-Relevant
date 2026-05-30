# 2026-05-17 系统初始化配置

> 本文档记录 OpenClaw 助手与 salim 在 2026 年 5 月 17 日完成的系统配置工作，包含每项操作的背景、完整命令及逐行拆解。

---

## 目录

1. [Samba 局域网共享配置](#1-samba-局域网共享配置)
2. [Zerotier 虚拟网络](#2-zerotier-虚拟网络)
3. [NetBIOS 名称解析 (nmbd)](#3-netbios-名称解析-nmbd)
4. [微信通道接入](#4-微信通道接入)
5. [免密 sudo 权限](#5-免密-sudo-权限)
6. [Windows NTFS 磁盘隔离](#6-windows-ntfs-磁盘隔离)

---

## 1. Samba 局域网共享配置

### 背景

将 `/run/media/salim/FeatureDisk/Sharing` 文件夹通过 Samba 协议共享给局域网其他设备，无需密码即可访问。

### 涉及文件

- `/etc/samba/smb.conf` — Samba 主配置文件

### 操作与命令

#### 1.1 修改共享段为无密码访问

Samba 配置中 `[Sharing]` 段原本要求 `valid users = salim` + `guest ok = no`，需要改为允许来宾访问。

```bash
sudo sed -i '/^\[Sharing\]/,/^\[/{
  s/guest ok = no/guest ok = yes/
  s/valid users = salim/# valid users = salim/
}' /etc/samba/smb.conf
```

**逐行拆解：**

| 部分 | 说明 |
|------|------|
| `sudo sed -i` | 以 root 身份运行 sed，`-i` 表示原地修改文件 |
| `'/^\[Sharing\]/,/^\[/'` | 范围匹配：从 `[Sharing]` 行开始，到下一个 `[` 开头的行结束 |
| `s/guest ok = no/guest ok = yes/` | 替换：`guest ok = no` → `guest ok = yes`，允许匿名访客 |
| `s/valid users = salim/# valid users = salim/'` | 注释掉用户限制行 |

#### 1.2 重启 Samba 服务

```bash
sudo systemctl restart smbd
```

| 部分 | 说明 |
|------|------|
| `systemctl restart` | 停止并重新启动服务 |
| `smbd` | Samba SMB 守护进程（处理文件共享请求） |

#### 1.3 修复父目录 ACL 权限

问题：`/run/media/salim` 目录的 ACL 中 `other::---` 阻止了 Samba guest（nobody）进入。

```bash
sudo setfacl -m o:r-x /run/media/salim
```

| 部分 | 说明 |
|------|------|
| `setfacl` | 设置文件/目录的访问控制列表（ACL） |
| `-m o:r-x` | 修改（modify）other 用户权限为读+执行（r-x），允许进入目录 |
| `/run/media/salim` | 目标目录 |

#### 1.4 修复父目录基础权限

```bash
sudo chmod 755 /run/media/salim/FeatureDisk
```

| 部分 | 说明 |
|------|------|
| `chmod 755` | owner=rwx, group=r-x, other=r-x（所有人可进入和读取） |

#### 1.5 验证共享

```bash
smbclient -N //localhost/Sharing -c "ls"
```

| 部分 | 说明 |
|------|------|
| `smbclient` | Samba 命令行客户端 |
| `-N` | 无密码登录 |
| `//localhost/Sharing` | 连接本地 Samba 的 Sharing 共享 |
| `-c "ls"` | 执行 `ls` 命令，列出文件 |

#### 最终效果

局域网设备可通过以下地址访问：
- `\\192.168.3.229\Sharing`（IP 地址）
- 无需用户名和密码，即可读写

---

## 2. Zerotier 虚拟网络

### 背景

启动 Zerotier 以加入 `unitednetwork` 虚拟局域网。

### 操作与命令

```bash
sudo systemctl start zerotier-one
```

| 部分 | 说明 |
|------|------|
| `systemctl start` | 启动 systemd 管理的服务 |
| `zerotier-one` | Zerotier 守护进程 |

#### 验证状态

```bash
sudo zerotier-cli listnetworks
```

输出示例：
```
200 listnetworks a581878f7d9bf67f unitednetwork 7e:19:49:60:ba:82 OK PRIVATE ztfl6by6ic 10.179.137.24/24
```

| 字段 | 值 | 说明 |
|------|-----|------|
| Network ID | `a581878f7d9bf67f` | Zerotier 网络标识 |
| Name | `unitednetwork` | 网络名称 |
| Status | `OK` | 连接正常 |
| Type | `PRIVATE` | 私有网络 |
| IP | `10.179.137.24/24` | 分配到的虚拟 IP |

---

## 3. NetBIOS 名称解析 (nmbd)

### 背景

nmbd 允许局域网设备通过主机名（`\\SALIM-MS-7D17`）而非 IP 地址访问 Samba 共享。默认配置下 nmbd 拒绝启动（ExecCondition 检查失败）。

### 操作与命令

#### 3.1 创建 override 配置跳过检查

```bash
sudo mkdir -p /etc/systemd/system/nmbd.service.d
sudo tee /etc/systemd/system/nmbd.service.d/override.conf << 'EOF'
[Service]
ExecCondition=
EOF
```

| 部分 | 说明 |
|------|------|
| `mkdir -p` | 创建目录（已存在则忽略） |
| `tee` | 将标准输入写入文件并输出到终端 |
| `<< 'EOF'` | heredoc：将多行文本作为输入，引号阻止变量展开 |
| `[Service]` | systemd service 段 |
| `ExecCondition=` | 空的 ExecCondition 覆盖原有的条件检查（等于移除了检查） |

#### 3.2 重新加载并启动

```bash
sudo systemctl daemon-reload
sudo systemctl start nmbd
```

| 部分 | 说明 |
|------|------|
| `daemon-reload` | 让 systemd 重新读取所有 unit 文件（含 override） |
| `start nmbd` | 启动 NetBIOS 名称服务 |

---

## 4. 微信通道接入

### 背景

通过 `@tencent-weixin/openclaw-weixin` 插件，让 OpenClaw 接入微信。

### 操作与命令

#### 4.1 扫码登录

```bash
/home/salim/.hermes/node/bin/openclaw channels login --channel openclaw-weixin
```

| 部分 | 说明 |
|------|------|
| `channels login` | 登录到指定通道 |
| `--channel openclaw-weixin` | 目标通道为微信插件 |

终端显示二维码后用微信扫描并确认登录。

#### 4.2 重启网关

```bash
/home/salim/.hermes/node/bin/openclaw gateway restart
```

| 部分 | 说明 |
|------|------|
| `gateway restart` | 重启 OpenClaw 网关进程，让新通道配置生效 |

#### 验证

```bash
/home/salim/.hermes/node/bin/openclaw channels status
```

---

## 5. 免密 sudo 权限

### 背景

允许 OpenClaw 助手无需密码执行 root 命令，避免每次操作都需要用户手动输入 sudo。

⚠️ **风险等级：高** — 任何能向 OpenClaw 发送消息的人理论上都能执行 root 命令。

### 操作与命令

```bash
echo "salim ALL=(ALL) NOPASSWD: ***" | sudo tee /etc/sudoers.d/openclaw-nopasswd && sudo chmod 440 /etc/sudoers.d/openclaw-nopasswd
```

| 部分 | 说明 |
|------|------|
| `echo "..."` | 输出 sudo 规则 |
| `\| sudo tee` | 以 root 身份写入文件（`>` 重定向不继承 sudo） |
| `/etc/sudoers.d/` | sudo 规则片段目录，主配置 `#includedir` 会加载此目录 |
| `salim ALL=(ALL) NOPASSWD: ***` | 用户 salim 在所有主机以所有身份执行所有命令均无需密码 |
| `&&` | 前一条成功才执行后一条 |
| `chmod 440` | 设置文件权限：owner 可读、group 可读、other 无权限（sudo 安全要求） |

### 验证

```bash
sudo whoami
# 输出: root（不需要输入密码）
```

---

## 6. Windows NTFS 磁盘隔离

### 背景

双系统（Windows + Ubuntu）共存的机器上，Ubuntu 的 udisks2 服务会自动挂载检测到的 NTFS 分区。为防止 Linux 误操作 Windows 系统盘，需要禁止自动挂载。

### 涉及的磁盘

| 设备 | 大小 | 标签 | 文件系统 |
|------|------|------|----------|
| nvme0n1p3 | 296.6G | 系统 | NTFS |
| nvme0n1p5 | 1.6T | 软件 | NTFS |
| nvme1n1p1 | 931.5G | (无) | NTFS |

### 操作与命令

#### 6.1 卸载已自动挂载的磁盘

```bash
sudo umount /run/media/salim/软件
```

| 部分 | 说明 |
|------|------|
| `umount` | 卸载（断开挂载连接），磁盘数据不丢失 |

#### 6.2 创建 udev 规则禁止自动挂载

```bash
sudo tee /etc/udev/rules.d/99-block-windows-disks.rules << 'EOF'
# Block udisks2 auto-mount for Windows NTFS disks
ENV{ID_FS_UUID}=="2F41861BE0645388", ENV{UDISKS_IGNORE}="1", ENV{UDISKS_AUTO}="0"
ENV{ID_FS_UUID}=="B4EE922BF666B176", ENV{UDISKS_IGNORE}="1", ENV{UDISKS_AUTO}="0"
ENV{ID_FS_UUID}=="0D7E0FB40D7E0FB4", ENV{UDISKS_IGNORE}="1", ENV{UDISKS_AUTO}="0"
EOF
```

| 部分 | 说明 |
|------|------|
| `ENV{ID_FS_UUID}=="..."` | 匹配文件系统 UUID（唯一标识磁盘分区） |
| `UDISKS_IGNORE="1"` | 告诉 udisks2 忽略此设备 |
| `UDISKS_AUTO="0"` | 禁止自动挂载 |

#### 6.3 重新加载 udev 规则

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

| 部分 | 说明 |
|------|------|
| `udevadm control --reload-rules` | 重新加载所有 udev 规则文件 |
| `udevadm trigger` | 用新规则重新检查已有设备 |

### 最终效果

- Linux 下只能看到 FeatureDisk（EXT4），Windows 的三个 NTFS 盘不会被挂载
- Windows 盘的数据完全不受 Linux 操作影响
- 重启后 udev 规则仍然有效

---

## 变更日志

本文档对应操作记录详见 `../OpenClawChanges.md`。

---

> 生成时间：2026-05-17 03:00 CST
> 生成者：OpenClaw (deepseek-v4-pro)
