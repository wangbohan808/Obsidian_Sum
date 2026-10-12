---
type: 项目
status: 整理中
tags:
  - 技术/Linux
project: 猫咪监控
created: "2026-08-23"
updated: "2026-08-23"
up: "[[00-README]]"
---

# Linux 虚拟机通过 SSH 向 Windows 宿主机传文件

> 环境：VMware 虚拟机与宿主机在同一虚拟局域网（VMnet8 NAT），**不使用共享文件夹 / HGFS**。

---

## 1. 总体原理

Cursor Agent **不能直接写 Windows 的 `D:\`**。它只在 Linux 里执行命令。要让文件出现在宿主机，必须先有一条「虚拟机 → 宿主机」的通道。

本次通道是 **SSH（TCP 22）**，不是 VMware 共享文件夹：

```yaml
Windows 宿主机                         Linux 虚拟机（Cursor）
VMware Network Adapter VMnet8
  192.168.203.1  ◄── 同一虚拟交换机 ──►  192.168.203.x
  OpenSSH sshd :22                         scp / sftp / rsync
         │
         ▼
D:\Github_Code\项目规划\猫咪饮食监控\
```

数据路径：虚拟网卡 → TCP 三次握手 :22 → SSH 认证 → SFTP 写 NTFS。  
不经过 `/mnt/hgfs`，也不经过家里 Wi-Fi（`192.168.70.37`）。

和 HGFS 的对比：

| | HGFS `/mnt/hgfs` | 本次 SSH |
| --- | --- | --- |
| 通道 | 虚拟机监视器文件系统 | 虚拟局域网 TCP/22 |
| 要不要共享文件夹 | 要 | 不要 |
| Agent 动作 | `cp` 到挂载点 | `scp`/`rsync` 到 `wangb@192.168.203.1` |

宿主机 `ipconfig` 中与虚拟机相关的地址：

| 适配器 | IPv4 | 何时使用 |
| --- | --- | --- |
| VMnet8 | **192.168.203.1** | NAT（本次使用） |
| VMnet1 | 192.168.115.1 | Host-only |
| WLAN | 192.168.70.37 | 仅桥接时 |

---

## 2. SSH 公钥登录在验什么

```text
客户端（虚拟机）                         服务端（Windows sshd）
私钥 ~/.ssh/id_ed25519_winhost           打开 AuthorizedKeysFile 指定的文件
用私钥对挑战做签名                       用文件里的公钥验签
                                         通过 → 免密；失败 → 要密码
```

- **私钥只留在虚拟机**，证明「我是钥匙主人」。
- **公钥必须出现在 Windows sshd 会打开的那个文件里**。sshd **不会全盘搜索** `.pub`。
- 路径写死是为了安全和按用户隔离。

OpenSSH 默认尝试的私钥文件名是 `id_ed25519` / `id_rsa` 等。本次文件名是 **`id_ed25519_winhost`**，因此必须在 `~/.ssh/config` 里写 `IdentityFile`，或命令加 `-i`。否则即使密钥存在，`ssh`/`scp` **也不会自动用它**。

ROCK 5C 开发板与这把钥匙无关，除非把私钥另行拷到板子上。

---

## 3. 完整操作流程（已实测）

### 3.1 宿主机：安装并启动 OpenSSH Server

管理员 PowerShell：

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' `
  -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

用户名：`wangb`（`whoami` → `wangbohan\wangb`）。

### 3.2 虚拟机：生成专用密钥

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519_winhost -C "vm-to-winhost"
cat ~/.ssh/id_ed25519_winhost.pub
```

实际公钥（一行）：

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAXdzvelnE9sclvnwPWVS/jOngnFaFpJBTHFot+IdU1Y vm-to-winhost
```

指纹：`SHA256:JrYg2kOR7ozfqVWGOzkaonQ1t0hfaODFUnYguMnXxF4`

### 3.3 虚拟机：SSH config

`~/.ssh/config`：

```text
Host winhost
    HostName 192.168.203.1
    User wangb
    IdentityFile ~/.ssh/id_ed25519_winhost
    IdentitiesOnly yes
```

```bash
chmod 600 ~/.ssh/config ~/.ssh/id_ed25519_winhost
```

之后 `ssh winhost` 等价于指定用户、IP、私钥。

