# Clash / Mihomo AI-PROXY 最小修改指南（公开版）

本指南适用于使用 WgetCloud 或类似订阅服务，并通过 Clash、Mihomo、ClashX Pro 或兼容客户端进行规则分流的用户。

目标是采用 **Minimal Patch / 最小修改**：保留原订阅中已经正常工作的普通流量逻辑，只为 OpenAI / ChatGPT 与 Anthropic / Claude 增加一个独立的手动选择组 `AI-PROXY`。未来如有需要，还可以把自己的独立静态住宅 / ISP SOCKS5 作为可选出口加入该组。

本文所有节点、地址、端口和认证字段均为示例或占位符，不包含可用代理凭据。

## 1. 为什么采用最小修改

订阅服务通常已经维护了大量节点、代理组、国内直连规则、海外代理规则、DNS、fake-ip 和最终兜底逻辑。大规模重写会增加三个问题：

- 上游更新后难以重新套用；
- 很难判断故障来自原订阅还是自定义规则；
- 容易意外改变普通网站、国内应用或系统服务的路径。

因此建议只做必要差异：

1. 新增独立 `AI-PROXY`；
2. 把明确的 OpenAI / Anthropic 核心域名指向它；
3. 保持普通流量和最终 `MATCH` 的原订阅行为；
4. 用日志和真机测试补充真正需要的精确域名。

## 2. 核心架构

```text
普通流量
   ↓
保持原订阅规则和策略组

OpenAI / Anthropic
        ↓
     AI-PROXY
     ↙      ↘
普通代理节点   AI-RESIDENTIAL
                  ↓
              SOCKS5 出口
```

`AI-PROXY` 是 `select` 选择器，不是真实代理节点。它的成员才是真正的出口。

这样可以做到：

- 切换普通代理节点时，AI 出口不必跟着变化；
- 切换 `AI-PROXY` 时，普通代理流量不受影响；
- 以后增加或移除住宅出口时，不需要修改 OpenAI / Anthropic 规则。

## 3. 开始前：保护原始订阅

不要直接修改唯一一份订阅 YAML。建议：

1. 保存刚下载的原始订阅；
2. 将它作为只读基线；
3. 计算本地 SHA-256，便于确认基线未被修改；
4. 逐字节复制出工作副本；
5. 确认 YAML 能被客户端或 YAML 解析器正常读取；
6. 审计顶层 `proxies`、`proxy-groups`、`rules`、DNS、fake-ip 和最终 `MATCH`；
7. 只在工作副本中实施后续步骤。

原始订阅可能包含订阅 URL、服务器地址、端口和认证信息，不应上传到公开仓库或公开聊天。

## 4. 审计实际节点名称

先从当前 YAML 的顶层 `proxies` 中选择一个或多个适合 AI 服务的现有节点。候选数量、地区和顺序应由实际订阅与测试结果决定，不存在必须使用的国家或固定数量。

节点名称必须逐字符匹配，包括：

- Emoji；
- 空格；
- 大小写；
- 编号；
- 全角或半角符号。

不要根据旧订阅猜名称，也不要把并不存在的节点写进组内。

## 5. 新增独立 AI-PROXY

在现有 `proxy-groups` 中加入一个独立 `select` 组：

```yaml
proxy-groups:
  - name: AI-PROXY
    type: select
    proxies:
      - '<AI_NODE_1>'
      - '<AI_NODE_2>'
```

注意：

- `<AI_NODE_1>` 和 `<AI_NODE_2>` 都是占位符；
- 必须替换为你的当前订阅中真实存在的节点名称；
- 可以只选一个，也可以选择多个；
- 不要随意把订阅中的全部节点都加入；
- `AI-PROXY` 应独立于原有普通节点选择组；
- 本方案不使用 `url-test`、`fallback`、`load-balance` 或随机选择。

部分客户端会记住上一次选择，不能仅凭成员顺序推断当前出口。加载配置后，应在 GUI 中确认 `AI-PROXY` 的实际选择。

## 6. 十一个当前 AI 域名

建议至少审计以下 OpenAI / Anthropic 核心及客户端支持域名。

OpenAI / ChatGPT：

