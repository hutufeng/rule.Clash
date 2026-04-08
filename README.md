# clash-mrs-builder

自动将 [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Clash) 的 Clash 规则转换为 mihomo 高性能二进制 **MRS 格式**，通过 GitHub Actions 定时构建，发布到 `mrs` 分支。支持将所有规则集**合并去重**为单一产物，供 Clash Verge Rev 极简订阅使用。

---

## 目录

- [为什么用 MRS](#为什么用-mrs)
- [文件结构](#文件结构)
- [快速上手](#快速上手)
- [ruleset-config.yml 说明](#ruleset-configyml-说明)
- [behavior 模式详解](#behavior-模式详解)
- [格式转换规则](#格式转换规则)
- [ProxyAll 合并机制](#proxyall-合并机制)
- [极简配置模板](#极简配置模板)
- [触发构建](#触发构建)
- [常见问题](#常见问题)

---

## 为什么用 MRS

| 格式 | 加载速度 | 内存 | 查找复杂度 |
|------|---------|------|-----------|
| YAML classical | 慢（文本解析） | 高 | O(N) 线性扫描 |
| MRS domain | **快**（内存映射） | 低 | **O(L)** Trie 树 |
| MRS ipcidr | **快** | 低 | **O(L)** 前缀树 |

> L = 域名/IP 长度（与规则数量无关）。6000 条和 100 条的查找速度相同。

blackmatrix7 的 `.yaml` 是混合格式（DOMAIN + IP-CIDR + PROCESS-NAME 混在一起），无法直接用 `behavior: domain`，本项目脚本自动解析、拆分、转换格式。

---

## 文件结构

```
主分支/
├── ruleset-config.yml          ← ✏️ 规则集配置（唯一需要修改的文件）
├── proxy-template.yaml         ← 📋 极简配置模板（填入节点即可使用）
└── .github/workflows/
    └── build-mrs.yml           ← ⚙️  构建脚本（通常无需修改）
```

`mrs` 分支（自动生成，勿手动修改）：

```
mrs 分支/
├── Lan-domain.mrs              ← 局域网直连域名
├── Lan-ip.mrs                  ← 局域网直连 IP 段
│
├── ProxyAll-domain.mrs         ← ⭐ 所有代理域名合并去重
├── ProxyAll-ip.mrs             ← ⭐ 所有代理 IP 段合并去重
├── ProxyAll-keyword.yaml       ← ⭐ 所有 DOMAIN-KEYWORD 合并（classical YAML）
│
├── OpenAI.mrs                  ← 各服务精细 MRS（按需单独使用）
├── Google-domain.mrs
├── Google-ip.mrs
├── Telegram-domain.mrs
├── Telegram-ip.mrs
├── ...
├── manifest.json               ← 文件清单
└── rule-providers-snippet.yaml ← 精细版配置片段参考
```

---

## 快速上手

### 1. Fork 仓库

在 GitHub Fork 本仓库（务必**公开**，私有仓库无法作为 rule-providers HTTP 源）。

### 2. 启用 Actions 写权限

**Settings → Actions → General → Workflow permissions → Read and write permissions → Save**

### 3. 手动触发首次构建

**Actions → Build MRS Rulesets → Run workflow**

构建完成后 `mrs` 分支会生成所有 `.mrs` 文件。

### 4. 使用极简配置模板

将 `proxy-template.yaml` 复制到本地，填入你的节点信息，在 Clash Verge Rev 中加载即可。

---

## ruleset-config.yml 说明

```yaml
rulesets:
  - name: Lan              # 生成的文件名前缀
    behavior: split        # 转换模式：domain / ipcidr / split
    exclude_merge: true    # 不合并进 ProxyAll（Lan 直连，不参与代理全集）
    url: "https://..."     # blackmatrix7 源文件 URL

  - name: OpenAI
    behavior: domain       # 纯域名服务
    url: "https://..."     # 自动合并进 ProxyAll（无 exclude_merge）

  - name: GlobalProxy
    behavior: split        # 含 IP 规则的服务
    url: "https://..."
```

### `exclude_merge` 字段

| 设置 | 行为 |
|------|------|
| 未设置（默认） | 自动合并进 `ProxyAll-*`，配置文件无需修改 |
| `exclude_merge: true` | 不进入 ProxyAll，需在配置文件里单独引用（适合直连服务） |

**适合 `exclude_merge: true` 的情况：**
- 需要直连的服务（如 Microsoft、某些国内服务）
- 需要走特定代理组的服务（而不是统一的 Proxy 组）

### 修改构建周期

> ⚠️ cron 无法动态读取配置，需同时修改两处：

`ruleset-config.yml`（文档记录）：
```yaml
schedule:
  daily: true    # 改为每天
```

`.github/workflows/build-mrs.yml`（实际生效）：
```yaml
- cron: "0 22 * * *"    # 每天 UTC 22:00
```

---

## behavior 模式详解

### `domain` — 纯域名服务

从混合 YAML 提取所有域名规则，生成单个 `{name}.mrs`。

```yaml
- name: OpenAI
  behavior: domain
  url: "https://raw.githubusercontent.com/blackmatrix7/.../OpenAI/OpenAI.yaml"
```

生成：`OpenAI.mrs`（behavior: domain）  
适用：OpenAI、Gemini、Claude、YouTube、GitHub 等

---

### `split` — 同时含域名和 IP 的服务

从同一文件同时拆出两类，生成两个 MRS：

```yaml
- name: Telegram
  behavior: split
  url: "https://raw.githubusercontent.com/blackmatrix7/.../Telegram/Telegram.yaml"
```

生成：
- `Telegram-domain.mrs`（behavior: domain）
- `Telegram-ip.mrs`（behavior: ipcidr）

适用：Telegram、Google、Lan 等

---

### `ipcidr` — 纯 IP 场景

仅提取 IP-CIDR 规则，生成单个 `{name}.mrs`（较少用）。

---

## 格式转换规则

blackmatrix7 的混合格式经脚本自动转换：

| 原始格式 | 转换结果 | 说明 |
|---------|---------|------|
| `DOMAIN-SUFFIX,github.com` | `+.github.com` | domain MRS 标准格式 |
| `DOMAIN,api.github.com` | `api.github.com` | 精确匹配 |
| `DOMAIN-KEYWORD,github` | 收集进 `*-keyword.yaml` | domain MRS 不支持 keyword |
| `IP-CIDR,91.108.0.0/16` | `91.108.0.0/16` | 去掉前缀，纯 CIDR |
| `IP-CIDR6,2001:b28::/48` | `2001:b28::/48` | 同上 |
| `IP-CIDR,...,no-resolve` | 去掉 `,no-resolve` | 在 rules 里统一声明 |
| `PROCESS-NAME,...` | ❌ 丢弃 | MRS 不支持进程规则 |

**去重**：同一规则（带/不带 `no-resolve`）只保留一条。

---

## ProxyAll 合并机制

所有 `exclude_merge` **未设置**的规则集，构建完毕后会自动合并去重，生成三个总文件：

| 文件 | behavior | 格式 | 内容 |
|------|---------|------|------|
| `ProxyAll-domain.mrs` | domain | MRS | 所有代理域名（`+.xxx` / `xxx`） |
| `ProxyAll-ip.mrs` | ipcidr | MRS | 所有代理 IP 段 |
| `ProxyAll-keyword.yaml` | classical | YAML | 所有 `DOMAIN-KEYWORD` 规则 |

### 单独 keyword 文件

仅当规则集设置了 `exclude_merge: true` 时，才**额外**生成 `{name}-keyword.yaml`（用于直连服务的 keyword 控制）。

---

## 极简配置模板

`proxy-template.yaml` 是基于 ProxyAll 的极简配置，**只需 5 个 rule-providers、4 行 rules 逻辑**：

```yaml
rule-providers:
  Lan-domain:      # 局域网直连域名
  Lan-ip:          # 局域网直连 IP
  ProxyAll-domain: # 所有代理域名（合并去重）
  ProxyAll-ip:     # 所有代理 IP（合并去重）
  ProxyAll-keyword:# 所有代理 KEYWORD（classical yaml）

rules:
  - RULE-SET,Lan-domain,DIRECT
  - RULE-SET,Lan-ip,DIRECT,no-resolve
  - RULE-SET,ProxyAll-keyword,Proxy
  - RULE-SET,ProxyAll-domain,Proxy
  - RULE-SET,ProxyAll-ip,Proxy,no-resolve
  - MATCH,DIRECT
```

### DNS 分流

```yaml
nameserver-policy:
  "rule-set:ProxyAll-domain":
    - https://dns.google/dns-query
    - https://1.1.1.1/dns-query
```

使用 Trie 树（O(L)），6000+ 条规则的查找延迟 < 0.1ms，不影响 DNS 性能。

### 按需添加直连服务（如 Microsoft）

**Step 1** — `ruleset-config.yml` 标记不合并：
```yaml
- name: Microsoft
  behavior: domain
  exclude_merge: true   # 不进入 ProxyAll
  url: "https://..."
```

**Step 2** — Push 触发重建（Microsoft.mrs 重新生成，且不再包含在 ProxyAll 内）

**Step 3** — `proxy-template.yaml` 添加单独 provider 和 DIRECT 规则：
```yaml
# rule-providers 里加
Microsoft:
  type: http
  behavior: domain
  format: mrs
  url: "...hutufeng/rule.Clash/mrs/Microsoft.mrs"
  path: ./providers/Microsoft.mrs
  interval: 86400

# rules 里 Lan 后面加
- RULE-SET,Microsoft,DIRECT
```

这样 Microsoft 域名的 DNS 也会走国内 DNS 解析（不在 ProxyAll-domain 里），直连速度最优。

---

## 触发构建

| 方式 | 条件 |
|------|------|
| 修改配置 | Push 修改 `ruleset-config.yml` 或 `build-mrs.yml` |
| 手动触发 | Actions → Build MRS Rulesets → Run workflow |
| 定时触发 | 默认每周一 UTC 22:00（北京时间周二 06:00） |

---

## 常见问题

### Q：规则没有生效，日志显示 `match Match using DIRECT`？

规则集未加载成功。解决步骤：
1. 确认 Actions 已成功执行（`mrs` 分支有文件）
2. 在 Clash Verge Rev → 规则提供者 → **强制更新所有规则集**
3. 重载配置

### Q：某个服务的 `-ip.mrs` 不存在？

该服务 YAML 中没有 IP-CIDR 规则，脚本自动跳过，属正常现象（如 Microsoft 只有域名规则）。

### Q：如何添加新的代理服务？

在 `ruleset-config.yml` 末尾新增：
```yaml
- name: Netflix
  behavior: split    # 有 IP 用 split，纯域名用 domain
  url: "https://raw.githubusercontent.com/blackmatrix7/ios_rule_script/master/rule/Clash/Netflix/Netflix.yaml"
```
Push 后自动触发构建，新服务域名自动合并进 `ProxyAll-domain.mrs`，**无需修改 proxy-template.yaml**。

### Q：DOMAIN-KEYWORD 规则去哪了？

合并进了 `ProxyAll-keyword.yaml`（classical behavior YAML），在 proxy-template.yaml 里通过 `ProxyAll-keyword` provider 引用，会正常匹配。

### Q：`no-resolve` 写在哪里？

写在 `rules` 引用 IP 规则集时：
```yaml
- RULE-SET,ProxyAll-ip,Proxy,no-resolve    # ✅ 正确
```
不需要在 rule-providers 里声明，MRS 文件不含此标记。

### Q：GitHub API rate limit 导致 Actions 失败？

已内置 `GITHUB_TOKEN` 认证（5000次/小时），并添加备用固定版本兜底，正常情况不会再触发此错误。

---

## 致谢

- [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script) — 规则数据来源
- [MetaCubeX/mihomo](https://github.com/MetaCubeX/mihomo) — `convert-ruleset` 转换工具
- [clash-verge-rev](https://github.com/clash-verge-rev/clash-verge-rev) — 本地客户端
