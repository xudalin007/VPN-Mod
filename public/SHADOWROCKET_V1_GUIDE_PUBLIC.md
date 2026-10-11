# Shadowrocket AI-PROXY 最小修改指南（公开版）

本指南适用于使用 WgetCloud 或类似订阅服务，并希望在 Shadowrocket 中为 OpenAI / ChatGPT 与 Anthropic / Claude 设置独立出口的用户。

核心方法是 **Minimal Patch / 最小修改**：保留原订阅中已经正常工作的 `DIRECT`、普通代理、DNS、IPv6、TUN 和最终规则，只新增一个独立的 `AI-PROXY` 选择组及必要的 AI 核心域名规则。未来如有需要，可以加入自己的独立静态住宅 / ISP SOCKS5。

本文所有节点、地址、端口和认证字段都是示例或占位符，不包含可用代理凭据。

## 1. 核心架构

```text
普通流量
   ↓
保持原订阅规则

OpenAI / Anthropic
        ↓
    AI-PROXY
        ↓
用户当前选择的真实节点
        ↓
     Internet
```

未来加入住宅节点后：

```text
OpenAI / Anthropic
        ↓
    AI-PROXY
     ↙      ↘
普通订阅节点  🏠 AI-RESIDENTIAL
                    ↓
                SOCKS5 出口
```

`AI-PROXY` 是 `select` 代理组 / 选择器，不是真实代理节点。组内成员才是真正的出口。

## 2. 为什么坚持最小修改

订阅配置通常已经维护大量国内直连、海外代理、Apple、局域网、GEOIP、DNS 和最终兜底规则。大范围重构会增加更新、排错和回滚成本。

建议只做：

1. 新增一个独立 `AI-PROXY`；
2. 将明确的 OpenAI / Anthropic 核心域名指向它；
3. 保持普通 `PROXY`、`DIRECT` 和 `FINAL` 的原配置逻辑；
4. 通过 Shadowrocket 日志和真机测试识别是否需要精确补充域名。

## 3. 开始前：备份与只读审计

不要直接修改唯一一份原始配置。建议：

1. 保存原始订阅或原始 `.conf`；
2. 记录文件大小和本地 SHA-256；
3. 创建工作副本；
4. 确认配置可正常加载；
5. 审计 `[General]`、`[Proxy]`、`[Proxy Group]`、`[Rule]` 和最终 `FINAL`；
6. 查找已有 OpenAI / Anthropic 规则及其目标策略；
7. 查找可能提前匹配的关键词、共享 CDN 或通用代理规则；
8. 只在工作副本上实施后续修改。

原始订阅可能包含订阅 URL、服务器地址和认证信息，不能公开上传。

## 4. 创建独立 AI-PROXY

Shadowrocket `.conf` 使用 INI / conf 风格，不是 Clash YAML。通用代理组示意：

```ini
[Proxy Group]
AI-PROXY = select,<AI_NODE_1>,<AI_NODE_2>,policy-select-name=<AI_NODE_1>
```

说明：

- `<AI_NODE_1>`、`<AI_NODE_2>` 都是占位符；
- 必须替换为你的订阅中真实存在的节点名称；
- 可以选择一个或多个候选；
- `policy-select-name` 如使用，其值也必须与真实成员名称完全一致；
- `AI-PROXY` 应与普通代理选择相互独立；
- 本方法不使用 `url-test`、`fallback`、`load-balance`、随机或自动切换。

不同 Shadowrocket 版本的可用选项可能变化，修改后应实际编译或加载配置确认。

## 5. 醒目提醒：节点名称必须逐字符匹配

订阅节点的 Emoji、空格、编号和符号都可能属于名称的一部分。

例如，假设订阅中的真实名称是：

```text
🇰🇷 韩国 01
```

那么下面的写法不是同一个名称：

```text
韩国 01
```

这只是名称匹配示例，不表示你的订阅必须存在韩国节点。实际使用时必须以自己的当前订阅节点列表为准，不能照抄示例、旧订阅或其他人的配置。

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

Shadowrocket 规则示意：