- `openai.com`
- `chatgpt.com`
- `oaistatic.com`
- `oaistatsig.com`
- `oaiusercontent.com`

Anthropic / Claude：

- `anthropic.com`
- `claude.ai`
- `claude.com`
- `claude.app`
- `claudeusercontent.com`
- `claudemcpcontent.com`

对应的 Clash 规则示意：

```yaml
rules:
  - DOMAIN-SUFFIX,openai.com,AI-PROXY
  - DOMAIN-SUFFIX,chatgpt.com,AI-PROXY
  - DOMAIN-SUFFIX,oaistatic.com,AI-PROXY
  - DOMAIN-SUFFIX,oaistatsig.com,AI-PROXY
  - DOMAIN-SUFFIX,oaiusercontent.com,AI-PROXY
  - DOMAIN-SUFFIX,anthropic.com,AI-PROXY
  - DOMAIN-SUFFIX,claude.ai,AI-PROXY
  - DOMAIN-SUFFIX,claude.com,AI-PROXY
  - DOMAIN-SUFFIX,claude.app,AI-PROXY
  - DOMAIN-SUFFIX,claudeusercontent.com,AI-PROXY
  - DOMAIN-SUFFIX,claudemcpcontent.com,AI-PROXY
```

如果上游已经存在相同的规则表达式，但目标是普通代理组，应将目标策略改为 `AI-PROXY`，不要同时保留两个策略不同的重复表达式。只有确实缺失时才新增。

这 11 条是当前建议的审计起点，不是永久完整的域名全集。服务域名和客户端行为可能变化；后续应以官方信息和实际连接日志为依据做精确补充。

## 7. first-match：规则顺序比规则数量更重要

Clash / Mihomo 通常从上到下采用首次匹配。如果原配置在核心规则之前存在更宽泛的规则，例如：

```yaml
- DOMAIN-KEYWORD,chatgpt,<NORMAL_PROXY_GROUP>
```

那么 `chatgpt.com` 可能先进入普通代理组。应把精确的 AI 核心规则放到此类宽泛规则之前。

推荐顺序：

1. 精确的 OpenAI / Anthropic AI 规则；
2. 原订阅中的明确直连或业务规则；
3. 原有普通海外代理规则；
4. GEOIP、其他现有规则；
5. 原订阅最终 `MATCH`。

具体顺序仍应以实际基线为准，不要为了套模板而大范围移动无关规则。

## 8. 不要扩大共享父域

不要因为某个 AI 页面使用了第三方服务，就把以下大型共享域整体锁定到 `AI-PROXY`：

- Cloudflare；
- AWS；
- Azure；
- Stripe；
- Sentry；
- Auth0；
- Datadog。

这些平台承载大量无关网站。若日志确认 AI 链路需要某个第三方地址，应优先使用足够精确的 `DOMAIN` 或窄范围 `DOMAIN-SUFFIX`，并重新做顺序审计。

## 9. 保持原订阅普通逻辑

除非另有明确目标，Minimal Patch 不应修改：

- 原 `proxies`；
- 原普通 `proxy-groups`；
- DNS；
- fake-ip；
- IPv6；
- mixed-port、mode、external-controller；
- 国内规则与 GEOIP；
- 普通海外代理规则；
- 最终 `MATCH` 及其目标策略。

不要把本指南理解为要求将最终规则改为 `MATCH,DIRECT`。最终兜底应保持你的原订阅设计，除非你另行评估并明确承担其行为变化。

本方法也不强制 DeepSeek 直连，不增加 Gemini 专用规则。

## 10. 可选：独立静态住宅 / ISP SOCKS5

住宅代理是可选出口，不是使用本方法的必要条件。提供商通常会给出：

- IP；
- PORT；
- USERNAME；
- PASSWORD。

具体格式、认证方式和功能应以代理服务商说明为准。

通用 Clash YAML 模板：

```yaml
proxies:
  -
    name: '🏠 AI-RESIDENTIAL'
    type: socks5
    server: YOUR_IP
    port: YOUR_PORT
    username: YOUR_USERNAME
    password: YOUR_PASSWORD
```

