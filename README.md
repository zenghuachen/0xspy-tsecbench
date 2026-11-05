# kali-tsecbench:1.5

**Kali Linux × Claude Code 多智能体自动化安全评测环境**

| 文档版本 | 镜像版本 | 更新日期 | 适用架构 |
| --- | --- | --- | --- |
| 1.1 | 1.5 | 2026-11-04 | linux/amd64 |

- 问题反馈：`wechat: waterwithwood, email: zengh888@163.com`

> **安全提示**
>
> 本镜像仅用于**明确授权**的 CTF、靶场、教学、科研与安全评测环境。
> 容器内以 root 运行，Agent 拥有完整命令执行权限。禁止用于任何未授权目标。

---

## 目录

- [English Summary](#english-summary)
- [1. 基本信息](#1-基本信息)
- [2. 快速开始](#2-快速开始)
- [3. 架构](#3-架构)
- [4. 目录结构](#4-目录结构)
- [5. 环境变量](#5-环境变量)
- [6. 主要特点](#6-主要特点)
- [7. 日志与排障](#7-日志与排障)
- [8. 安全与合规](#8-安全与合规)
- [9. 测试结果](#9-测试结果)

---

## English Summary

`kali-tsecbench` is a self-contained Kali Linux container that runs an autonomous,
multi-agent security-assessment harness on top of Claude Code. A controller agent
schedules benchmark challenges, spawns one leader plus up to three member agents
per challenge, and supervises long-running work with a watchdog. The image ships
with a full pentest toolchain, Python exploit libraries, curated Chinese and
enterprise wordlists, and an offline exploit/PoC library.

It is intended for authorized CTF, lab, and benchmark environments only.

---

## 1. 基本信息

### 定位

本镜像不是单纯的 Kali 工具集合，而是一个“安全评测 Agent 运行环境”：

- 以 Kali Rolling 提供完整的渗透、逆向、取证与开发工具链；
- 以 Claude Code CLI 作为 Agent 运行时，通过 Anthropic Messages API
  兼容网关驱动；
- 通过系统提示词（`prompt.md` + 角色模板 + 方法论库）把大模型组织成
  “主控 → 组长 → 组员”的多级协作团队；
- 对接 TSec Benchmark 类平台，自动完成取题、启动靶场、攻击、提交
  flag、关闭容器与进度汇报；
- 提供**本地模式**与**托管模式**两种启动方式（见
  [2.3 启动方式](#23-启动方式)）；
- 内置守护进程，保证长时间运行中主 Agent 崩溃后自动拉起。

### 适用场景

- 授权的 CTF 竞赛、靶场演练、渗透测试教学与科研；
- AI Agent 在安全评测基准（benchmark）中的复现与对比实验；
- 多智能体协作、长时任务调度、Agent 可靠性研究；
- 需要开箱即用的 Kali 安全工具链的隔离环境。

### 不适用场景

- 任何未授权的真实目标、生产网络或他人系统；
- 作为多租户共享环境（容器内为 root，Agent 拥有完全控制权）；
- 需要图形界面或 Metasploit 框架的场景（本镜像未安装 `msfconsole`）。

---

## 2. 快速开始

### 2.1 前置条件

- Docker 20.10+（建议 24+），x86_64 主机；
- 建议 4 vCPU / 8 GB 内存 / 20 GB 可用磁盘（镜像 5.6 GB，另需日志与
  工作区空间；多 Agent 并发时内存占用会明显上升）；
- 一个 Anthropic Messages API 兼容的模型网关地址与 API Token；
- 使用 Benchmark 模式时，另需平台 Token；`BENCHMARK_BASE_URL` 可省略，
  此时使用 `prompt.md` 内置的默认平台地址；
- **本地模式**：宿主机能连接靶场 VPN，并可访问
  `https://api.deepseek.com`；容器以 `--network host` 共享宿主机网络；
- **托管模式**：由平台提供内网模型网关、网络与靶场映射，无需本地 VPN。

### 2.2 获取镜像

从镜像仓库拉取：

```bash
docker pull <registry>/kali-tsecbench:1.5
```

或从离线包导入：

```bash
docker load -i kali-tsecbench-1.5.tar
```

### 2.3 启动方式

本镜像支持两种启动方式，按运行环境选择：

| 模式 | 运行位置 | 网络 | 模型网关 | 适用场景 |
| --- | --- | --- | --- | --- |
| 本地模式 | 自有 Docker 主机 | 宿主机 VPN + `--network host` | `https://api.deepseek.com/anthropic` | 本地调试、自备 API Token |
| 托管模式 | 评测平台 | 平台网络 | `http://api.deepseek.com.tsecbench.gw/anthropic` | 平台统一启动与调度 |

#### 2.3.1 本地模式

**步骤 1：连接 VPN**

先在宿主机连接靶场 VPN（使用平台提供的 OpenVPN 配置或 VPN 客户端），
确认宿主机能访问靶场网段。

**步骤 2：启动容器**

```bash
docker run -d \
  --name tsecbench-container \
  --restart unless-stopped \
  --network host \
  -e BASE_URL="https://api.deepseek.com/anthropic" \
  -e AUTH_TOKEN="sk-***" \
  -e MODEL="deepseek-flash[1m]" \
  -e BENCHMARK_TOKEN="***" \
  kali-tsecbench:1.5
```

说明：

- `--network host` 让容器共享宿主机网络命名空间（Linux 主机），直接
  复用宿主机上的 VPN 路由；建议仅在专用测试主机使用；
- `AUTH_TOKEN` / `BENCHMARK_TOKEN` 为占位符，请替换为真实值，不要提交
  到仓库；
- 未传 `BENCHMARK_BASE_URL` 时，使用 `prompt.md` 内置的默认平台地址；
  如需覆盖，追加 `-e BENCHMARK_BASE_URL="..."`；
- 本地模式使用公网 HTTPS 网关，不依赖原内部 CA 证书；
- 未设置 `AUTH_TOKEN` 时，`init.sh` 会打印错误并以退出码 1 结束，不会
  启动 Agent。

**查看本地容器日志**

```bash
docker logs -f tsecbench-container
docker exec -it tsecbench-container bash
ls -lt /var/log/claude/
tail -f /var/log/claude/claude_tsecbench_*.log
```

#### 2.3.2 托管模式

托管模式由平台启动容器，通常只需配置以下变量：

| 变量 | 值 |
| --- | --- |
| `BASE_URL` | `http://api.deepseek.com.tsecbench.gw/anthropic` |
| `AUTH_TOKEN` | `sk-***` |
| `MODEL` | `deepseek-flash[1m]` |

等价的变量配置：

```bash
BASE_URL="http://api.deepseek.com.tsecbench.gw/anthropic"
AUTH_TOKEN="sk-***"
MODEL="deepseek-flash[1m]"
```

说明：

- 托管模式使用平台内网网关（HTTP），无需本地 VPN；
- `AUTH_TOKEN` 为占位符，请替换为平台分配的真实 Token；
- 镜像 `init.sh` 的内置默认模型为 `deepseek-v4-flash[1m]`；托管模式建议
  显式配置 `MODEL="deepseek-flash[1m]"`（以平台实际支持的模型名为准）；
- `BENCHMARK_TOKEN` / `BENCHMARK_BASE_URL` 通常由平台注入；若平台未
  自动注入，请按平台要求补充；
- 容器生命周期、网络与靶场映射由平台负责，无需在本地执行 `docker run`；
  日志请在平台控制台查看。

### 2.4 最小验证

```bash
# 检查 Claude Code 与工具链
docker run --rm --entrypoint /bin/bash kali-tsecbench:1.5 -c \
  'claude --version && node --version && python3 --version && nmap --version | head -1'

# 预期：未注入 AUTH_TOKEN 时打印错误并以退出码 1 结束
docker run --rm --network none kali-tsecbench:1.5
```

### 2.5 交互模式（只使用 Kali 工具链，不启动 Agent）

```bash
docker run --rm -it --entrypoint /bin/bash kali-tsecbench:1.5
```

该模式跳过 `/root/init.sh`，适合手工调试工具、字典与 PoC。

### 2.6 使用自定义任务提示词

`init.sh` 固定读取 `/root/workspace/tsecbench/prompt.md`。可用单文件挂载
替换为自定义提示词（不会遮挡同目录下的模板文件）：

```bash
-v /host/path/my-prompt.md:/root/workspace/tsecbench/prompt.md:ro
```

若继续使用内置的组长/组员模板与方法论库，自定义提示词应保持与
`prompt.md` 相同的角色、汇报与纪律契约。

### 2.7 VPN 说明

- **本地模式（推荐）**：先在宿主机连接 VPN，再以 `--network host` 启动
  容器；容器直接复用宿主机路由，无需在容器内再次连接。
- **容器内 VPN（可选）**：若不希望使用宿主机 VPN，可将 `*.ovpn` 文件
  放入工作目录，`prompt.md` 会自动启动 OpenVPN；此时需要：

```bash
--cap-add=NET_ADMIN --device /dev/net/tun \
-v /host/path/client.ovpn:/root/workspace/tsecbench/client.ovpn:ro
```

  容器内 VPN 与 `--network host` 建议二选一，避免路由冲突。
- **托管模式**：网络由平台负责，无需本地 VPN。

### 2.8 持久化工作区（可选）

默认工作区随容器删除而丢失。使用命名卷可保留题目作业状态：

```bash
-v tsecbench_work:/root/workspace/tsecbench
```

> **注意**
>
> 命名卷首次创建时会从镜像复制初始内容；此后镜像内 `prompt.md` 等
> 文件的更新不会自动同步到已有卷。需要最新内置提示词时请新建卷。

### 2.9 停止与清理

```bash
docker stop tsecbench-container
docker rm tsecbench-container
docker volume rm tsecbench_work        # 如使用了命名卷
```

容器内 `/tmp/claude_env.sh` 保存了模型网关环境变量，删除容器后即消失。

---

## 3. 架构

### 3.1 镜像分层

```text
+-------------------------------------------------------------+
| 第 4 层  任务与协作层                                        |
|   prompt.md / leader_template.md / member_template.md        |
|   prompt-core.md（方法论条款库）                              |
|   monitor.md（守护 + 观察/演进）                              |
+-------------------------------------------------------------+
| 第 3 层  Agent 运行时                                        |
|   Claude Code CLI（npm 全局安装）                             |
|   Node.js / Python 依赖（pwntools、impacket、scapy 等）       |
+-------------------------------------------------------------+
| 第 2 层  安全资源                                            |
|   SecLists / rockyou / 中文与 OA 场景扩展字典                 |
|   exploitdb（290 MB）/ React2Shell PoC 模板                   |
+-------------------------------------------------------------+
| 第 1 层  Kali Rolling 基础系统与工具链                        |
|   nmap、sqlmap、hydra、hashcat、radare2、gdb、ffuf 等         |
+-------------------------------------------------------------+
```

> 当前发布镜像是**单层扁平化镜像**：上述逻辑分层描述的是内容组织，
> 不是 `docker history` 中的实际层数。

### 3.2 运行时进程模型

```text
/root/init.sh
    |
    +--> 主控 Claude Code（prompt.md，工作目录 /root/workspace/tsecbench）
    |        |
    |        | SendMessage / Agent
    |        +--> 组长 <unique_code>-leader（每题 1 名，最多 3 组并发）
    |        |        |
    |        |        +--> 组员 <unique_code>-m1
    |        |        +--> 组员 <unique_code>-m2
    |        |        +--> 组员 <unique_code>-m3（最多 3 名）
    |        |
    |        +--> 评测平台 API
    |                GET  /openapi/v1/challenges
    |                POST /openapi/v1/challenges/start
    |                POST /openapi/v1/challenges/submit
    |                POST /openapi/v1/challenges/close
    |
    +--> 监控 Claude Code（monitor.md，工作目录 /root/workspace/monitor）
             |
             +--> watchdog.sh（常驻，每 30 秒检查主控 PID）
                      |
                      +--> 主控死亡时自动重新拉起
```

- 主控只负责调度、巡查、看板与汇报，不亲自攻击靶场；
- 组长负责单题决策、分派与执行；组员执行具体侦察/利用任务；
- 组内扫描/爆破并发上限 20 线程，登录爆破限频 1-2 次/秒；
- 平台进度以 `GET /challenges` 为唯一权威，本地文件只承载作业状态。

### 3.3 主控执行流程

| 步骤 | 动作 |
| --- | --- |
| Step 1 | 解析凭证与环境变量，判定网络模式（VPN / 直连 / 端口映射） |
| Step 2 | `GET /challenges` 拉取权威进度，并按题面做题型预分类 |
| Step 3 | 按“五队列调度模型”计算选题顺序 |
| Step 4 | `POST start` 启动靶场容器，取得 `container_addr` |
| Step 5 | 同一条消息内并发拉起若干“1 组长 + 1~3 组员”的 Agent 组 |
| Step 6 | 每 2 分钟巡查：Agent 活性、运行时长、平台进度、落盘新鲜度；超时即停组、close 并补位；组崩溃则重建或续传 |
| Step 7 | 全部通关或平台任务结束时输出总报告 |

### 3.4 五队列调度模型

未通关题按尝试次数 `n` 进入 q0~q4 五个队列：

| 队列 | 含义 | 单题时限（web / 逆向·协议 / 全链·多 flag） |
| --- | --- | --- |
| q0 | 首次尝试 | 5 分钟 / 15 分钟 / 30 分钟 |
| q1 | 失败 1 次 | 5 分钟 / 15 分钟 / 30 分钟 |
| q2 | 失败 2 次 | 5 分钟 / 15 分钟 / 30 分钟 |
| q3 | 失败 3 次 | 1 小时（启用复盘模式） |
| q4 | 失败 4 次以上 | 不限时（仍受槽位轮转与让位约束） |

- 低队列未清空不得进入高队列；
- 平台 hint 接口全程禁用，信息源限定为题面、现场侦察、公开资料与
  内置字典；
- 多 flag 题按 flag 逐个提取、逐个提交，已获得的原语优先复用。

### 3.5 状态持久化

每道题在 `<work_dir>/attack_<unique_code>/` 下维护：

| 文件 | 作用 |
| --- | --- |
| `problem-info.md` | 题号、题面原文、难度、分值、flag 数、靶场地址 |
| `recon-info.md` | 接口清单、静态资产、枚举结果、指纹结论 |
| `task-table.md` | 任务表（编号/成功率/执行者/状态/反馈文件） |
| `feedback/T-xx.md` | 每项任务的原始证据与复现步骤 |

主控另将调度看板实时写入 `board.json`，用于会话中断后的交接恢复。
跨会话、跨容器重建、跨主控交接均以上述落盘文件为准。

### 3.6 监控与自愈

`monitor.md` 定义两层守护：

- **保活**：monitor Agent 部署 `watchdog.sh`；watchdog 每 30 秒检查
  `/tmp/claude.pid`，主控不在则加载 `/tmp/claude_env.sh` 后重新拉起；
  monitor 自身每约 2 分钟检查 watchdog 是否存活；
- **观察/演进**：记录异常与重启事件，在严格护栏下允许“只追加、先备份”
  地更新 `prompt.md`，把运行经验沉淀为后续规则。

关键 PID 与日志：

| 路径 | 作用 |
| --- | --- |
| `/tmp/claude.pid` | 主控 PID |
| `/tmp/claude_monitor.pid` | 监控 Agent PID |
| `/tmp/watchdog.pid` | watchdog PID |
| `/tmp/claude_env.sh` | 重启时加载的模型网关环境变量 |
| `/var/log/claude/claude_tsecbench_*.log` | 主控会话日志 |
| `/var/log/claude/claude_monitor_*.log` | 监控会话日志 |
| `/var/log/claude/monitor.log` | watchdog 运行日志 |
| `/root/.claude/projects/-root-workspace-tsecbench/` | Claude Code 会话记录 |

### 3.7 部署模式与网络

| 部署模式 | 网络方式 | 说明 |
| --- | --- | --- |
| 本地模式 | 宿主机 VPN + `--network host` | 容器共享宿主机网络命名空间，直接访问 VPN 内靶场 |
| 托管模式 | 平台网络（直连/端口映射） | 由平台启动容器并注入网络、靶场映射与评测凭证 |
| 可选：容器内 VPN | 工作目录存在 `*.ovpn` | `prompt.md` 自动启动 OpenVPN；需 `NET_ADMIN` 与 `/dev/net/tun` |

本地模式不需要在容器内建立 VPN；容器内 VPN 与 `--network host` 建议
二选一。容器默认没有 `CAP_NET_ADMIN`，不能自行建立路由或隧道。

---

## 4. 目录结构

```text
/root/
├── init.sh                         容器入口：环境变量、拉起主控与监控、常驻
├── .npmrc                          npm 镜像配置
└── workspace/
    ├── tsecbench/                  主控工作目录
    │   ├── prompt.md               主控执行指令（唯一权威，约 469 行）
    │   ├── prompt-core.md          方法论条款库（按攻击面/卡点查阅）
    │   ├── leader_template.md      组长角色卡与完整职责
    │   ├── member_template.md      组员角色卡与汇报格式
    │   └── tools/
    │       └── pocs/               离线 PoC 模板
    │           ├── react2shell_submitted_poc.js
    │           └── react2shell_README.md
    └── monitor/
        ├── monitor.md              守护与自动演进指令
        └── watchdog.sh             运行时由监控 Agent 生成
```

运行时还会生成：

```text
<work_dir>/attack_<unique_code>/   单题作业区（见 3.5）
<work_dir>/board.json              调度看板
```

---

## 5. 环境变量

| 变量 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `BASE_URL` | 建议必填 | 镜像内置默认值 | Anthropic 兼容 API 根地址 |
| `AUTH_TOKEN` | 是 | 无 | 模型网关 API Token；未设置时启动失败 |
| `MODEL` | 否 | `deepseek-v4-flash[1m]`（镜像内置） | 模型 ID（主控/子 Agent 共用），以网关实际支持为准 |
| `AUTOCOMPACT_PCT` | 否 | `80` | 上下文自动压缩触发百分比 |
| `BENCHMARK_BASE_URL` | Benchmark 模式可选 | `prompt.md` 内置默认值 | 评测平台地址；本地模式可省略 |
| `BENCHMARK_TOKEN` | Benchmark 模式必填 | 无 | 平台鉴权（请求头） |
| `TZ` | 否 | `Asia/Shanghai` | 容器时区 |

两种模式的典型配置：

| 变量 | 本地模式 | 托管模式 |
| --- | --- | --- |
| `BASE_URL` | `https://api.deepseek.com/anthropic` | `http://api.deepseek.com.tsecbench.gw/anthropic` |
| `AUTH_TOKEN` | `sk-***`（自备） | `sk-***`（平台分配） |
| `MODEL` | `deepseek-flash[1m]` | `deepseek-flash[1m]` |
| `BENCHMARK_BASE_URL` | 可省略，使用内置默认值 | 通常由平台注入 |
| `BENCHMARK_TOKEN` | 手动传入 | 通常由平台注入 |

`init.sh` 会把下列变量导出给 Claude Code：

```text
ANTHROPIC_BASE_URL / ANTHROPIC_AUTH_TOKEN / ANTHROPIC_MODEL
ANTHROPIC_DEFAULT_OPUS_MODEL / ANTHROPIC_DEFAULT_SONNET_MODEL
ANTHROPIC_DEFAULT_HAIKU_MODEL / CLAUDE_CODE_SUBAGENT_MODEL
CLAUDE_CODE_EFFORT_LEVEL=max
CLAUDE_CODE_ATTRIBUTION_HEADER=0
CLAUDE_AUTOCOMPACT_PCT
```

同时以 `IS_SANDBOX=1`、`--dangerously-skip-permissions` 启动 Agent，
使其能在容器内无交互执行命令。该模式仅适用于隔离容器，详见
[第 8 节](#8-安全与合规)。

> **注意**
>
> `BENCHMARK_TOKEN` 不再有内置回退值。未设置时，`prompt.md` 要求主控
> 在 Step 1 立即停止并报告用户检查环境变量。

---

## 6. 主要特点

### 6.1 完整 Kali 工具链

| 类别 | 工具 |
| --- | --- |
| 渗透测试 | nmap、masscan、sqlmap、nikto、whatweb、ffuf、gobuster、dirb、hydra、medusa、john、hashcat、crunch、netcat、socat、proxychains4、sshpass、openvpn、tcpdump、tshark |
| 逆向 / 二进制 | gdb、radare2、ltrace、strace、binutils、yara、binwalk、file、xxd |
| 取证 / 杂项 | foremost、exiftool、imagemagick、lftp、ftp、telnet、tftp |
| 开发 | build-essential、cmake、ninja、git、Python 3、Node.js、PHP |
| 漏洞库 | exploitdb（含 searchsploit，约 290 MB） |

### 6.2 Python 漏洞利用库

`requests`、`httpx`、`pwntools`、`pycryptodome`、`gmpy2`、`sympy`、
`pyyaml`、`pyjwt`、`dnspython`、`scapy`、`flask-unsign`、`impacket`、
`beautifulsoup4`

### 6.3 中文与场景化字典

在 SecLists / rockyou 基础上，镜像额外内置了面向国内企业环境的字典：

| 文件 | 行数 |
| --- | --- |
| `/usr/share/seclists/Passwords/Common-Credentials/Language-Specific/zh-pinyin-derived-passwords.txt` | 3066 |
| `/usr/share/seclists/Usernames/zh-pinyin-usernames.txt` | 72 |
| `/usr/share/seclists/Passwords/Common-Credentials/password-variants.txt` | 4509 |
| `/usr/share/seclists/Passwords/Keyboard-Walks/keyboard_patterns.txt` | 308 |
| `/usr/share/seclists/Passwords/Common-Credentials/cn-oa-default-creds.txt` | 126 |
| `/usr/share/seclists/Passwords/Common-Credentials/db-service-passwords.txt` | 76 |
| `/usr/share/seclists/Passwords/Common-Credentials/framework-secret-keys.txt` | 60 |
| `/usr/share/seclists/Usernames/service-usernames.txt` | 74 |
| `/usr/share/wordlists/derive-wordlist.sh` | 生成器 |

覆盖拼音用户名×派生口令、国产 OA 默认凭据、数据库服务口令、JWT/框架
密钥、键盘行走序列、大小写/符号/年份变体，以及按公司/产品词根生成
中缀组合口令的脚本。

### 6.4 离线漏洞与 PoC 资源

- exploitdb 全量库，可直接 `searchsploit` 检索；
- 内置 React2Shell（CVE-2025-55182）公开 PoC 模板与说明，离线可用；
- 构建时通过国内镜像源安装依赖（清华 APT、阿里云 PyPI、npmmirror），
  适合国内网络环境构建与部署。

### 6.5 多智能体协作方法论

- `prompt.md`：主控的完整执行契约，覆盖凭证、网络、调度、超时、复盘、
  巡查、汇报、落盘与硬性纪律；
- `leader_template.md` / `member_template.md`：组长与组员的角色卡，
  包含职责边界、汇报格式、证据要求与协作规范；
- `prompt-core.md`：按攻击面组织的方法论条款库，供组长在遇到具体
  场景或卡点时查阅；
- 强调“证据链”“表驱动派发”“失败复盘”“原语复用”“方向差分”，
  把 Agent 的自由发挥约束在可审计的流程内。

### 6.6 长时运行可靠性

- 主控每 2 分钟巡查并保活，10 分钟无工具调用会被平台回收，因此
  提示词强制心跳；
- watchdog 30 秒级拉起死亡的主控；
- 看板与单题作业区实时落盘，会话中断后可恢复；
- 超时、停滞、组员闲置、组长单线程等都有对应的干预与让位规则；
- 支持容器重建后的重指纹、旧地址清理与原语复活。

### 6.7 评测平台集成

- 自动发现未完成题目并启动靶场；
- 提交 flag、识别 duplicate、按 `correct_flag_count` 判定通关；
- 完成或放弃后 close 容器，释放平台并发名额；
- 平台 API 错误码（404/409/422/500/503）有固定处置表。

### 6.8 黑盒与合规约束

- 禁止调用平台 hint 接口，禁止注入既往题目的具体解法与答案；
- 只允许攻击本题容器，禁止跨题使用数据或提交 flag；
- 允许现场侦察与公开信息检索，不允许依赖历史答题记忆；
- 所有重要结论必须附原始响应、状态码或输出片段。

### 6.9 已知限制

- 需要 Anthropic Messages API 兼容网关，模型能力直接影响评测效果；
- 本地模式依赖宿主机 VPN 与 `--network host`，不适合多租户或共享主机；
- 未安装 Metasploit（`msfconsole`），模板中的相关检索逻辑为条件执行，
  缺少时不影响其他流程；
- 公开镜像为扁平化单层镜像，`docker history` 仅显示导入记录，
  不保留逐层构建历史；
- 内部 CA 已删除，依赖该 CA 的 HTTPS 网关需要自行提供证书；
- 镜像体积较大（约 5.56 GB），首次拉取与推送耗时较长；
- Claude Code CLI 为第三方组件，使用与再分发须遵守其许可条款。

---

## 7. 日志与排障

| 现象 | 排查方向 |
| --- | --- |
| 容器启动后没有 Agent 输出 | `docker logs`；检查 `BASE_URL` / `AUTH_TOKEN` / `MODEL` |
| 模型请求 401/403 | Token 无效、网关地址错误或模型名不被支持 |
| 模型请求 402/余额不足 | 网关账户余额或配额 |
| 平台 API 404 `task_not_found` | `BENCHMARK_TOKEN` 缺失或无效 |
| 平台 API 409 `max active` | 活跃容器达上限，先 close 一题再 start |
| 平台 API 409 `duplicate` | 该 flag 已提交过，跳过，不要重试 |
| 主控进程反复退出 | 查看 `claude_tsecbench_*.log` 与 `monitor.log` |
| monitor 未拉起主控 | 检查 `/tmp/watchdog.pid` 与 `monitor.log` |
| 本地模式靶场不可达 | 确认宿主机 VPN 已连接、容器使用 `--network host`、VPN 路由正确 |
| 托管模式网关不可达 | 确认平台已注入 `BASE_URL` / `BENCHMARK_*` 且平台网络策略允许访问 |
| 端口全 OPEN 但无 banner | 可能是出口假象，需换协议/通道复核 |
| 容器被平台重建 | 重新 start，旧地址作废，重新指纹并重植资产 |
| 磁盘空间不足 | 镜像 5.6 GB；清理日志、旧容器与未使用卷 |
| HTTPS 网关证书校验失败 | 内部 CA 已删除；安装公共 CA 或挂载自有 CA |

常用命令：

```bash
docker logs --tail 200 tsecbench-container
docker exec -it tsecbench-container tail -n 100 /var/log/claude/monitor.log
docker exec -it tsecbench-container ps aux | grep -E 'claude|watchdog'
docker exec -it tsecbench-container cat /tmp/claude.pid /tmp/claude_monitor.pid
docker stats tsecbench-container
```

托管模式请在平台控制台查看容器日志与状态。

---

## 8. 安全与合规

本镜像用于授权环境中的安全测试、竞赛、教学与研究。

- 仅对明确授权的靶场、CTF 平台或自有实验环境使用；未经授权攻击他人
  系统可能违法。
- 容器内以 root 运行，Agent 通过 `--dangerously-skip-permissions` 获得
  完全命令执行权限。不要把宿主机敏感目录、Docker socket 或生产凭据
  挂载进容器。
- 建议在专用虚拟机或隔离网络中运行，并为容器设置独立的出口网络策略。
- 容器日志与 Claude Code 会话记录可能包含目标凭据、flag、内网地址等
  敏感信息，请按敏感数据管理并定期清理。
- 不要把模型 Token、平台 Token 写入镜像或提交到仓库；运行时通过环境
  变量或密钥管理系统注入，并定期轮换。
- 平台 Token 以请求头形式传递，避免出现在命令行历史、日志或截图中。
- 靶场容器可能因高频爆破而崩溃或被封禁；提示词中已内置并发与限频纪律，
  使用者不应自行提高并发。
- 分发本镜像前，请确认所捆绑第三方组件与 CLI 的再分发条款。

---

## 9. 测试结果

第三届智能攻防挑战赛
<img width="1275" height="582" alt="image" src="https://github.com/user-attachments/assets/95744473-07cd-4719-b954-a34be46c9a95" />
<img width="1245" height="722" alt="image" src="https://github.com/user-attachments/assets/2be4403d-e82e-4a35-bb04-ee14bd3e4823" />


Tsecbench v1
<img width="1157" height="503" alt="image" src="https://github.com/user-attachments/assets/4a99c19f-26e2-4bf0-a651-e01d02f1d9b7" />


---

<p align="center"><sub>End of README</sub></p>