```ini
[Rule]
DOMAIN-SUFFIX,openai.com,AI-PROXY
DOMAIN-SUFFIX,chatgpt.com,AI-PROXY
DOMAIN-SUFFIX,oaistatic.com,AI-PROXY
DOMAIN-SUFFIX,oaistatsig.com,AI-PROXY
DOMAIN-SUFFIX,oaiusercontent.com,AI-PROXY
DOMAIN-SUFFIX,anthropic.com,AI-PROXY
DOMAIN-SUFFIX,claude.ai,AI-PROXY
DOMAIN-SUFFIX,claude.com,AI-PROXY
DOMAIN-SUFFIX,claude.app,AI-PROXY
DOMAIN-SUFFIX,claudeusercontent.com,AI-PROXY
DOMAIN-SUFFIX,claudemcpcontent.com,AI-PROXY
```

如果原配置已经有 `[Rule]` 段，只需把规则插入该段的正确位置，不要机械复制第二个同名段头。其他示例段落也遵循同一原则。

如果配置中已经存在相同表达式，但目标是普通代理，应只把其目标调整为 `AI-PROXY`，不要保留两条策略不同的重复规则。只有确实缺失时才新增。

这 11 条是当前建议的审计起点，不是永久完整的域名全集。服务域名和客户端行为可能变化；后续应以官方信息和 Shadowrocket 实际日志为依据做精确补充。

## 7. first-match：必须检查首次命中

Shadowrocket 按规则顺序首次匹配。若在核心规则之前存在更宽泛的规则，例如：

```ini
DOMAIN-KEYWORD,chatgpt,PROXY
```

那么 `chatgpt.com` 可能先进入普通代理。应将精确的 AI 核心规则放在这类宽泛规则之前。

推荐思路：

1. 精确 OpenAI / Anthropic 规则；
2. 原有明确直连规则；
3. 原有普通代理和业务规则；
4. GEOIP 与其他原规则；
5. 原配置最终 `FINAL`。

具体顺序仍应以实际配置为准。不要为了套用模板而重排无关规则。

## 8. 不要扩大共享父域

不要因为 AI 页面使用第三方服务，就把以下大型共享域整体强制到 `AI-PROXY`：

- Cloudflare；
- AWS；
- Azure；
- Stripe；
- Sentry；
- Auth0；
- Datadog。

如果日志确认某个第三方地址确实是必要链路，应优先增加经过验证的精确 `DOMAIN` 或足够窄的 `DOMAIN-SUFFIX`，并重新检查首次命中。

## 9. Shadowrocket GUI 管理 select 组

Shadowrocket GUI 可以对 `select` 分组：

- 添加候选节点；
- 删除候选节点；
- 调整候选顺序；
- 手动选择当前节点。

因此，配置文件负责建立 `AI-PROXY` 和 AI 域名规则；日常候选管理可以在 GUI 中完成。未来住宅节点真正存在后，可执行：

```text
AI-PROXY
→ 添加
→ 🏠 AI-RESIDENTIAL
```

不要为了每次增删候选都重新生成整个配置，也不要预写第二个同名 `AI-PROXY`。重新导入配置后，应重新检查 GUI 中的成员、顺序和当前选择，不能假定运行状态一定保留。

## 10. 可选：独立静态住宅 / ISP SOCKS5

住宅代理是可选出口，不是使用本方法的前提。提供商通常会给出：

- IP；
- PORT；
- USERNAME；
- PASSWORD。

具体参数格式、认证方式和功能应以购买的代理服务商说明为准。

Shadowrocket `.conf` 通用模板：

```ini
[Proxy]
🏠 AI-RESIDENTIAL=socks5,YOUR_IP,YOUR_PORT,YOUR_USERNAME,YOUR_PASSWORD
```

如果配置已经存在 `[Proxy]` 段，只添加节点定义，不要再创建重复段头。

这只是格式模板：

- `YOUR_*` 不是可用参数；
- 不要把未填写的模板作为有效节点加载；
- 不要在公开文件中填入真实值；
- 具体 Shadowrocket 版本和代理服务商参数可能存在差异；
- 实际使用前必须完成配置加载和连通性验证。

如果需要提前保留模板，应将完整 `[Proxy]` 段和节点定义保持为注释。单 `#` 或双 `##` 可以作为个人维护标记，但不要把 `##` 描述为 Shadowrocket 官方定义的独立语法；启用前必须通过实际版本的配置加载测试。

## 11. 启用住宅节点的标准步骤

1. 从服务商取得真实参数，并确认认证格式；
2. 在本地私密副本中替换 `YOUR_IP`、`YOUR_PORT`、`YOUR_USERNAME`、`YOUR_PASSWORD`；
3. 使 `[Proxy]` 段和住宅节点定义同时成为有效配置；
4. 重新导入、编译或加载 Shadowrocket 配置；
5. 确认节点列表出现准确名称 `🏠 AI-RESIDENTIAL`；
6. 先单独测试节点连通性；
7. 进入 `AI-PROXY`，通过 GUI 添加该住宅节点；
8. 保留至少一个经过验证的普通代理节点作为备用；
9. 完成出口和 AI 服务测试后，再决定是否将其设为当前 AI 出口。