这只是模板。填入真实参数后，该节点定义必须符合你所用 Clash / Mihomo 内核的语法，并通过本地解析和连通性测试。

随后把节点名称加入 `AI-PROXY`：

```yaml
proxy-groups:
  - name: AI-PROXY
    type: select
    proxies:
      - '🏠 AI-RESIDENTIAL'
      - '<AI_BACKUP_NODE>'
```

建议至少保留一个经过验证的普通代理节点作为故障备用。

## 11. 住宅节点不是默认链式代理

`AI-RESIDENTIAL` 与普通订阅节点是 `AI-PROXY` 下的同级出口。

选择住宅节点时，默认路径是：

```text
App
 ↓
Clash / Mihomo
 ↓
AI-PROXY
 ↓
AI-RESIDENTIAL
 ↓
Internet
```

它不是：

```text
普通代理节点 → 住宅代理
```

如确实需要链式代理，应作为独立项目评估客户端支持、隐私、性能和故障行为；本 V1 方法不默认实现代理链。

## 12. CLI 与桌面程序的代理继承

规则按目标域名分流，而不是按“Codex CLI”或“Claude Code”等应用名称分流：

- CLI 访问 OpenAI 官方域名时，命中 OpenAI 规则；
- CLI 访问 Anthropic 官方域名时，命中 Anthropic 规则；
- CLI 访问其他 API 时，继续按该目标域名和原订阅规则处理。

部分 CLI 程序不一定自动继承 GUI 或系统代理。必要时可显式设置：

```text
HTTP_PROXY=http://127.0.0.1:<LOCAL_HTTP_PROXY_PORT>
HTTPS_PROXY=http://127.0.0.1:<LOCAL_HTTP_PROXY_PORT>
```

`<LOCAL_HTTP_PROXY_PORT>` 必须替换为客户端实际监听端口。例如某些 Clash 客户端可能使用 `7890`，但它不是所有环境的固定值。设置后应通过日志确认进程确实进入本地代理。

## 13. 新订阅后的正确复用方法

不要永久维护一份越来越旧的订阅 YAML。新订阅后应重新实施最小补丁：

1. 获取新的原始订阅；
2. 原样保存新的基线；
3. 确认 YAML 可解析并记录本地 SHA-256；
4. 重新检查真实节点名称；
5. 从新基线创建新的工作副本；
6. 在新副本中新增独立 `AI-PROXY`；
7. 审计当前 11 个 AI 域名：已有时改策略，缺失时才新增；
8. 检查 first-match，确保宽泛规则不会抢先命中；
9. 保持新基线的普通流量、DNS、fake-ip、IPv6 和最终 `MATCH`；
10. 如需要，再添加经过本地保护的住宅 SOCKS5；
11. 做结构级静态差异审计；
12. 加载配置并完成真机测试。

不要把旧订阅中的服务器、端口、密码或其他认证信息覆盖到新订阅。这是一套可重复实施的方法，不是永久复用旧 YAML。

## 14. 冲突、去重和差异审计

建议按以下规则处理：

- 相同规则表达式、策略不同：保留表达式并改为目标策略；
- 完全相同的重复规则：去重；
- 精确 `DOMAIN` 与较宽 `DOMAIN-SUFFIX`：根据实际用途保留必要规则，并审计顺序；
- AI 核心域名：不得被旧普通代理规则提前命中；
- 住宅成员引用：必须同时存在真实 proxy 定义；
- 代理组成员：必须引用当前 YAML 中真实存在的节点名称。

修改后不要只看普通文本 diff。还应比较：

- 顶层 key 是否意外变化；
- 原 `proxies` 的数量、顺序和内容；
- 原普通代理组；
- DNS、fake-ip、IPv6；
- 非 AI 规则的数量、顺序和策略；
- 最终 `MATCH`；
- 新增组、修改规则和新增规则的实际数量。

## 15. 静态验收清单

