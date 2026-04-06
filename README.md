# clash-mrs-builder

自动将 [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Clash) 的 Clash 规则转换为 mihomo（Clash Meta）高性能二进制 **MRS 格式**，通过 GitHub Actions 定时构建，发布到 `mrs` 分支，供 clash-verge-rev 直接订阅使用。

---

## 目录

- [背景与原理](#背景与原理)
- [文件结构](#文件结构)
- [快速上手](#快速上手)
- [配置文件说明](#配置文件说明)
- [behavior 模式详解](#behavior-模式详解)
- [本地 clash-verge-rev 配置](#本地-clash-verge-rev-配置)
- [白名单模式完整示例](#白名单模式完整示例)
- [触发构建](#触发构建)
- [常见问题](#常见问题)

---

## 背景与原理

### 为什么用 MRS 而不是 YAML？

| 格式 | 加载速度 | 内存占用 | 可读性 |
|------|---------|---------|-------|
| YAML（rule-providers） | 慢（文本解析） | 高 | ✅ 可读 |
| MRS（二进制） | **快**（内存映射） | **低** | ❌ 二进制 |

MRS（Mihomo Rule Set）是 mihomo 专属二进制格式，通过内存映射直接加载，启动更快、内存更省，适合规则量大的场景。

### 为什么不直接用 rule-providers 引用 blackmatrix7？

blackmatrix7 的 `.yaml` 文件是**混合格式**（Classical），同时包含 `DOMAIN-SUFFIX`、`IP-CIDR`、`PROCESS-NAME` 等多种规则，无法直接作为 `behavior: domain` 的 rule-providers 使用，且 classical behavior 不支持转 MRS。

本项目的构建脚本会自动**解析并拆分**，按需生成纯 domain 或纯 ipcidr 的 MRS 文件。

### blackmatrix7 文件命名说明

> 经实际核实：该仓库 **大多数服务没有 `_Domain.yaml`**，只有一个混合格式的 `RuleName.yaml`。本项目构建脚本负责自动过滤和拆分，无需手动处理。

---

## 文件结构

```
your-repo/
├── ruleset-config.yml          ← ✏️ 唯一需要修改的配置文件
└── .github/
    └── workflows/
        └── build-mrs.yml       ← GitHub Actions 构建脚本（通常无需改动）
```

构建完成后自动生成 `mrs` 分支：

```
mrs 分支/
├── Google-domain.mrs           ← split 模式：域名部分
├── Google-ip.mrs               ← split 模式：IP 部分
├── OpenAI.mrs                  ← domain 模式：仅域名
├── Telegram-domain.mrs
├── Telegram-ip.mrs
├── ...
├── manifest.json               ← 所有文件清单（含下载链接）
└── rule-providers-snippet.yaml ← 可直接使用的 mihomo 配置片段
```

---

## 快速上手

### 1. Fork 或创建仓库

在 GitHub 新建一个**公开仓库**（私有仓库需要 Token 授权才能作为 rule-providers 的 HTTP 源），将本项目的两个文件推送上去：

```
ruleset-config.yml
.github/workflows/build-mrs.yml
```

### 2. 启用 Actions 写权限

进入仓库 → **Settings → Actions → General → Workflow permissions**  
选择 **Read and write permissions** → Save

### 3. 修改配置文件

编辑 `ruleset-config.yml`，按需增减规则集（详见下节），Push 到 `main` 分支即可触发首次构建。

### 4. 手动触发测试

进入仓库 → **Actions → Build MRS Rulesets → Run workflow**

### 5. 使用生成的配置片段

构建完成后，查看 `mrs` 分支中的 `rule-providers-snippet.yaml`，将其内容合并到本地 clash-verge-rev 配置即可。

---

## 配置文件说明

`ruleset-config.yml` 结构：

```yaml
# 同步计划（需同步修改 build-mrs.yml 中的 cron）
schedule:
  daily:   false   # 每天构建  → cron: "0 22 * * *"
  weekly:  true    # 每周构建  → cron: "0 22 * * 1"（默认）
  monthly: false   # 每月构建  → cron: "0 22 1 * *"

rulesets:
  - name: OpenAI          # MRS 文件名（不含扩展名）
    behavior: domain      # 转换模式：domain / ipcidr / split
    url: "https://raw.githubusercontent.com/..."  # 源文件 URL
```

### 修改同步周期

> ⚠️ GitHub Actions 的 cron 无法动态读取配置文件，需要**同时修改两处**：

**`ruleset-config.yml`**（仅作文档记录）：
```yaml
schedule:
  daily: true
```

**`.github/workflows/build-mrs.yml`**（实际生效）：
```yaml
- cron: "0 22 * * *"    # 改为每天
```

---

## behavior 模式详解

### `domain` 模式

从混合 YAML 中**只提取域名规则**，生成单个 `name.mrs`。

```yaml
- name: OpenAI
  behavior: domain
  url: "https://raw.githubusercontent.com/.../OpenAI/OpenAI.yaml"
```

生成文件：`OpenAI.mrs`（behavior: domain）

**适用场景**：OpenAI、Gemini、Claude、YouTube 等几乎只有域名规则的服务。

---

### `ipcidr` 模式

从混合 YAML 中**只提取 IP 段规则**，生成单个 `name.mrs`。

```yaml
- name: TelegramIP
  behavior: ipcidr
  url: "https://raw.githubusercontent.com/.../Telegram/Telegram.yaml"
```

生成文件：`TelegramIP.mrs`（behavior: ipcidr）

**适用场景**：只需要 IP 段规则时使用（较少见）。

---

### `split` 模式（推荐用于含 IP 的服务）

从同一个混合 YAML 文件**同时拆出两类规则**，生成两个 MRS 文件：

```yaml
- name: Telegram
  behavior: split
  url: "https://raw.githubusercontent.com/.../Telegram/Telegram.yaml"
```

生成文件：
- `Telegram-domain.mrs`（behavior: domain）
- `Telegram-ip.mrs`（behavior: ipcidr）

**适用场景**：Telegram、Google、Microsoft、Lan 等同时有域名规则和 IP 段的服务。

拆分时脚本会自动：
- 去掉末尾的策略名（`,DIRECT` / `,PROXY`）
- 去掉 IP 规则的 `no-resolve` 标记（MRS 不需要，在 `rules` 中单独声明）

---

## 本地 clash-verge-rev 配置

### Step 1：添加 rule-providers

将 `mrs` 分支中 `rule-providers-snippet.yaml` 的内容复制到你的配置：

```yaml
rule-providers:
  OpenAI:
    type: http
    behavior: domain
    format: mrs
    url: "https://github.com/你的用户名/你的仓库/raw/mrs/OpenAI.mrs"
    path: ./providers/OpenAI.mrs
    interval: 86400    # 自动更新间隔（秒），86400 = 每天

  Telegram-domain:
    type: http
    behavior: domain
    format: mrs
    url: "https://github.com/你的用户名/你的仓库/raw/mrs/Telegram-domain.mrs"
    path: ./providers/Telegram-domain.mrs
    interval: 86400

  Telegram-ip:
    type: http
    behavior: ipcidr
    format: mrs
    url: "https://github.com/你的用户名/你的仓库/raw/mrs/Telegram-ip.mrs"
    path: ./providers/Telegram-ip.mrs
    interval: 86400

  # ... 其余规则集同理
```

### Step 2：配置 rules

```yaml
rules:
  # ── 局域网直连 ──
  - RULE-SET,Lan-domain,DIRECT
  - RULE-SET,Lan-ip,DIRECT,no-resolve      # IP 规则必须加 no-resolve

  # ── AI 服务 ──
  - RULE-SET,OpenAI,Proxy
  - RULE-SET,Gemini,Proxy
  - RULE-SET,Claude,Proxy
  - RULE-SET,Copilot,Proxy
  - RULE-SET,Bing,Proxy

  # ── 开发者与推送 ──
  - RULE-SET,GitHub,Proxy
  - RULE-SET,GoogleFCM,Proxy
  - RULE-SET,Telegram-domain,Proxy
  - RULE-SET,Telegram-ip,Proxy,no-resolve

  # ── 基础服务 ──
  - RULE-SET,YouTube,Proxy
  - RULE-SET,Twitter,Proxy
  - RULE-SET,Google-domain,Proxy
  - RULE-SET,Google-ip,Proxy,no-resolve
  - RULE-SET,Microsoft-domain,Proxy
  - RULE-SET,Microsoft-ip,Proxy,no-resolve

  # ── 兜底（白名单模式）──
  - MATCH,DIRECT
```

> **`no-resolve` 规则**：IP 类规则（`-ip.mrs`）必须在 `RULE-SET` 行尾加 `no-resolve`，
> 防止 mihomo 对已知 IP 段触发不必要的 DNS 解析。

---

## 白名单模式完整示例

**白名单模式**：只有明确列出的服务走代理，其余全部直连。核心是最后一条 `MATCH,DIRECT`。

```yaml
mixed-port: 7890
geodata-mode: false      # 不加载 geosite.dat/geoip.dat，完全依赖 rule-providers

dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16

  # DNS 分流（用 rule-set 替代 geosite）
  nameserver-policy:
    "rule-set:Lan-domain":
      - 223.5.5.5
      - 119.29.29.29
    "rule-set:OpenAI,rule-set:Gemini,rule-set:Claude,rule-set:Google-domain,rule-set:Telegram-domain":
      - https://dns.google/dns-query
      - https://1.1.1.1/dns-query

  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query

proxy-groups:
  - name: Proxy
    type: select
    proxies: [自动选择, 节点1, 节点2]

  - name: 自动选择
    type: url-test
    url: https://www.gstatic.com/generate_204
    interval: 300
    proxies: [节点1, 节点2]

rule-providers:
  Lan-domain:
    type: http
    behavior: domain
    format: mrs
    url: "https://github.com/YOUR_NAME/YOUR_REPO/raw/mrs/Lan-domain.mrs"
    path: ./providers/Lan-domain.mrs
    interval: 604800    # 一周更新一次

  # ... 其余 rule-providers

rules:
  - RULE-SET,Lan-domain,DIRECT
  - RULE-SET,Lan-ip,DIRECT,no-resolve
  - RULE-SET,OpenAI,Proxy
  - RULE-SET,Gemini,Proxy
  - RULE-SET,Claude,Proxy
  - RULE-SET,Anthropic,Proxy
  - RULE-SET,Copilot,Proxy
  - RULE-SET,Bing,Proxy
  - RULE-SET,GitHub,Proxy
  - RULE-SET,GoogleFCM,Proxy
  - RULE-SET,Telegram-domain,Proxy
  - RULE-SET,Telegram-ip,Proxy,no-resolve
  - RULE-SET,YouTube,Proxy
  - RULE-SET,Twitter,Proxy
  - RULE-SET,Google-domain,Proxy
  - RULE-SET,Google-ip,Proxy,no-resolve
  - RULE-SET,Microsoft-domain,Proxy
  - RULE-SET,Microsoft-ip,Proxy,no-resolve
  # 手动补充的域名
  - DOMAIN-SUFFIX,grok.com,Proxy
  - DOMAIN-SUFFIX,xai.com,Proxy
  - DOMAIN-SUFFIX,deepl.com,Proxy
  # 兜底：白名单核心
  - MATCH,DIRECT
```

---

## 触发构建

| 方式 | 说明 |
|------|------|
| **修改配置文件** | 编辑 `ruleset-config.yml` 后 Push，自动触发 |
| **手动触发** | Actions → Build MRS Rulesets → Run workflow |
| **定时触发** | 按 `build-mrs.yml` 中的 cron 配置，默认每周一 |

---

## 常见问题

### Q：clash-verge-rev 提示下载 .mrs 文件失败？

确认 GitHub 仓库是**公开仓库**，`mrs` 分支已成功建立，URL 中的用户名/仓库名正确。  
也可以在 clash-verge-rev 设置中配置代理后再拉取。

### Q：某个规则集没有生成 `-ip.mrs`？

说明该服务的 `.yaml` 文件中没有 IP-CIDR 规则，脚本会自动跳过，只生成 `-domain.mrs`，属于正常情况。

### Q：如何添加新的规则集？

在 `ruleset-config.yml` 的 `rulesets:` 下新增一条：

```yaml
- name: Netflix
  behavior: split    # 有 IP 规则用 split，纯域名用 domain
  url: "https://raw.githubusercontent.com/blackmatrix7/ios_rule_script/master/rule/Clash/Netflix/Netflix.yaml"
```

Push 后自动触发构建。

### Q：`no-resolve` 加在哪里？

加在 `rules` 中引用 IP 类规则集的行尾：

```yaml
- RULE-SET,Telegram-ip,Proxy,no-resolve    # ✅ 正确
- RULE-SET,Telegram-ip,Proxy               # ❌ 会触发多余 DNS 解析
```

**不需要**在 rule-providers 配置中声明，MRS 文件本身不含这个标记。

### Q：如何确认哪些 blackmatrix7 文件实际存在？

直接访问 `https://raw.githubusercontent.com/blackmatrix7/ios_rule_script/master/rule/Clash/服务名/服务名.yaml`，返回 200 则存在。  
注意：该仓库的文件名大小写敏感（如 `GitHub/GitHub.yaml`，不是 `github/github.yaml`）。

---

## 致谢

- [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script) — 规则数据来源
- [MetaCubeX/mihomo](https://github.com/MetaCubeX/mihomo) — `convert-ruleset` 转换工具
- [clash-verge-rev](https://github.com/clash-verge-rev/clash-verge-rev) — 本地客户端
