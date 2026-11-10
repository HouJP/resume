# FXCenter: Interview Story and Follow-up Questions

本文用于英文面试练习，基于本人提供的项目经历整理。重点是高流量客户端接入时的缓存、数据完整性与故障行为设计，而不是将整个 FXCenter 平台归为个人实现。未确认的实现、业务风险边界与性能指标明确标注。

## 1. 故事主线

集中汇率供给 → 接入高流量客户端 → 查询完全本地化 → 预加载与完整性检查 → 两层 fallback → 告警和人工恢复。

| 要素 | 已知内容 |
| --- | --- |
| 平台背景 | 新建统一汇率平台；此前不同 components 分别直接从银行获取 rates |
| 数据来源 | JPMC、HSBC、Bloomberg 等 vendors |
| 业务差异 | Checkout 在银行 rates 基础上加 markup；Search、View Item 使用当天 Bloomberg mid-market rates |
| 规模变化 | 最初只有流量较小的 Checkout；Search、View Item 接入后，需要避免查询流量直接冲击 FXCenter server |
| 个人责任 | 根据业务需求设计缓存，包括 client library cache、retention、completeness check、fallback、metrics 与 monitoring |
| 核心约束 | Client 的 get-rate lookup 不访问 server；预加载、重建 cache 时通过 search API 访问 FXCenter |
| 数据生命周期 | 每天 7 点准备、8 点预加载、10 点生效；有效期从一天 10 点到下一天 10 点 |
| 客户端策略 | 缓存该 client 所需的全部 rates，并保留前一天数据；collection 不完整时整批 fallback |
| 监控与恢复 | Completeness false 产生 metrics，Grafana 告警；人工修复后重启 client 实例，重新获取 rates |

结果边界：已明确设计与实际恢复方式，但没有提供 QPS、延迟、请求下降比例、故障次数等量化结果，不编造性能或零故障结论。

### 1.1 口述提示卡：看关键词，不背全文

每行用自己的话讲 1–2 句。Vendor 切换留给追问，先讲客户端设计。

| 顺序 | English cues | 想起什么 |
| --- | --- | --- |
| 平台与业务 | vendors → Checkout markup / Search mid-market rates | 统一拿 rates，不同业务需要不同的 rates |
| 规模变化 | Checkout first → Search and View Item → more traffic | 新 clients 流量大，不能让每次查询都访问 server |
| 我的设计 | client cache → local lookups → no server call | 我设计客户端缓存；业务查询只读本地 |
| 数据周期 | prepare at 7 → load at 8 → valid at 10 | 生效前加载，每期有效到次日 10 点，保留前一天数据 |
| 缺失风险 | configuration issue → missing rates → repeated requests | 源数据齐全，派生数据仍可能缺失；反复拉取可能冲击 server |
| 保护 | completeness check → whole-set fallback → yesterday's rates | 不完整就整批回退，不不断重试 |
| 恢复 | metric → Grafana alert → fix → restart clients | 告警后人工修复，重启 client 重新加载 |
| 设计重点 | local lookups → stop repeated server requests | 查询不依赖 server，避免缺数据导致大量请求；不编性能数字 |

只记一条线：rates → more traffic → local cache → preload → missing data → fallback → recovery。

卡住时用："At first..." / "Then..." / "The main rule was..." / "If rates were missing..." / "After we fixed it..."。

## 2. 30-second Overview

> I worked on FXCenter, which provides exchange rates to eBay applications. When Search and View Item needed to use it, we had to support much more traffic. I designed the client cache so rate lookups stayed local. Clients loaded rates ahead of time and kept the previous day's rates as a fallback. We also checked for missing rates and added alerts to avoid repeated requests to the server.

## 3. Main Story: Simple Spoken English

> One project I worked on was helping FXCenter support more traffic.
>
> FXCenter provides exchange rates to eBay applications. It gets rates from vendors like JPMorgan, HSBC, and Bloomberg. Different applications need different rates. Checkout adds a markup to bank rates. Search and View Item use Bloomberg mid-market rates.
>
> At first, only Checkout used FXCenter, and its traffic was fairly low. Search and View Item had much more traffic. If every rate lookup called our server, we were worried it wouldn't keep up.
>
> I designed the client caching approach. That included how long to keep rates, what to do when rates were missing, and what to monitor.
>
> The main rule was simple: rate lookups must use the local cache. Clients could call FXCenter to load or rebuild the cache, but not to fetch a rate during a business request.
>
> We prepared rates at seven, clients loaded them at eight, and they became valid at ten. The rates were valid until ten the next day. Clients also kept the previous day's rates as a fallback.
>
> We also checked that each client had all the rates it needed. Even if the vendor gave us a full set, a configuration problem could still leave some business rates missing.
>
> If any required rates were missing, the API returned completeness false. The client then used the previous day's set instead of asking the server again and again. We recorded a metric and added a Grafana alert so the team could investigate.
>
> After we fixed the problem, we restarted the client instances to load the corrected rates.
>
> The main idea was to keep lookups local and stop missing data from causing too many requests to the server.