- [ ] YAML 可以正常解析；
- [ ] `AI-PROXY` 是独立的 `select` 组；
- [ ] 没有 `url-test`、`fallback`、`load-balance` 或随机逻辑；
- [ ] 所有候选均引用当前 `proxies` 中真实存在的节点；
- [ ] 当前 11 个 AI 域名的首次匹配都是 `AI-PROXY`；
- [ ] 宽泛关键词或普通代理规则不会提前命中 AI 核心域名；
- [ ] 未把大型共享父域整体加入 `AI-PROXY`；
- [ ] 原 `proxies`、普通组、DNS、fake-ip、IPv6 和普通规则未被意外修改；
- [ ] 最终 `MATCH` 保持原订阅值；
- [ ] 如启用住宅代理，proxy 定义和代理组引用同时存在；
- [ ] 不存在重复节点名或悬空引用；
- [ ] 配置、日志和报告没有泄漏真实凭据。

## 16. 真机验收清单

加载配置后，通过客户端连接日志分别检查：

- [ ] 配置正常加载，无语法错误；
- [ ] ChatGPT 核心请求进入 `AI-PROXY`；
- [ ] OpenAI API / CLI 请求在适用时进入 `AI-PROXY`；
- [ ] Claude Web 请求进入 `AI-PROXY`；
- [ ] Claude Desktop 请求在适用时进入 `AI-PROXY`；
- [ ] Claude Code 请求在适用时进入 `AI-PROXY`；
- [ ] 普通海外流量仍按原订阅代理组处理；
- [ ] 国内和直连流量仍按原订阅处理；
- [ ] 切换普通策略组不会改变 AI 出口；
- [ ] 切换 `AI-PROXY` 不会改变普通流量出口。

住宅节点启用后还应检查：

- [ ] 出口 IP；
- [ ] ASN；
- [ ] ISP；
- [ ] 地理位置；
- [ ] SOCKS5 连通性；
- [ ] 延迟；
- [ ] 丢包；
- [ ] 长时间稳定性；
- [ ] OpenAI 可访问性；
- [ ] Anthropic 可访问性。

固定、稳定的出口可以减少不必要的出口变化，但不能保证账号安全，也不能保证不会触发平台风控或限制。

## 17. 公开分享前的安全检查

不要：

- 把真实 proxy password 发到公开聊天；
- 公开上传原始订阅 YAML；
- 分享订阅 URL；
- 在公开教程中填写 SOCKS5 username / password；
- 发布未经遮挡的真实配置截图。

分享前搜索并审查：

- `server`；
- `port`；
- `password`；
- `username`；
- `uuid`；
- `token`；
- subscription URL；
- 含认证信息的代理 URI；
- 用户目录和账户标识。

字段名称可以出现在教程中，但实际值必须删除或替换为 `YOUR_*`、`<PLACEHOLDER>`。

## 18. 本 V1 方法明确不做

- 自动订阅更新；
- 自动 YAML merge；
- `build.py`；
- DNS 大范围重构；
- fake-ip 大范围重构；
- IPv6 大范围重构；
- DeepSeek 专用分流；
- Gemini 专用分流；
- 自动选择最佳 AI 节点；
- 强制所有海外流量走住宅代理；
- 默认链式代理；
- 规避平台地域限制或平台政策。

## 19. 常见踩坑

1. 把 `AI-PROXY` 当成真实节点，而不是选择器；
2. 候选名称与 `proxies` 不完全一致；
3. 新增了 AI 规则，但被更早的关键词规则抢先命中；
4. 同一表达式同时保留普通代理和 `AI-PROXY` 两条冲突规则；
5. 为了覆盖一个 AI 子域，把整个共享 CDN 强制到 AI 出口；
6. 只把住宅名称加入组，却没有有效的住宅 proxy 定义；
7. 假定 GUI 当前选择永远不会因重新加载而变化；
8. 直接在旧订阅上长期累积修改，而不是对新基线重复实施；
9. 把某个客户端名称等同于固定 API 域名；
10. 把本地代理端口 `7890` 当成所有环境的固定值。

## 20. 免责声明

本指南仅用于网络配置和流量分流说明。请遵守所在地法律法规，以及网络服务、代理服务和 AI 平台的服务条款。任何 IP 类型都不能保证避免账号限制或平台风控。服务域名、客户端行为、配置语法和服务条款可能变化，实际使用时应重新核对并测试。