不要在住宅 proxy 定义尚未生效时，把它作为有效组成员引用。

## 12. 住宅节点与普通订阅节点是同级出口

默认路径是：

```text
App
 ↓
Shadowrocket
 ↓
AI-PROXY
 ↓
🏠 AI-RESIDENTIAL
 ↓
Internet
```

它不是：

```text
普通订阅节点 → 住宅代理
```

选择住宅出口后，AI 流量通常直接交给住宅 SOCKS5，而不再经过普通订阅节点。链式代理是另一个需要单独验证客户端支持和故障行为的设计，本 V1 方法不默认实现。

## 13. 保持普通流量原逻辑

除非另有明确目标，Minimal Patch 不应修改：

- DNS Server；
- IPv6；
- TUN 路由；
- LAN / 局域网直连；
- Apple 原有设计；
- 国内和 GEOIP 规则；
- 普通 `PROXY`；
- `DIRECT`；
- 最终 `FINAL`；
- Google、YouTube、Telegram、X / Twitter、Facebook、Instagram 等原有规则。

本方法不规定 `FINAL` 必须是 `DIRECT` 或 `PROXY`，而是要求保留当前可正常使用的原订阅逻辑，除非另行评估。

### Discord 在 `FINAL,DIRECT` 配置中的处理

如果配置采用“国内流量 `DIRECT`、特定海外服务 `PROXY`、最终 `FINAL,DIRECT`”的组合，Discord 缺少专用规则时，相关请求可能落入 `DIRECT`，从而出现 App 无法登录或部分功能异常。可根据实际情况在 `[Rule]` 中、最终规则之前加入：

```ini
DOMAIN-SUFFIX,discord.com,PROXY
DOMAIN-SUFFIX,discord.gg,PROXY
DOMAIN-SUFFIX,discordapp.com,PROXY
DOMAIN-SUFFIX,discordapp.net,PROXY
DOMAIN-SUFFIX,discord.media,PROXY
DOMAIN-SUFFIX,discordsays.com,PROXY
```

Discord 应使用普通 `PROXY`，不属于 `AI-PROXY`。本项目的 iPhone 真机测试中，这 6 条规则已满足 Discord 登录、消息收发、图片加载和语音功能；这不代表未来 Discord 的所有功能都会永久只使用这些域名。

如果仍有问题，应先查看 Shadowrocket 请求日志，确认实际漏网请求后再做最小补充。不建议无依据加入 Sentry、AppsFlyer、Cloudflare 等大型共享或第三方域名。

### Mixin / WalletConnect 的最小分流示例

如果使用 `FINAL,DIRECT`，Mixin 的聊天和转账正常，但扫码登录或连接外部 DApp 失败，可先查看 Shadowrocket 日志。某次实际排查中，`api.mixin.one`、`blaze.mixin.one` 和 `relay.walletconnect.org` 落入了 `FINAL,DIRECT`；按该证据，在 `[Rule]` 中、最终规则之前加入：

```ini
DOMAIN-SUFFIX,mixin.one,PROXY
DOMAIN,relay.walletconnect.org,PROXY
```

这两条使用普通 `PROXY`，不进入 `AI-PROXY`；WalletConnect 只精确匹配 `relay.walletconnect.org`。该案例在 iPhone Shadowrocket“配置”模式下完成静态检查与功能真机验收：聊天、转账继续正常，扫码登录及外部 DApp / WalletConnect 连接恢复正常。修改后未额外执行日志级路由复核，因此不能把功能测试等同于每条请求的出口验证。

其他环境应先核对实际日志和首次命中规则。不要无依据扩大到整个 `walletconnect.org`，也不要顺带加入其他尚未确认必要的第三方域名。

本方法不增加 DeepSeek 专用规则，也不增加 Gemini 专用规则。

## 14. 重新订阅后的正确复用方法

不要永久维护一份越来越旧的 `.conf`，也不要把旧订阅连接参数机械覆盖到新订阅。

每次重新订阅后：