讲述提示：

- I 用于本人负责的缓存与故障行为设计；we / the platform 用于团队和平台能力。不声称所有 vendor integrations、派生逻辑都由本人实现。
- "不访问 server"指单次 get-rate lookup，不是 client 永远不访问 FXCenter。
- 时间安排有明确生效边界，但具体时区和启动、切换实现尚未补充；不要自行加入 UTC、本地时区或原子换表等说法。
- 恢复方式是人工修复后重启 client，不包装成自动通知或自动刷新。
- 说明设计意图与已知行为，不将"避免重试冲击"写成发生过的实际 outage。

## 4. Technical Deep Dive

### 4.1 查询与加载是两条路径

```text
正常查询：
业务请求 -> client library -> 本地 rate cache
                           不触发 FXCenter server 请求

预加载 / cache 重建：
client -> FXCenter search API -> 检查所需 rate collection 的完整性
    完整：加载数据供相应有效期使用
    不完整：返回 completeness false，client 整批使用前一天 collection

修复：
告警 -> 人工修复数据或配置 -> 重启 client 实例 -> 重新加载
```

这不是常见的"本地 cache miss 后按需请求 server"的策略。设计约束是将查询路径与服务端获取数据的路径分开，避免缺失 rates 在高流量下造成重复请求。

### 4.2 两层 fallback：不要混为一谈

| 层次 | 触发条件 | 已确认行为 |
| --- | --- | --- |
| Rate source 层 | In-house rate derivation 当天选中的银行 rates 不完整 | 每天在 JPMC、HSBC 之间按 round-robin 选择；所选 vendor 不完整则切到另一家；两家都不完整则使用前一天的源 rates，再派生业务所需 rates |
| Client collection 层 | Client 所需的 derived collection 不完整，例如配置导致缺失 | Search API 返回 completeness false；client 不反复拉取缺失数据，整批 fallback 到前一天 collection，并触发告警 |

> We checked the source rates and the client rates separately. Even if the source rates were complete, some rates needed by a client could still be missing.

银行切换策略是已描述的 in-house derivation 场景，不能扩大成所有 Bloomberg rates 或所有业务都使用同样的 vendor 选择规则。

### 4.3 Fallback 与 SLA 的边界

本人确认所有 clients 使用同样的前一天 fallback 策略，并依赖"一天内解决 rate 缺失"的运营 SLA。

SLA 不是数据存在性的数学保证，也不是金融风险已经被证明可接受的依据。前一天数据在正常运行与该运营约束下被预期可用；连续缺失、全新 client、无历史 cache 等边界的实际行为尚未确认。

面试中准确说法：

> We used the previous day's rates as a fallback. Our SLA was to fix missing-rate issues within one day.

不要说 "yesterday's rates can never be missing" 或 "stale rates are always safe for Checkout"。Checkout 的业务接受依据和损失控制需要另补事实。

## 5. Follow-up Questions

### Requirements and Ownership

**1. "Why did eBay need a centralized rate platform?"**

回答重点：此前各 components 分别从银行获取 rates，没有统一供给平台。FXCenter 集中接入 vendors，并按不同业务需求提供 rates。

待补：如果说明接入重复、运维成本或口径差异，需给出实际经历，不能仅因"分散"就断言发生过这些问题。

**2. "Why couldn't every client use the same rates?"**

> Checkout needed bank rates with a markup. Search and View Item only needed Bloomberg mid-market rates for display.

回答重点：需求决定 rate 类型，不是任意按 client 名称区分。不要将展示 rates 与实际成交 rates 混为一谈。

**3. "What did you personally design?"**

> I designed how the client cache worked: how long to keep rates, how to check for missing data, when to fall back, and what to monitor. I also made sure rate lookups didn't call the server.

待补：本人直接编码与推动团队实现的边界。可以说明设计责任，不将全部平台开发归为个人工作。

### Caching and Data Lifecycle

**4. "Why not fetch a missing rate on demand?"**

> If we did that, a missing rate could make lots of clients keep calling the server. We wanted lookups to stay local, so we used the previous day's full set instead.

这是方案理由，不代表已经测量过一次实际重试风暴。

**5. "How did preloading and the effective period work?"**

回答重点：7 点准备，8 点预加载，10 点生效；rate 有效期为一个 10 点到下一个 10 点。Client 保留所需 rates 及前一天数据。

待补：使用的时区、8 点数据对应的日期标识、10 点前后的选择机制，以及旧数据清理和新实例启动流程。不要将"预加载"不加解释地说成一定是下一自然日的数据。

**6. "Could scheduled refreshes still overload the server?"**

回答重点：Local-only lookup 消除了随页面流量增长的查询请求，但定时预加载仍会访问 server。这两类负载需要分开讨论。

