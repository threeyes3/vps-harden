# VPS SSH 加固脚本（无 UFW）

[中文](#中文) | [English](#english)

---

## 中文

一个用于 Linux VPS 的 **SSH 一键加固脚本**（不依赖 UFW），支持：

- **交互式运行**：一步一步询问并配置（推荐）
- **快速运行模式（非交互）**：适合远程一键 / 批量部署（需谨慎）

脚本主要做这些事：

- ✅ 修改 SSH 端口（默认 2222，可选）
- ✅ 可选禁用密码登录（建议启用，仅密钥）
- ✅ 可选配置 `AllowUsers`（限制允许登录的用户）
- ✅ 安装并启用 Fail2Ban（保护 sshd，防爆破）
- ✅ 自动备份 `sshd_config`，并做 `sshd -t` 语法检查
- ✅ 执行前输出变更摘要、可选 diff 预览
- ✅ 执行后输出可复制的 SSH 登录命令

> 说明：本项目 **不使用 UFW**。更推荐你在云厂商（阿里云 / 腾讯云 / AWS / Oracle 等）通过 **安全组** 控制端口和来源 IP。

---

### 目录

- [使用前必读（强烈建议）](#使用前必读强烈建议)
- [安装 / 运行方式](#安装--运行方式)
  - [方式 A：交互式运行（推荐）](#方式-a交互式运行推荐)
  - [方式 B：快速运行模式（非交互远程一键）](#方式-b快速运行模式非交互远程一键)
- [参数说明（快速运行模式）](#参数说明快速运行模式)
- [执行后检查](#执行后检查)
- [Fail2Ban 常用命令](#fail2ban-常用命令)
- [回滚方法（救援）](#回滚方法救援)
- [常见问题（FAQ）](#常见问题-faq)

---

## 使用前必读（强烈建议）

⚠️ **避免锁死的关键原则：**

1. **请先确保你已经配置了 SSH 密钥登录**（`~/.ssh/authorized_keys` 存在且可用）  
2. **改端口前**，如果你是云 VPS：请先在云控制台 **安全组放行新端口**（例如 40022）  
3. 执行脚本时 **不要关闭当前 SSH 会话**  
4. 脚本完成后，务必在 **新终端** 测试新端口登录成功，再考虑关闭旧端口（如 22）

✅ 本脚本包含“防锁死检查”：当你选择禁用密码登录但未检测到可用公钥时，会提示并在非交互模式下自动降级为保留密码登录（尽量避免锁死）。

---

## 安装 / 运行方式

你可以选择下面两种方式运行：

---

### 方式 A：交互式运行（推荐）

适合：你正在手动 SSH 到 VPS 上进行加固，希望逐步选择配置（**最稳妥**）。

```bash
curl -fsSL https://raw.githubusercontent.com/threeyes3/vps-ssh-harden/main/harden-ssh.sh | sudo bash
```

交互式会询问：
- 新 SSH 端口
- 是否禁用密码登录
- 是否启用 Fail2Ban 以及参数
- 是否设置 AllowUsers（可选）

---

### 方式 B：快速运行模式（非交互/远程一键）

适合：你想“一行命令直接跑”，或批量部署（**更快，但更容易因环境差异导致问题**）。

#### 快速运行示例（强烈建议你先理解参数含义）

```bash
curl -fsSL https://raw.githubusercontent.com/threeyes3/vps-ssh-harden/main/harden-ssh.sh | sudo \
NEW_PORT=40022 \
DISABLE_PASSWORD=yes \
ENABLE_FAIL2BAN=yes \
FAIL2BAN_MAXRETRY=3 \
FAIL2BAN_FINDTIME=10m \
FAIL2BAN_BANTIME=24h \
ALLOW_USERS="root,ubuntu" \
bash
```

#### 关于“快速运行模式”的重要说明

快速运行模式不会提问，你传什么它就按什么做。请确保：

- ✅ 新端口已在云安全组放行（否则 SSH 会断开且新端口不可达）
- ✅ 你有可用 SSH 密钥（否则禁用密码会锁死；脚本会尽量做防护，但不要赌）
- ✅ `ALLOW_USERS` 里包含你实际登录的用户名（否则你自己也会被拒绝）
- ✅ 端口没有被占用（脚本会检测到端口占用并在非交互模式下直接退出）

> 建议策略：  
> 先用 **交互式** 在一台机器上跑通，再把参数固定下来用于批量“快速运行”。

---

## 参数说明（快速运行模式）

| 参数 | 默认值 | 说明 |
|---|---:|---|
| `NEW_PORT` | `2222` | 新的 SSH 端口 |
| `DISABLE_PASSWORD` | `yes` | 是否禁用密码登录（`yes/no`） |
| `ENABLE_FAIL2BAN` | `yes` | 是否启用 Fail2Ban（`yes/no`） |
| `FAIL2BAN_MAXRETRY` | `3` | 最大失败次数 |
| `FAIL2BAN_FINDTIME` | `10m` | 时间窗口（如 `10m`） |
| `FAIL2BAN_BANTIME` | `24h` | 封禁时长（如 `24h`） |
| `ALLOW_USERS` | 空 | 允许登录的用户名列表（逗号或空格分隔；留空表示不限制） |

---

## 执行后检查

脚本执行完成后，会输出一条可复制的 SSH 命令，例如：

```bash
ssh -p 40022 ubuntu@<VPS_IP>
```

请按顺序检查：

1. ✅ **新开一个终端**，用新端口登录成功
2. ✅ 登录成功后，再考虑在云安全组里关闭 22 端口入站
3. ✅ 检查 Fail2Ban 状态（如果你启用了）

---

## Fail2Ban 常用命令

```bash
# 查看 fail2ban 总状态
sudo fail2ban-client status

# 查看 sshd jail 状态（封禁数量、最近封禁 IP 等）
sudo fail2ban-client status sshd

# 解封某个 IP
sudo fail2ban-client set sshd unbanip 1.2.3.4
```

---

## 回滚方法（救援）

如果你改错配置导致 SSH 不可登录：

1. 使用云厂商控制台的 **VNC/串口/救援模式** 登录服务器
2. 回滚 sshd 配置备份（脚本会创建类似备份文件）：

```bash
sudo ls -l /etc/ssh/sshd_config.bak.*
sudo cp -a /etc/ssh/sshd_config.bak.<TIMESTAMP> /etc/ssh/sshd_config
sudo sshd -t
sudo systemctl restart sshd
```

---

## 常见问题（FAQ）

### 1) 为什么不使用 UFW？
很多云 VPS 场景更推荐使用 **云安全组** 控制入站规则；此外 Fail2Ban 本身会通过系统防火墙机制封禁攻击 IP，本脚本专注 SSH 加固，不强耦合 UFW。

### 2) 我改了端口后登不回去
最常见原因：**云安全组没有放行新端口**。  
请先在云控制台放行 `TCP NEW_PORT`，再测试登录。

### 3) 端口被占用怎么办？
脚本会检测端口占用：
- 交互模式会提示你换一个端口
- 快速运行模式会直接退出，要求更换 `NEW_PORT`

### 4) AllowUsers 怎么用更安全？
建议把你常用登录用户加入，例如：
- `ALLOW_USERS="root,ubuntu"`（或你自己的用户名）
如果你不确定，先留空（不限制），等确认没问题再加限制。

---

## English

A one-click **SSH hardening script** for Linux VPS (No UFW). Supports:

- **Interactive mode** (recommended): guided configuration
- **Fast mode (non-interactive)**: suitable for remote one-liners and batch provisioning (use with care)

What it does:

- ✅ Change SSH port (default 2222)
- ✅ Optionally disable password authentication (recommended: key-only)
- ✅ Optional `AllowUsers` (limit which users can SSH in)
- ✅ Install & enable Fail2Ban (protect sshd against brute force)
- ✅ Backup `sshd_config` and run `sshd -t` syntax checks
- ✅ Print a change summary and (optionally) a diff preview
- ✅ Print a ready-to-copy SSH command after completion

> Note: This project does **not** use UFW. On cloud VPS, it’s recommended to manage inbound rules via **cloud security groups**.

---

### Table of Contents

- [Before You Run (Strongly Recommended)](#before-you-run-strongly-recommended)
- [Run Options](#run-options)
  - [Option A: Interactive Mode (Recommended)](#option-a-interactive-mode-recommended)
  - [Option B: Fast Mode (Non-interactive / One-liner)](#option-b-fast-mode-non-interactive--one-liner)
- [Parameters (Fast Mode)](#parameters-fast-mode)
- [Post-run Checks](#post-run-checks)
- [Fail2Ban Commands](#fail2ban-commands)
- [Rollback / Recovery](#rollback--recovery)
- [FAQ](#faq)

---

## Before You Run (Strongly Recommended)

⚠️ To avoid locking yourself out:

1. Make sure **SSH key login is already working** (`~/.ssh/authorized_keys` exists and is valid)
2. On cloud VPS: **open the new SSH port in your security group first**
3. Do **not** close your current SSH session while running this script
4. After completion, test the new port in a **new terminal** before closing port 22

✅ This script includes basic anti-lockout checks. If password auth is disabled but no usable keys are detected, it will warn you and (in non-interactive mode) may automatically keep password auth to reduce lockout risk.

---

## Run Options

### Option A: Interactive Mode (Recommended)

```bash
curl -fsSL https://raw.githubusercontent.com/threeyes3/vps-ssh-harden/main/harden-ssh.sh | sudo bash
```

The script will prompt for:
- SSH port
- Disable password auth or not
- Fail2Ban settings
- Optional AllowUsers list

---

### Option B: Fast Mode (Non-interactive / One-liner)

```bash
curl -fsSL https://raw.githubusercontent.com/threeyes3/vps-ssh-harden/main/harden-ssh.sh | sudo \
NEW_PORT=40022 \
DISABLE_PASSWORD=yes \
ENABLE_FAIL2BAN=yes \
FAIL2BAN_MAXRETRY=3 \
FAIL2BAN_FINDTIME=10m \
FAIL2BAN_BANTIME=24h \
ALLOW_USERS="root,ubuntu" \
bash
```

#### Important notes about Fast Mode

Fast mode does not prompt. You must ensure:

- ✅ The new port is allowed in your cloud security group
- ✅ SSH key auth is ready (otherwise disabling password auth may lock you out)
- ✅ `ALLOW_USERS` includes your actual SSH login user
- ✅ The chosen port is not already in use (fast mode will exit if occupied)

Recommendation: run once in **interactive mode** on a test VPS, then reuse the same parameters for batch deployment.

---

## Parameters (Fast Mode)

| Parameter | Default | Description |
|---|---:|---|
| `NEW_PORT` | `2222` | New SSH port |
| `DISABLE_PASSWORD` | `yes` | Disable password auth (`yes/no`) |
| `ENABLE_FAIL2BAN` | `yes` | Enable Fail2Ban (`yes/no`) |
| `FAIL2BAN_MAXRETRY` | `3` | Max retries |
| `FAIL2BAN_FINDTIME` | `10m` | Find time window |
| `FAIL2BAN_BANTIME` | `24h` | Ban time |
| `ALLOW_USERS` | empty | Allowed SSH users (comma/space separated; empty = no restriction) |

---

## Post-run Checks

The script prints a command like:

```bash
ssh -p 40022 ubuntu@<VPS_IP>
```

Checklist:
1. ✅ Open a new terminal and confirm you can SSH in via the new port
2. ✅ Only then consider closing inbound port 22 in your security group
3. ✅ Check Fail2Ban status if enabled

---

## Fail2Ban Commands

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip 1.2.3.4
```

---

## Rollback / Recovery

If you cannot SSH in after changes:

1. Use your provider console (VNC/serial/rescue) to access the VPS
2. Restore the backup:

```bash
sudo ls -l /etc/ssh/sshd_config.bak.*
sudo cp -a /etc/ssh/sshd_config.bak.<TIMESTAMP> /etc/ssh/sshd_config
sudo sshd -t
sudo systemctl restart sshd
```

---

## FAQ

### Why not UFW?
Cloud security groups are often the preferred layer for inbound control. Fail2Ban also handles banning via system mechanisms. This repo focuses on SSH hardening and avoids coupling to UFW.

### I changed the port and can’t login
Most common cause: the new port is not allowed in your security group.

### What about AllowUsers?
If you’re not sure, leave it empty. Once stable, restrict users to reduce exposure.

---