1. 保存新的原始配置；
2. 将它作为新的只读基线；
3. 记录本地 SHA-256；
4. 检查真实节点名称，尤其是 Emoji、空格和编号；
5. 创建新的工作副本；
6. 重新建立或检查独立 `AI-PROXY`；
7. 只加入新订阅中实际存在的候选；
8. 审计当前 11 个 AI 域名：已有时调整策略，缺失时才新增；
9. 确认核心规则位于可能抢先匹配的宽泛规则之前；
10. 保持新基线的普通 `DIRECT`、`PROXY`、`FINAL`、DNS、IPv6 和 TUN 逻辑；
11. 如需要，重新加入未泄漏凭据的住宅节点模板；
12. 做静态差异审计；
13. 导入 Shadowrocket 并完成真机测试；
14. 在 GUI 中重新核对候选和当前选择。

这是可重复实施的方法，不是永久复用同一份旧配置。不要假定旧节点名称、服务器参数和认证信息在新订阅中仍然有效。

## 15. Shadowrocket 与 Clash 语法不同

不要把 Clash YAML 的以下结构直接复制到 Shadowrocket `.conf`：

```yaml
proxies:
proxy-groups:
type:
server:
port:
```

Shadowrocket 使用自己的 INI / conf 风格，例如：

```ini
[Proxy]
节点名称=socks5,地址,端口,用户名,密码

[Proxy Group]
AI-PROXY = select,<AI_NODE_1>,<AI_NODE_2>
```

即使逻辑相同，也必须针对实际客户端重新确认语法、代理组行为和加载结果。

## 16. 静态验收清单

- [ ] 配置能正常读取或编译；
- [ ] 只有一个有效的同名 `AI-PROXY`；
- [ ] `AI-PROXY` 类型是 `select`；
- [ ] 候选节点名称逐字符匹配当前订阅；
- [ ] 当前 11 个 AI 域名首次命中 `AI-PROXY`；
- [ ] 宽泛关键词和普通代理规则不会提前命中核心域名；
- [ ] 未把大型共享父域整体加入 `AI-PROXY`；
- [ ] 普通 `DIRECT`、`PROXY`、`FINAL` 和其他原规则未被意外修改；
- [ ] DNS、IPv6、TUN、Apple 和 LAN 逻辑保持原样；
- [ ] 如启用住宅节点，`[Proxy]` 定义和 GUI / 组引用同时存在；
- [ ] 不存在重复组名、悬空节点引用或真实凭据泄漏。

## 17. 真机验收清单

通过 Shadowrocket 请求日志分别检查：

- [ ] 配置正常加载，无语法错误；
- [ ] ChatGPT 核心请求进入 `AI-PROXY`；
- [ ] OpenAI API / CLI 请求在适用时进入 `AI-PROXY`；
- [ ] Claude Web 请求进入 `AI-PROXY`；
- [ ] Claude Desktop 请求在适用时进入 `AI-PROXY`；
- [ ] Claude Code 请求在适用时进入 `AI-PROXY`；
- [ ] 每个 AI 请求最终使用 `AI-PROXY` 当前选择的真实节点；
- [ ] 普通代理流量仍按原订阅处理；
- [ ] 国内和直连流量仍按原订阅处理；
- [ ] 切换普通代理不会意外改变 AI 出口；
- [ ] 切换 `AI-PROXY` 不会改变普通流量路径。

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

## 18. 公开分享前的安全检查

不要：

- 把真实 proxy password 发到公开聊天；
- 公开上传原始订阅或含节点的 `.conf`；
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

## 19. 本 V1 方法明确不做

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

## 20. 常见踩坑

1. 把 `AI-PROXY` 当成真实节点；
2. 节点名称缺少 Emoji、空格或编号；
3. 新增规则却被更早的关键词规则抢先命中；
4. 同一表达式保留两个不同策略，造成冲突；
5. 为一个 AI 子域强制整个共享 CDN；
6. 住宅节点尚未定义就提前加入组；
7. 同时存在两个有效的同名 `AI-PROXY`；
8. 假定 GUI 候选和选择在每次重新导入后必然保留；
9. 把 Clash YAML 语法直接粘贴进 Shadowrocket；
10. 长期修改旧订阅，而不是在新基线上重复实施。

## 21. 免责声明

本指南仅用于网络配置和流量分流说明。请遵守所在地法律法规，以及网络服务、代理服务和 AI 平台的服务条款。任何 IP 类型都不能保证避免账号限制或平台风控。服务域名、客户端行为、配置语法和服务条款可能变化，实际使用时应重新核对并测试。