待补：实际实例规模、刷新是否错峰、server 缓存及容量验证。不虚构 jitter、分批刷新或压力测试结果。

**7. "Why fall back for the whole collection instead of individual missing pairs?"**

回答重点：实际采用 collection completeness check 和整批 fallback，避免业务在缺少所需 rate 时反复请求 server。

待补：当时是否还考虑了跨币种或跨规则版本一致性。可以将其作为可能的讨论方向，但不能写成已确认决策依据。

### Reliability and Recovery

**8. "What if the vendor's rates were incomplete?"**

回答重点：In-house derivation 先切换另一家银行，若两家都不完整则使用前一天的 source rates，再派生 client 所需 rates。后续 client collection 检查是另一个保护层。

**9. "Why check completeness if the source rates were already complete?"**

> A full set of source rates wasn't enough. A configuration problem could still leave some business rates missing. We checked that the client had everything it needed before using the set.

待补：完整性检查中实际的 expected currency pairs、rate 类型和日期等判断项，不自动加入尚未描述的检查。

**10. "How did you recover after fixing incomplete rates?"**

> Recovery wasn't automatic. We fixed the data and then restarted the client instances so they could load the corrected rates.

待补：为什么当时选择重启、是否逐实例执行，以及如何控制对业务的影响。不假设恢复是零停机或自动滚动重启。

**11. "Why was using yesterday's rate acceptable for Checkout?"**

回答重点：所有 clients 的实际策略一致，缺失修复 SLA 为一天。这说明运营安排，但业务接受旧 rates 的依据仍待补。

若只记得技术策略，不冒充风险决策者：

> The system used the previous day's rates. That's the technical policy. Whether those rates are acceptable for Checkout is a separate business risk decision.

待补：业务方如何批准、旧数据最长可用时间，以及前一天也不存在时的实际处理。

**12. "What metrics and alerts did you add?"**

> We recorded a metric when the search API returned completeness false. A Grafana alert then told us that rates were missing.

目前只确认这一告警链路，不编造 cache hit ratio、fallback 使用量、数据年龄或 vendor latency 等已落地指标。

### Results and Reflection

**13. "How did you measure the impact?"**

回答重点：先说明结构性变化，即查询完全本地化，server 接收预加载和 cache 重建请求，而非每次页面查询。当前没有可引用的 QPS、延迟或请求下降比例。

> More page views didn't mean more rate requests to our server, because lookups stayed local. I don't have exact performance numbers to share.

待补：可公开的接入规模、观测结果或测试证据；不要因没有数字而编造指标。

**14. "What trade-off did you make, and what would you improve?"**

> When rates were missing, we used the previous day's set and fixed the problem manually. The trade-off was using older rates rather than making clients keep calling the server.

"接受旧 rates"只描述已确认策略，不替代 Checkout 业务风险解释。正常生效周期不是异常 fallback，不笼统说整个系统牺牲了正常 freshness。

可讨论的改进方向：更明确的历史数据边界、无需重启的受控刷新、预加载负载控制。仅作为未来设计讨论，不说成当时已经实施，也不编造真实故障或后悔事件。

## 6. Useful Expressions

| English | 中文与使用边界 |
| --- | --- |
| Centralized exchange-rate platform | 统一汇率平台 |
| Mid-market rate | 中间价；与有 markup 的报价区分 |
| Apply a markup | 在基础 rates 上加价 |
| Rate derivation | 根据源 rates 与业务规则派生 rates |
| Local-only lookup | 单次查询只读本地，不按需访问 server |
| Preload the rate collection | 在生效前加载该 client 所需的 rates |
| Effective period / validity window | 汇率有效期；不是 cache retention 的同义词 |
| Completeness check | 检查业务所需数据是否齐全 |
| Previous-period fallback | 异常时使用上一期数据，不代表重新变成当天新鲜 rates |
| Round-robin vendor selection | 在描述的银行间按天轮换选择，不是每笔查询轮询 |
| Repeated requests / retry storm | 重复请求 / 重试风暴；本故事后者是防范风险，不是已确认事故 |
| Operational recovery | 人工修复并重启加载，不等于自动恢复 |

## 7. 下一轮需要补齐的事实

- 生效时间的时区、日期标识与具体 cache 切换机制。
- Collection completeness 的实际检查项及预期集合来源。
- 前一天数据不存在、连续异常和新实例启动时的实际行为。
- Checkout 接受旧 rates 的业务依据与风险边界。
- 为什么选择重启恢复，以及生产执行方式。
- 预加载并发、server-side cache 与容量验证的实际实现。
- 可公开的接入结果、性能证据或真实故障案例，以及本人编码范围。

练习顺序：30 秒概述 → 完整故事 → 优先回答问题 3、4、9、10、11、13。先讲清查询边界和完整性保护，再按追问展开 vendor fallback；不逐字背稿。