### 3.4 宿主机：放入公钥

普通用户文件：

```text
C:\Users\wangb\.ssh\authorized_keys
```

必须是文件名 `authorized_keys`（无 `.txt`），一行一个公钥，UTF-8 无 BOM。

PowerShell 里 `icacls ... (OI)(CI)F` 会把括号当成自己的语法。应使用：

```powershell
cmd /c "icacls %USERPROFILE%\.ssh /inheritance:r"
cmd /c "icacls %USERPROFILE%\.ssh /grant %USERNAME%:(OI)(CI)F"
cmd /c "icacls %USERPROFILE%\.ssh\authorized_keys /inheritance:r"
cmd /c "icacls %USERPROFILE%\.ssh\authorized_keys /grant %USERNAME%:F"
cmd /c "icacls %USERPROFILE%\.ssh\authorized_keys /grant SYSTEM:F"
```

### 3.5 关键故障：管理员账号读错文件

现象：虚拟机日志为

```text
Offering public key: .../id_ed25519_winhost ...
Authentications that can continue: publickey,password,...
```

网络通、私钥对、sshd 开着，但服务端不接受公钥，于是要密码。

原因：Windows OpenSSH 默认对 **Administrators 组** 覆盖查找路径：

```text
Match Group administrators
    AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

即实际读取：

```text
C:\ProgramData\ssh\administrators_authorized_keys
```

**不再读取** `C:\Users\wangb\.ssh\authorized_keys`。家庭电脑上的 `wangb` 几乎都是管理员。

---

## 4. 两种修法的原理

登录时 sshd 的步骤：① 用户名 → ② 是否管理员组 → ③ 按 `sshd_config` 决定 `AuthorizedKeysFile` → ④ 打开该文件验签。

两种修法都是让第 ④ 步打开的文件里**真有那一行公钥**。

### 方法 A：改配置（让管理员也走用户目录）

在 `C:\ProgramData\ssh\sshd_config` **注释掉**整个 `Match Group administrators` 段，然后 `Restart-Service sshd`。

原理：去掉覆盖规则，`AuthorizedKeysFile` 回到 `.ssh/authorized_keys`，即 `C:\Users\wangb\.ssh\authorized_keys`。与 Linux 行为一致。sshd 只在启动时读配置。

### 方法 B：改文件位置（迁就默认规则）

不改 `Match`，把同一行公钥写入：

```text
C:\ProgramData\ssh\administrators_authorized_keys
```

并收紧 ACL（仅 SYSTEM / Administrators）。用户目录里那份登录时仍被忽略。

原理：服从微软「管理员公钥放 ProgramData」的隔离策略。

| | 方法 A | 方法 B |
| --- | --- | --- |
| 改什么 | 查找规则 | 公钥落点 |
| sshd 最终读谁 | `C:\Users\wangb\.ssh\authorized_keys` | `C:\ProgramData\ssh\administrators_authorized_keys` |
| 用户目录公钥 | 会生效 | 登录时仍忽略 |

---

## 5. 传文件

免密成功后：

```bash
ssh winhost "echo ok"
# 探路
echo test > /tmp/ssh-probe.txt
scp /tmp/ssh-probe.txt winhost:"/D:/Github_Code/项目规划/猫咪饮食监控/"
```

Windows 远程路径优先用正斜杠：`/D:/Github_Code/项目规划/猫咪饮食监控/`。

同步项目（排除大编译产物）：

```bash
cd /home/wbh808/cat-monitor-grok46
rsync -avz -e ssh \
  --exclude 'arch/work/' --exclude 'arch/rootfs/' --exclude '*.img' \
  ./ winhost:"/D:/Github_Code/项目规划/猫咪饮食监控/"
```

`arch/output/rock5c-arch.img` 实际约 2.4G，需要时再单独 `scp`。

---

## 6. 实测结论（2026-08-23）

- 虚拟机 → `192.168.203.1:22`（OpenSSH_for_Windows_9.5）连通。
- 客户端已正确出示 `id_ed25519_winhost`。
- 修正管理员 `AuthorizedKeysFile` 后免密成功：`ssh winhost` → `ok`，用户 `wangbohan\wangb`。
- 目标目录 `D:\Github_Code\项目规划\猫咪饮食监控` 已存在（`dest-ok`）。
