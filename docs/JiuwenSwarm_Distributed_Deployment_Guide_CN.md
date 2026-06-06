# JiuwenSwarm 分布式部署完整指南

## 📋 目录

- [快速概览](#快速概览)
- [前置要求](#前置要求)
- [架构理解](#架构理解)
- [网络规划](#网络规划)
- [安装步骤](#安装步骤)
- [Leader 节点配置](#leader-节点配置)
- [Teammate 节点配置](#teammate-节点配置)
- [启动流程](#启动流程)
- [验证部署](#验证部署)
- [常见问题](#常见问题)
- [故障排查](#故障排查)

---

## 快速概览

JiuwenSwarm 分布式部署是一个多机器的系统，包括：

| 角色 | 功能 | 运行机器 | 数量 |
|------|------|---------|------|
| **A2X Registry** | 服务发现和注册中心 | 任意一台（推荐独立）| 1 台 |
| **Leader 节点** | AI Agent 领导者，分发任务 | 机器 A | 1 台 |
| **Teammate 节点** | AI Agent 协作者，执行任务 | 机器 B、C、D... | N 台 |
| **数据库** | 共享任务和状态存储 | 任意一台（推荐独立）| 1 台 |

**最小部署**：4 台机器（Registry + Leader + Teammate + 数据库）或 3 台（如果 Registry 和数据库在同一机器）

---

## 前置要求

### 1. 系统要求

| 项目 | 要求 | 备注 |
|------|------|------|
| **操作系统** | Linux (Ubuntu 20.04+, CentOS 8+) | Windows/macOS 不支持 |
| **Python** | 3.11、3.12、3.13 | 使用 `python3 --version` 检查 |
| **内存** | 每节点 4GB 以上 | Leader 和 Teammate 各需 4GB 以上 |
| **磁盘** | 每节点 10GB 以上 | 用于日志和工作目录 |
| **网络** | 节点之间网络互通 | 确保防火墙开放必要端口 |

### 2. 必需的 API 密钥

在开始部署前，准备好以下内容：

- **LLM API 密钥**（OpenAI、DeepSeek、ZhiPu 等）
- **模型 API 地址**（api_base）
- **数据库连接字符串**（PostgreSQL 或 SQLite 路径）

### 3. 网络拓扑示意图

```
┌─────────────┐
│  A2X        │  端口: 8000
│  Registry   │
└─────────────┘
     ▲
     │ (连接)
     │
┌────┴────────────────────┬────────────────────┐
│                         │                    │
▼                         ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Leader      │    │ Teammate 1   │    │ Teammate 2   │
│ (机器 A)     │    │ (机器 B)     │    │ (机器 C)     │
│ 端口:18555   │    │ 端口:18600   │    │ 端口:18601   │
│ 端口:29101   │    │              │    │              │
└──────┬───────┘    └──────────────┘    └──────────────┘
       │
       └─────────────────────┬────────────────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │   PostgreSQL DB  │
                    │  (机器 D/共享)    │
                    │  端口: 5432      │
                    └──────────────────┘
```

---

## 架构理解

### Leader（领导者）

- **职责**：
  - 接收用户请求
  - 规划和分配任务
  - 从 Registry 动态获取 Teammate
  - 监督任务执行
  
- **开放端口**：
  - `18555` - 与 Teammate 通信（ZMQ direct）
  - `18556` - 发布消息（ZMQ pub）
  - `18557` - 订阅消息（ZMQ sub）
  - `29101` - 网关服务（HTTP）
  - `29100` - Web UI（HTTP）

### Teammate（协作者）

- **职责**：
  - 自动注册到 Registry
  - 等待 Leader 分配任务
  - 执行具体的 AI 任务
  - 将结果返回给 Leader
  
- **开放端口**：
  - `18600+` - 与 Leader 通信（ZMQ direct）
  - `28193` - Agent Server（HTTP）

### A2X Registry（服务发现）

- **职责**：
  - 管理 Teammate 注册和发现
  - 处理 Teammate 预留请求
  - 维护健康检查

### 共享数据库

- **职责**：
  - 存储任务信息
  - 存储成员状态
  - 存储消息日志
  - 所有节点都要能访问

---

## 网络规划

### IP 地址分配示例

假设使用 `192.168.1.0/24` 网段：

| 节点 | 主机名 | IP 地址 | 功能 |
|------|--------|---------|------|
| Registry | registry | 192.168.1.100 | 服务中心 |
| Leader | leader | 192.168.1.101 | 主控节点 |
| Teammate 1 | teammate1 | 192.168.1.102 | 工作节点 |
| Teammate 2 | teammate2 | 192.168.1.103 | 工作节点 |
| 数据库 | database | 192.168.1.104 | 数据存储 |

### 防火墙规则

```bash
# Registry 防火墙规则
sudo ufw allow 8000/tcp from any

# Leader 防火墙规则
sudo ufw allow 18555/tcp from any
sudo ufw allow 18556/tcp from any
sudo ufw allow 18557/tcp from any
sudo ufw allow 29101/tcp from any
sudo ufw allow 29100/tcp from any

# Teammate 防火墙规则
sudo ufw allow 18600:18700/tcp from any  # 给多个 Teammate 留出端口范围
sudo ufw allow 28193/tcp from any

# 数据库防火墙规则
sudo ufw allow 5432/tcp from 192.168.1.0/24
```

---

## 安装步骤

### 步骤 1：在所有节点安装依赖

#### 1.1 更新系统

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

#### 1.2 安装 Python 和基础工具

```bash
# Ubuntu
sudo apt-get install -y python3.11 python3-pip python3-venv git curl wget

# 检查 Python 版本
python3 --version  # 应该是 3.11+
```

#### 1.3 安装 PostgreSQL 驱动程序（所有节点需要）

```bash
sudo apt-get install -y postgresql-client libpq-dev
```

### 步骤 2：在数据库节点部署 PostgreSQL

#### 2.1 安装 PostgreSQL

```bash
sudo apt-get install -y postgresql postgresql-contrib

# 启动 PostgreSQL 服务
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

#### 2.2 创建数据库和用户

```bash
sudo -u postgres psql <<EOF
CREATE USER jiuwen WITH PASSWORD 'your_secure_password_here';
CREATE DATABASE jiuwen_team OWNER jiuwen;
GRANT ALL PRIVILEGES ON DATABASE jiuwen_team TO jiuwen;
\c jiuwen_team
GRANT ALL PRIVILEGES ON SCHEMA public TO jiuwen;
EOF
```

#### 2.3 配置 PostgreSQL 允许远程连接

编辑 `/etc/postgresql/*/main/postgresql.conf`（将 `*` 替换为版本号，如 `14`）：

```bash
sudo nano /etc/postgresql/14/main/postgresql.conf
```

找到并修改这一行（或在末尾添加）：

```
listen_addresses = '0.0.0.0'
```

编辑 `/etc/postgresql/14/main/pg_hba.conf`：

```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
```

在文件末尾添加：

```
# 允许局域网访问
host    jiuwen_team     jiuwen      192.168.1.0/24      md5
```

重启 PostgreSQL：

```bash
sudo systemctl restart postgresql
```

#### 2.4 测试连接

在其他节点上执行：

```bash
psql -h 192.168.1.104 -U jiuwen -d jiuwen_team -c "SELECT 1"
# 输入密码后应该看到 1
```

### 步骤 3：在所有 JiuwenSwarm 节点安装 JiuwenSwarm

```bash
# 使用 pip 安装（推荐）
pip install --upgrade pip
pip install jiuwenswarm

# 验证安装
jiuwenswarm-init --help
```

或者从源代码安装：

```bash
git clone https://github.com/openJiuwen-ai/jiuwenswarm.git
cd jiuwenswarm

# 创建虚拟环境
python3 -m venv venv
source venv/bin/activate

# 安装依赖
pip install -e ".[all]"
```

### 步骤 4：安装 A2X Registry

在 Registry 节点上执行：

```bash
git clone -b feature/Agentregistry https://gitcode.com/openJiuwen/agent-protocol.git
cd agent-protocol
pip install -e .

# 验证安装
a2x-registry --help
```

---

## Leader 节点配置

### 1. 创建配置目录

```bash
# 在 Leader 机器上
mkdir -p ~/.jiuwenswarm/config
cd ~/.jiuwenswarm/config
```

### 2. 获取模板配置

```bash
# 从 JiuwenSwarm 源代码目录复制 Leader 模板
# 或者直接创建以下配置文件
```

### 3. 创建 Leader 配置文件

**文件位置**: `~/.jiuwenswarm/config/config.yaml`

```yaml
# ============================================
# JiuwenSwarm Leader 节点配置
# 机器: 192.168.1.101
# ============================================

# --- 基础模型配置（必需）---
api_base: "https://api.openai.com/v1"          # 改为你的模型 API 地址
api_key: "${LLM_API_KEY}"                      # 使用环境变量或直接填入 API 密钥
model: "gpt-4o"                                # 改为你的模型名称
model_provider: "OpenAI"                       # 改为你的模型提供商

# --- 分布式 Team 配置（核心） ---
team:
  # 运行时模式
  runtime:
    mode: distributed              # 启用分布式模式
    role: leader                   # 当前节点角色为 Leader
    member_name: "leader_node"     # Leader 的标识名
  
  # 传输协议配置
  transport:
    type: pyzmq                    # 使用 ZMQ 通信
    params:
      # 重要：使用 0.0.0.0 而非 127.0.0.1（允许远程连接）
      leader:
        direct_addr: "tcp://0.0.0.0:18555"              # Leader 直连地址
        pub_addr: "tcp://0.0.0.0:18556"                 # Leader 发布地址
        sub_addr: "tcp://0.0.0.0:18557"                 # Leader 订阅地址
      pubsub_bind: true                                  # Leader 绑定 PubSub

  # 共享存储配置
  storage:
    type: postgresql               # 使用 PostgreSQL
    params:
      connection_string: "postgresql+asyncpg://jiuwen:your_secure_password_here@192.168.1.104:5432/jiuwen_team"

  # Team 工作空间配置
  workspace:
    enabled: true
    root_path: "/tmp/jiuwenswarm/shared_workspace/jiuwen_team"
    version_control: false

# --- A2X Registry 配置（关键！） ---
react:
  a2x_registry:
    base_url: "http://192.168.1.100:8000"      # Registry 地址
    dataset: "jiuwen_team_dataset"             # 数据集名称

# --- 模式配置 ---
modes:
  team:
    jiuwen_team:
      transport:
        type: pyzmq
        params:
          leader:
            direct_addr: "tcp://0.0.0.0:18555"
            pub_addr: "tcp://0.0.0.0:18556"
            sub_addr: "tcp://0.0.0.0:18557"
          pubsub_bind: true

# --- 可选：其他配置 ---
preferred_language: "zh"                       # 语言设置
heartbeat:
  every: 3600                                  # 心跳间隔（秒）

# 日志配置（调试时有用）
logging:
  level: "INFO"
```

### 4. 启动 Leader 节点

#### 4.1 设置环境变量

```bash
export HOME="/home/leader_user"                      # Leader 用户的 HOME 目录
export GIT_AUTHOR_NAME="leader"
export GIT_AUTHOR_EMAIL="leader@example.com"
export GIT_COMMITTER_NAME="leader"
export GIT_COMMITTER_EMAIL="leader@example.com"
export AGENT_SERVER_PORT=28192                       # Leader Agent Server 端口
export GATEWAY_PORT=29101                            # Leader 网关端口
export WEB_PORT=29100                                # Leader Web UI 端口
export LLM_API_KEY="your_actual_api_key_here"       # 实际 API 密钥
export JIUWENSWARM_CONFIG_DIR="$HOME/.jiuwenswarm/config"
```

#### 4.2 启动 Leader

```bash
# 方式 1：使用 pip 安装的命令
cd $HOME
jiuwenswarm-start

# 方式 2：从源代码运行
cd /path/to/jiuwenswarm
source venv/bin/activate
python -m jiuwenswarm.app

# 方式 3：在后台运行（推荐）
nohup python -m jiuwenswarm.app > leader.log 2>&1 &
```

#### 4.3 验证 Leader 启动

```bash
# 查看日志
tail -f leader.log

# 检查端口是否开放
netstat -tlnp | grep -E "(18555|18556|18557|29101|29100)"

# 测试 Leader HTTP 端口
curl -I http://localhost:29101/health
```

---

## Teammate 节点配置

### 1. 创建配置目录

```bash
# 在每个 Teammate 机器上执行
mkdir -p ~/.jiuwenswarm/config
cd ~/.jiuwenswarm/config
```

### 2. 创建 Teammate 配置文件

**文件位置**: `~/.jiuwenswarm/config/config.yaml`

```yaml
# ============================================
# JiuwenSwarm Teammate 节点配置
# 机器: 192.168.1.102（以第一个 Teammate 为例）
# ============================================

# --- 基础模型配置（必需，与 Leader 相同） ---
api_base: "https://api.openai.com/v1"         # 与 Leader 保持一致
api_key: "${LLM_API_KEY}"                     # 使用环境变量或直接填入
model: "gpt-4o"                               # 与 Leader 保持一致
model_provider: "OpenAI"                      # 与 Leader 保持一致

# --- 分布式 Team 配置（核心） ---
team:
  # 运行时模式
  runtime:
    mode: distributed              # 启用分布式模式
    role: teammate                 # 当前节点角色为 Teammate
    member_name: "teammate_1"      # Teammate 的标识名（每个 Teammate 不同）
  
  # 传输协议配置
  transport:
    type: pyzmq                    # 使用 ZMQ 通信
    params:
      # 重要：使用实际 IP 而非 127.0.0.1
      # Teammate 作为工作节点，不绑定 PubSub
      teammate:
        direct_addr: "tcp://192.168.1.102:18600"      # 本 Teammate 的直连地址
        bootstrap_direct_addr: "tcp://192.168.1.102:18600"  # 用于注册的地址
      
      # Leader 地址（Teammate 需要知道 Leader 的地址以建立连接）
      leader:
        direct_addr: "tcp://192.168.1.101:18555"      # Leader 直连地址
        pub_addr: "tcp://192.168.1.101:18556"         # Leader 发布地址
        sub_addr: "tcp://192.168.1.101:18557"         # Leader 订阅地址
      
      pubsub_bind: false                              # Teammate 不绑定

  # 共享存储配置（与 Leader 相同！非常重要）
  storage:
    type: postgresql               # 与 Leader 相同
    params:
      connection_string: "postgresql+asyncpg://jiuwen:your_secure_password_here@192.168.1.104:5432/jiuwen_team"

  # Team 工作空间配置（与 Leader 相同路径！）
  workspace:
    enabled: true
    root_path: "/tmp/jiuwenswarm/shared_workspace/jiuwen_team"
    version_control: false

# --- A2X Registry 配置（与 Leader 相同！） ---
react:
  a2x_registry:
    base_url: "http://192.168.1.100:8000"      # Registry 地址（与 Leader 相同）
    dataset: "jiuwen_team_dataset"             # 数据集名称（与 Leader 相同）
    # Teammate 启动时的注册端点
    endpoint: "tcp://192.168.1.102:18600"      # 自己的可达地址

# --- 模式配置 ---
modes:
  team:
    jiuwen_team:
      transport:
        type: pyzmq
        params:
          teammate:
            direct_addr: "tcp://192.168.1.102:18600"
          leader:
            direct_addr: "tcp://192.168.1.101:18555"
            pub_addr: "tcp://192.168.1.101:18556"
            sub_addr: "tcp://192.168.1.101:18557"
          pubsub_bind: false

# --- 可选：其他配置 ---
preferred_language: "zh"
heartbeat:
  every: 3600

logging:
  level: "INFO"
```

### 3. 多 Teammate 配置（如果有多个 Teammate）

如果你有 3 个 Teammate，为每个创建配置文件，只需改变这些值：

**Teammate 2** (`192.168.1.103`)：
```yaml
team:
  runtime:
    member_name: "teammate_2"
  transport:
    params:
      teammate:
        direct_addr: "tcp://192.168.1.103:18600"
        bootstrap_direct_addr: "tcp://192.168.1.103:18600"
```

**Teammate 3** (`192.168.1.104`)：
```yaml
team:
  runtime:
    member_name: "teammate_3"
  transport:
    params:
      teammate:
        direct_addr: "tcp://192.168.1.104:18600"
        bootstrap_direct_addr: "tcp://192.168.1.104:18600"
```

### 4. 启动 Teammate 节点

#### 4.1 设置环境变量

```bash
export HOME="/home/teammate_user"                    # Teammate 用户的 HOME
export GIT_AUTHOR_NAME="teammate"
export GIT_AUTHOR_EMAIL="teammate@example.com"
export GIT_COMMITTER_NAME="teammate"
export GIT_COMMITTER_EMAIL="teammate@example.com"
export AGENT_SERVER_PORT=28193                       # Teammate 的 Agent Server 端口
export LLM_API_KEY="your_actual_api_key_here"       # 与 Leader 相同的 API 密钥
export JIUWENSWARM_CONFIG_DIR="$HOME/.jiuwenswarm/config"
```

#### 4.2 启动 Teammate

```bash
# 方式 1：使用 pip 安装的命令
cd $HOME
python -m jiuwenswarm.server.app_agentserver

# 方式 2：从源代码运行
cd /path/to/jiuwenswarm
source venv/bin/activate
python -m jiuwenswarm.server.app_agentserver

# 方式 3：在后台运行（推荐）
nohup python -m jiuwenswarm.server.app_agentserver > teammate.log 2>&1 &
```

#### 4.3 验证 Teammate 启动

```bash
# 查看日志
tail -f teammate.log

# 应该看到类似这样的日志：
# "teammate registered ... endpoint=tcp://192.168.1.102:18600"
# "teammate registered at registry ... service_id=xxx"

# 检查 Agent Server 端口
netstat -tlnp | grep 28193

# 测试 Teammate HTTP 端口
curl -I http://localhost:28193/health
```

---

## 启动流程

### 完整启动顺序

按照以下顺序启动各个服务，确保一个成功启动再启动下一个：

#### 1️⃣ 启动 PostgreSQL 数据库

```bash
# 在数据库机器上
sudo systemctl start postgresql
sudo systemctl status postgresql

# 验证连接
psql -h 192.168.1.104 -U jiuwen -d jiuwen_team -c "SELECT 1"
```

#### 2️⃣ 启动 A2X Registry

```bash
# 在 Registry 机器上的新终端
cd /path/to/agent-protocol
a2x-registry --host 0.0.0.0

# 或在后台运行
nohup a2x-registry --host 0.0.0.0 > registry.log 2>&1 &

# 验证
curl http://192.168.1.100:8000/health

# 看到响应表示 Registry 启动成功
```

#### 3️⃣ 启动 Leader 节点

```bash
# 在 Leader 机器上的新终端
cd /path/to/jiuwenswarm
export HOME="/home/leader_user"
export AGENT_SERVER_PORT=28192
export GATEWAY_PORT=29101
export WEB_PORT=29100
export LLM_API_KEY="your_actual_api_key"

python -m jiuwenswarm.app

# 或后台运行
nohup python -m jiuwenswarm.app > leader.log 2>&1 &

# 验证（等待 30 秒左右）
sleep 30
curl -I http://192.168.1.101:29101/health
```

#### 4️⃣ 启动 Teammate 节点

**对于每个 Teammate，在其相应的机器上**：

```bash
# 在 Teammate 机器上的新终端
cd /path/to/jiuwenswarm
export HOME="/home/teammate_user"
export AGENT_SERVER_PORT=28193
export LLM_API_KEY="your_actual_api_key"

python -m jiuwenswarm.server.app_agentserver

# 或后台运行
nohup python -m jiuwenswarm.server.app_agentserver > teammate.log 2>&1 &

# 验证
sleep 10
tail -f teammate.log | grep -i "registered"
```

---

## 验证部署

### 检查清单

#### 1️⃣ 检查所有服务是否运行

```bash
# Registry 机器
curl http://192.168.1.100:8000/health
# 期望: {"status": "ok"} 或类似

# Leader 机器
curl -I http://192.168.1.101:29101/health
# 期望: HTTP/1.1 200 OK

# Teammate 机器
curl -I http://192.168.1.102:28193/health
# 期望: HTTP/1.1 200 OK
```

#### 2️⃣ 检查网络连接

```bash
# 在 Teammate 上，验证能否到达 Leader
telnet 192.168.1.101 18555
# 按 Ctrl+C 退出；应该无法立即断开说明连通

# 在 Teammate 上，验证能否到达 Registry
curl http://192.168.1.100:8000/api/datasets/jiuwen_team_dataset/agents
# 期望: JSON 响应（即使为空）
```

#### 3️⃣ 检查 Teammate 注册状态

```bash
# 在任何有 curl 的机器上
curl http://192.168.1.100:8000/api/datasets/jiuwen_team_dataset/agents | jq .

# 期望看到类似：
# {
#   "agents": [
#     {
#       "service_id": "xxx",
#       "endpoint": "tcp://192.168.1.102:18600",
#       "status": "idle"
#     }
#   ]
# }
```

#### 4️⃣ 检查数据库连接

```bash
# 在任何有 psql 的机器上
psql -h 192.168.1.104 -U jiuwen -d jiuwen_team -c "SELECT COUNT(*) FROM pg_tables WHERE schemaname='public'"

# 期望: 返回表的数量（可能为 0，说明数据库可连接）
```

### 测试分布式 Team 功能

#### 1️⃣ 打开 Web UI

在浏览器中访问：
```
http://192.168.1.101:29100
```

#### 2️⃣ 执行测试任务

在 Web UI 中发送以下消息：

```text
请进入 team 模式并完成以下步骤（按顺序，不要跳过）：
1. 调用 team.build_team 创建 Team（Leader + Teammate）
2. 调用 team.create_task 创建任务"计算 1+1"，分配给 Teammate
3. 调用 team.send_message 向 Teammate 发送消息
4. 等待 Teammate 完成任务
5. 调用 team.view_task 查看任务是否完成

输出格式：
- 步骤 1: <结果>
- 步骤 2: <结果>
- 步骤 3: <结果>
- 步骤 4: <结果>
- 步骤 5: <结果>
- 最终答案: <答案>
```

#### 3️⃣ 查看日志验证

```bash
# Leader 日志
tail -f leader.log | grep -i "team"

# Teammate 日志
tail -f teammate.log | grep -i "team"
```

---

## 常见问题

### Q1: 启动时报错 "Address already in use"

**原因**：端口被占用

**解决**：
```bash
# 查看占用端口的进程
lsof -i :18555  # 检查 Leader port
lsof -i :18600  # 检查 Teammate port

# 杀死旧进程
kill -9 <PID>

# 或修改 config.yaml 中的端口
```

### Q2: Teammate 无法注册到 Registry

**原因**：Registry 地址配置错误或网络不通

**解决**：
```bash
# 1. 检查 Registry 是否运行
curl http://192.168.1.100:8000/health

# 2. 检查 config.yaml 中的 registry base_url
grep "base_url" ~/.jiuwenswarm/config/config.yaml

# 3. 尝试从 Teammate 机器连接 Registry
curl http://192.168.1.100:8000/health
```

### Q3: Teammate 无法连接 Leader

**原因**：防火墙或网络配置

**解决**：
```bash
# 1. 检查防火墙
sudo ufw status

# 2. 打开必要的端口
sudo ufw allow 18555/tcp
sudo ufw allow 18556/tcp
sudo ufw allow 18557/tcp

# 3. 测试连接
telnet 192.168.1.101 18555
```

### Q4: 数据库连接失败

**原因**：PostgreSQL 未启动或连接字符串错误

**解决**：
```bash
# 1. 检查 PostgreSQL 是否运行
sudo systemctl status postgresql

# 2. 测试连接
psql -h 192.168.1.104 -U jiuwen -d jiuwen_team -c "SELECT 1"

# 3. 检查用户权限
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE jiuwen_team TO jiuwen"
```

### Q5: 如何添加更多 Teammate？

**步骤**：
```bash
# 1. 在新机器上安装 JiuwenSwarm
pip install jiuwenswarm

# 2. 复制配置模板并修改：
mkdir -p ~/.jiuwenswarm/config
# 编辑 config.yaml，改变 member_name 和直连地址

# 3. 启动新 Teammate
python -m jiuwenswarm.server.app_agentserver

# 4. 验证注册
curl http://192.168.1.100:8000/api/datasets/jiuwen_team_dataset/agents
```

---

## 故障排查

### 问题诊断步骤

#### Step 1: 检查日志

```bash
# Leader 日志位置
less ~/.jiuwenswarm/agent.log
less leader.log

# Teammate 日志位置
less ~/.jiuwenswarm/agent.log
less teammate.log

# 查找错误
grep -i "error\|failed\|exception" leader.log
```

#### Step 2: 网络诊断

```bash
# 检查本机监听的端口
netstat -tlnp

# 检查特定端口
ss -tlnp | grep 18555

# 测试网络连接（从 Teammate 到 Leader）
telnet 192.168.1.101 18555

# 使用 ping 测试可达性
ping -c 3 192.168.1.101
```

#### Step 3: 配置验证

```bash
# 验证配置文件语法
python3 -c "import yaml; yaml.safe_load(open('~/.jiuwenswarm/config/config.yaml'))"

# 查看有效配置
grep -A 20 "team:" ~/.jiuwenswarm/config/config.yaml
```

### 常见错误信息及解决方案

| ���误信息 | 可能原因 | 解决方案 |
|---------|---------|---------|
| `ConnectionRefusedError` | 目标服务未启动 | 检查服务是否运行，查看日志 |
| `TimeoutError` | 网络超时 | 检查防火墙，验证 IP 地址 |
| `Authentication failed` | 数据库密码错误 | 检查连接字符串中的密码 |
| `zmq.error.ZMQError` | ZMQ 端口冲突 | 更改端口或杀死占用进程 |
| `postgresql connection failed` | 数据库不可达 | 检查 PostgreSQL 是否运行并可连接 |
| `registry not reachable` | Registry 地址错误 | 验证 base_url 配置 |

### 性能调优

#### 增加日志级别（调试）

编辑 `config.yaml`：
```yaml
logging:
  level: "DEBUG"  # 改为 DEBUG 获取更详细的日志
```

#### 优化网络

```bash
# 查看网络统计
netstat -i

# 检查网络延迟
ping -c 10 192.168.1.101 | grep "min/avg/max"

# 如果延迟高，考虑优化网络或检查带宽
```

---

## 总结

### 部署完成后

✅ 所有服务都在运行
✅ Leader 和 Teammate 能互相通信
✅ Teammate 已注册到 Registry
✅ 数据库能正常访问
✅ Web UI 正常工作

### 下一步

- 参考官方文档了解更多高级功能
- 查看配置文档进行更多定制
- 根据实际需求调整资源分配和网络配置

### 获取帮助

- 查看日志文件获取详细错误信息
- 检查官方 GitHub Issues
- 参考故障排查部分

---

**祝你部署顺利！** 🚀
