# OpenClaw Changes Log

> 本文档记录 OpenClaw 助手对系统内所有文件/配置的删改操作。
> ⚠️ 规则：此文件中的记录 OpenClaw 不得删除，需得到用户（salim）许可方可删除。

---

## 2026-05-17

### 01:46 - Samba 共享配置检查
- **操作**: 读取 /etc/samba/smb.conf
- **发现**: [Sharing] 段已存在，路径指向 /run/media/salim/FeatureDisk/Sharing
- **变更**: 无（仅读取）
- **状态**: 已完成

### 01:46 - Samba 无密码访问配置（待执行）
- **操作**: 计划修改 /etc/samba/smb.conf [Sharing] 段
- **变更**: guest ok = no → yes, 注释 valid users = salim
- **原因**: 用户要求局域网无密码访问
- **需要命令**: sudo sed/sudo systemctl restart smbd
- **状态**: ✅ 已执行（用户手动完成，01:55）

### 01:57 - FeatureDisk 父目录权限修改
- **操作**: sudo chmod 755 /run/media/salim/FeatureDisk
- **原因**: 父目录 700 权限导致 Samba guest 无法进入
- **状态**: ✅ 已执行

### 01:59 - /run/media/salim ACL 修改
- **操作**: sudo setfacl -m o:r-x /run/media/salim
- **原因**: 该目录 ACL other::--- 阻止了所有非 salim 用户（含 Samba guest）
- **变更**: other 权限从 --- 改为 r-x
- **状态**: ✅ 已执行

### 01:59 - /run/media/salim ACL 修改
- **操作**: sudo setfacl -m o:r-x /run/media/salim
- **原因**: 该目录 ACL other::--- 阻止了所有非 salim 用户（含 Samba guest）
- **变更**: other 权限从 --- 改为 r-x
- **状态**: ✅ 已执行

### 02:07 - Zerotier 启动
- **操作**: sudo systemctl start zerotier-one
- **状态**: ✅ 已执行（用户手动，active）

### 02:07 - Samba 共享验证通过
- **操作**: smbclient -N //localhost/Sharing -c "ls"
- **结果**: Anonymous login successful，文件正常列出
- **备注**: Windows 端用 IP (\\192.168.3.229\Sharing) 可正常访问
- **状态**: ✅ 完成

### 02:08 - nmbd 未启动 → 已启动
- **发现**: nmbd 处于 inactive 状态（ExecCondition 检查失败）
- **操作**: 
  1. 创建 /etc/systemd/system/nmbd.service.d/override.conf 覆盖 ExecCondition
  2. sudo systemctl daemon-reload && sudo systemctl start nmbd
- **状态**: ✅ 已启动（active running，02:14）
- **备注**: 现在 Windows 可通过主机名访问共享

### 02:25 - API Key 存储
- **操作**: 将 API key 存入 /run/media/salim/FeatureDisk/Sharing/.api-key
- **权限**: chmod 600（仅 owner 可读写）
- **状态**: ✅ 已存储

### 02:48 - 开放免密 sudo 全权限
- **操作**: echo "salim ALL=(ALL) NOPASSWD: ***" > /etc/sudoers.d/openclaw-nopasswd
- **后果**: OpenClaw 可执行任意 root 命令无需密码
- **风险等级**: 🔴 高
- **状态**: ✅ 已执行（02:49）

### 02:55 - 卸载并禁止 Windows NTFS 盘自动挂载
- **发现**: 「软件」(nvme0n1p5, 1.6T) 被 udisks2 自动挂载到 /run/media/salim/软件
- **操作 1**: sudo umount /run/media/salim/软件
- **操作 2**: 创建 /etc/udev/rules.d/99-block-windows-disks.rules
  - 屏蔽 3 个 NTFS 分区：软件 (2F41861BE0645388)、系统 (B4EE922BF666B176)、nvme1n1p1 (0D7E0FB40D7E0FB4)
- **操作 3**: sudo udevadm control --reload-rules && trigger
- **状态**: ✅ 已生效，重启后仍有效

### 03:00 - 创建知识文档
- **操作**: 将今日所有操作整理为文档
- **文件**: knowledges/2026-05-17-system-setup.md
- **内容**: 6 大项操作，含命令逐行拆解
- **状态**: ✅ 完成

### 03:04 - 创建 OpenClaw 命令行操作指南
- **操作**: 整理 OpenClaw CLI 常用命令
- **文件**: knowledges/openclaw说明书.md
- **内容**: 12 个模块，含状态查看、网关、通道、配置、插件、模型、定时任务、会话、诊断、更新、技能
- **状态**: ✅ 完成

### 03:08 - 创建工作空间文件指南
- **文件**: knowledges/workspace文件指南.md
- **内容**: 详解 AGENTS/SOUL/TOOLS/USER/IDENTITY/MEMORY 等文件的作用、Skills vs Tools 区别、跨通道建立规则的方法
- **状态**: ✅ 完成

---


