---
title: "多租户系统设计实战：拓扑隔离、数据库表设计与报表体系"
date: 2026-09-02T21:30:00+08:00
slug: distributed-14-multi-tenant-design
draft: false
tags: ["多租户", "SaaS", "架构设计", "MySQL", "微服务", "Java", "分布式系统"]
categories: ["分布式系统"]
author: "lesshash"
description: "从零设计一套多租户 SaaS 系统：拓扑分层与租户识别、三种数据隔离模型选型、共享表模式下的建表与索引规范、租户上下文全局注入、Redis 缓冲式计量与预聚合报表体系，附完整 DDL、Java 代码与踩坑清单。"
---

## 🎯 什么是多租户，难在哪

多租户（Multi-Tenant）= 一套系统同时服务多个客户（租户），客户之间数据完全隔离、互不可见，但共享同一份代码和大部分基础设施。这是 SaaS 的商业基础：边际成本随租户数增加而摊薄。

我维护的一个推送平台就是这个形态：平台方部署一套 Netty OpenAPI 服务，下游多个业务方（IM、直播、电商）各自接入，每个业务方就是一个租户，用自己的 appKey/appSecret 做 HMAC 签名调用。租户之间配额独立、数据隔离、报表分开看。

多租户的难点不在"功能"，而在**隔离**与**共享**的平衡：

```
隔离太狠（每租户独立部署）→ 成本线性涨，SaaS 不赚钱
共享太狠（什么都不分）     → 一家出事全家遭殃，大客户不敢来
                         ↓
        设计目标：按"付费等级 + 资源体量"动态选择隔离等级
```

下面按拓扑 → 库表 → 报表三块展开，这也是多租户系统的三根支柱。

## 🌟 一、拓扑设计：租户在哪一层被识别、被隔离

### 1.1 整体拓扑

```
租户A(入门)     租户B(专业)      租户X(企业/超大客户)
    │               │                  │
    ▼               ▼                  ▼
┌─────────────────────────────────────────────────┐
│ 接入层：网关（鉴权 + 租户识别 + 限流）              │
│   识别手段：子域名 / appKey / JWT claims            │
└───────────────────────┬─────────────────────────┘
                        │  注入租户上下文
┌───────────────────────▼─────────────────────────┐
│ 无状态应用集群（全租户共享）                        │
│   TenantContext: ThreadLocal / Channel attr       │
└─────────┬─────────────────────┬─────────────────┘
          ▼                     ▼
┌──────────────────┐   ┌──────────────────┐
│ 共享库·共享表      │   │ 独立库/独立实例    │
│ 租户A B C D...   │   │ 租户X 专属        │
│ (tenant_id 隔离)  │   │ (企业版专属资源)   │
└──────────────────┘   └──────────────────┘
```

拓扑设计的核心决策只有一个：**隔离模型放在哪几层**。四种选择：

| 模型 | 隔离强度 | 成本 | 典型问题 | 适合 |
|---|---|---|---|---|
| 共享库·共享表 | ★ | 最低（一套库） | 越权风险靠代码兜底 | 入门/长尾租户（占90%） |
| 共享库·独立Schema | ★★ | 低 | 迁移/备份按 schema 粒度 | 中型租户，合规要求一般 |
| 独立库 | ★★★ | 中 | 连接池膨胀、DB 实例数涨 | 专业版、数据量大的租户 |
| 独立实例/独立部署 | ★★★★ | 高（整套资源） | 运维复杂度 ×N | 企业版、超大客户 |

实践结论：**不要选一种，要分层混合**。定价直接映射隔离等级——入门版（如 ¥2899/年）进共享表，专业版（¥4999/年）独立库，企业版"面议"的本质就是独立部署+专属运维。隔离等级是卖点，不是成本包袱。

### 1.2 租户识别：三个入口，一个原则

租户身份从哪来，决定了整个系统的安全边界：

1. **子域名**：`tenantA.saas.com` → 网关按 Host 反查租户。注意多域名共用网关时，租户配置必须按 Host 动态返回，不能缓存成单例；
2. **appKey/签名**（服务间调用）：请求带 appKey，签名用 appSecret 校验，同时可叠加时间戳+nonce 防重放；
3. **JWT claims**（终端用户调用）：tenant_id 放进 token payload，**服务端解析提取，绝不从 URL 参数读**——URL 会进访问日志、浏览器历史、Referer，等于把租户身份到处抄送。

一个原则：**租户身份只能来自经过鉴权的凭证，不能来自客户端自行声明的参数**。请求里带 `tenant_id=123` 参数的接口，等于把越权的大门敞开着。

### 1.3 租户上下文传播

识别出租户后，要在调用链里一路传播。HTTP 同步链路用 ThreadLocal：

```java
public final class TenantContext {
    private static final ThreadLocal<Long> TENANT = new ThreadLocal<>();

    public static void set(Long tenantId) { TENANT.set(tenantId); }
    public static Long get() { return TENANT.get(); }
    public static void clear() { TENANT.remove(); }
}

// 网关/过滤器入口：try-finally 保证清理，防止线程池串租户
filterChain.doFilter(request, response);
// finally { TenantContext.clear(); }
```

三个传播断点必须显式处理：

- **异步线程池**：ThreadLocal 不跨线程，提交任务时必须把 tenantId 作为参数带入，或在包装 Runnable 时捕获再恢复；
- **长连接**（Netty/WebSocket）：绑定到 Channel 的 Attribute，断连清理；
- **MQ 消息**：tenant_id 放消息头（properties），消费端第一件事就是恢复上下文，别放消息体里让每个消费者自己解析。

## 🌟 二、数据库表设计：共享表模式的生存法则

混合隔离下，独立库/独立实例的表设计就是普通单租户设计；真正的难点是**共享表**——90% 的租户、100% 的越权风险都在这里。

### 2.1 租户主表与配额表

```sql
CREATE TABLE tenant (
  id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  app_key      CHAR(32)     NOT NULL COMMENT '对外标识',
  app_secret   CHAR(64)     NOT NULL COMMENT 'HMAC签名密钥',
  name         VARCHAR(128) NOT NULL,
  tier         TINYINT      NOT NULL DEFAULT 1 COMMENT '1入门 2专业 3企业',
  db_router    VARCHAR(64)  NOT NULL DEFAULT 'shared' COMMENT 'shared/shard-x/dedicated',
  status       TINYINT      NOT NULL DEFAULT 1 COMMENT '1正常 2冻结',
  created_at   DATETIME(3)  NOT NULL,
  UNIQUE KEY uk_app_key (app_key)
) ENGINE=InnoDB COMMENT='租户主表';
```

注意 `db_router` 字段：把"这个租户的数据放哪"沉到数据里，路由层读它决定连哪个库。租户升级隔离等级时改一行配置即可，不用改代码。

配额独立成表，不挤在主表里：

```sql
CREATE TABLE tenant_quota (
  tenant_id    BIGINT UNSIGNED NOT NULL,
  quota_key    VARCHAR(32)  NOT NULL COMMENT '如 daily_push_count',
  quota_limit  BIGINT        NOT NULL,
  PRIMARY KEY (tenant_id, quota_key)
) ENGINE=InnoDB COMMENT='租户配额';
```

### 2.2 业务表：三条铁律

以推送平台的任务表为例：

```sql
CREATE TABLE push_task (
  id           BIGINT UNSIGNED NOT NULL COMMENT '雪花ID，非自增',
  tenant_id    BIGINT UNSIGNED NOT NULL,
  title        VARCHAR(128) NOT NULL,
  payload      JSON         NOT NULL,
  status       TINYINT      NOT NULL,
  created_at   DATETIME(3)  NOT NULL,
  PRIMARY KEY (tenant_id, id),                -- 铁律1
  KEY idx_tenant_status_created (tenant_id, status, created_at)  -- 铁律2
) ENGINE=InnoDB COMMENT='推送任务';
```

**铁律1：业务主键不用自增，用雪花ID。** 共享表里自增 ID 是全局连续的，租户A今天创建拿到 ID=10000，明天拿到 10500，一减就知道整个平台的全局业务量——把平台的商业数据免费送给了每个租户。雪花ID趋势递增但不连续，且天然带时间信息。

**铁律2：主键和所有二级索引都以 tenant_id 打头。** 共享表里几乎所有查询都长这样：`WHERE tenant_id=? AND status=? ORDER BY created_at DESC`。索引最左列对齐 tenant_id，查询才能精准剪枝；漏了 tenant_id 的索引在这个模型里几乎没有价值，还会拖累写入。

**铁律3：没有"不带 tenant_id 的查询"。** 业务代码里出现 `WHERE status=?` 这种全表维度扫描，要么是平台运营场景（去报表体系做，见第三节），要么就是 bug。

### 2.3 行级隔离：用拦截器全局兜底

靠人肉在每个 SQL 里写 `tenant_id=?` 迟早漏。正确姿势是全局方案——ORM 层拦截器自动注入：

```java
// MyBatis-Plus 现成方案：一行配置自动改写 SQL
TenantLineInnerInterceptor tenantInterceptor = new TenantLineInnerInterceptor(new TenantLineHandler() {
    @Override
    public Expression getTenantId() {
        return new LongValue(TenantContext.get());   // 从上下文取，不从参数取
    }

    @Override
    public String getTenantIdColumn() { return "tenant_id"; }

    @Override
    public boolean ignoreTable(String tableName) {
        return tableName.startsWith("rpt_")         // 平台聚合报表表不带租户过滤
            || tableName.equals("tenant");          // 租户表本身按 appKey 查
    }
});
mybatisPlusInterceptor.addInnerInterceptor(tenantInterceptor);
```

拦截器改写后，`SELECT * FROM push_task WHERE status=1` 会自动变成 `WHERE status=1 AND tenant_id=123`。白名单（ignoreTable）要克制：每放行一张表都是一次人工确认"这张表确实该跨租户"。

兜底之上再加一道验证：**自动化测试里放两个租户的账号互相查对方的数据，断言查不到**。这条用例每年都能救几次线上。

### 2.4 缓存键也必须带租户

数据库隔离做得再好，缓存一串就全白搭：

```
✗ cache key: push:task:10001
✓ cache key: t:{tenant_id}:push:task:10001
```

Redis 的 key 没有命名空间强制，全靠约定。建议把"拼缓存 key 必带租户前缀"封装成唯一的工具类方法，禁止业务代码手拼。

## 🌟 三、报表设计：租户自助 + 平台运营两套打法

多租户系统的报表天然分两类，数据流方向完全不同：

```
             租户自助报表                     平台运营报表
          （租户看自己）                    （平台看全部租户）
                │                               │
   在线库.业务流水表                 在线库 → 只读副本 / 数仓
   （tenant_id 过滤）                （跨租户聚合只在这里做）
        │ 快照/增量                        │
        ▼                                 ▼
   租户维度预聚合表                   ODS→DWD→DWS 分层
   rpt_daily (tenant_id,…)          按 tenant_id 分区
        │                               │
        ▼                               ▼
   报表API（上下文强制过滤）          运营看板（营收/活跃/配额消耗）
```

### 3.1 计量与预聚合：Redis 缓冲 + 批量刷库

租户报表的第一手数据是计量（usage）：调用量、推送成功数、消息量。这些是**高频计数**，一条一写数据库必死。正确架构是缓冲批量写：

```
业务发生 ──► Redis HINCRBY 累加（内存，微秒级）
                │
                │  定时任务（如每5秒/每分钟）
                ▼
          Lua 脚本批量取出+清零 ──► INSERT ... ON DUPLICATE KEY UPDATE 刷入聚合表
                                     （一次性写几十上百行，数据库毫无压力）
```

聚合表按"租户 × 日期 × 指标"设计，天然去重、天然幂等：

```sql
CREATE TABLE rpt_daily (
  tenant_id  BIGINT UNSIGNED NOT NULL,
  stat_date  DATE          NOT NULL,
  metric     VARCHAR(32)   NOT NULL COMMENT 'push_total/apns_ok/mi_ok...',
  value      BIGINT        NOT NULL DEFAULT 0,
  PRIMARY KEY (tenant_id, stat_date, metric)
) ENGINE=InnoDB COMMENT='租户日聚合表';
```

这个模式的取舍要想清楚：**允许秒级丢失、容忍轻微延迟，换取写入端几乎零压力**。计量数据通常可接受（崩溃丢了 5 秒计数，不影响计费大局）；但交易类数据绝不能走这条路，必须落库后再聚合。

报表查询全部命中聚合表主键 `(tenant_id, stat_date, metric)`，任意租户查任意日期区间都是毫秒级，且查询成本与该租户的消息量无关——这一点很关键：**大租户不能靠多付费就把报表接口拖死**。

### 3.2 流水表：分区与归档

原始流水（如每条推送记录）是聚合表的证据链，量大但查询稀少：

- 按月分区（PARTITION BY RANGE），历史分区直接归档到冷存储/对象存储；
- 查询一律带 `tenant_id + 时间范围`，命中分区裁剪；
- 流水表严禁 JOIN 大表做实时统计——那是聚合表的工作。

### 3.3 平台运营报表：跨租户聚合只在库外做

"全平台今日推送量 Top10 租户"这类查询是共享表的天敌——它天然不带 tenant_id，拦截器会拦，业务上也确实要跨租户。规则：

1. **永远不在在线主库跑跨租户 GROUP BY**。一个没走索引的全表聚合能把全部租户的写入拖住；
2. 走**只读副本**或**数仓**（Canal/Binlog 同步到分析库），在线库只负责产生数据；
3. 数仓模型里 tenant_id 是最核心的分区键/维度键，所有租户画像（活跃度、配额消耗、流失预警）都在数仓算，算完回写到平台侧的运营库。

### 3.4 时区：租户的"今天"不是同一个今天

日聚合表里的 `stat_date` 是谁的日期？租户在吉隆坡和在西雅图，"今日调用量"差 15 小时。工程上干净的解法：

- **存储一律 UTC**，聚合原始数据按 UTC 时间桶；
- 租户档案里存 `timezone`，**时区转换在应用层做**（Java 的 `ZoneId`），把 UTC 桶换算成租户本地日期后再出报表——不要指望在 SQL 里用 `CONVERT_TZ` 修修补补，函数包列子句会让索引和分区双双失效。

## 📊 一张表总结设计决策

| 决策点 | 推荐方案 | 反面教材 |
|---|---|---|
| 隔离模型 | 按定价分层混合：共享表/独立库/独立部署 | 一刀切全共享或全独立 |
| 租户识别 | 子域名/appKey/JWT，服务端提取 | 接口收 tenant_id 参数 |
| 上下文传播 | ThreadLocal + 显式跨线程/跨网传递 | 假设上下文"到处都在" |
| 业务主键 | 雪花ID + PRIMARY KEY(tenant_id, id) | 全局自增暴露平台体量 |
| 二级索引 | tenant_id 最左 | 普通单列索引 |
| 行级隔离 | ORM 拦截器全局注入 + 双租户互测用例 | 人肉每个 SQL 手写 |
| 缓存键 | t:{tenant_id}:... 统一工具类 | 裸业务键 |
| 高频计量 | Redis 缓冲 + 定时批量刷聚合表 | 一条一写数据库 |
| 报表查询 | 只打聚合表 | 实时 GROUP BY 流水表 |
| 跨租户统计 | 只读副本/数仓 | 在线主库跑运营报表 |
| 时区 | 存 UTC，应用层按租户时区换算 | SQL 里 CONVERT_TZ |

## ⚠️ 踩坑清单

1. **某个 DAO 忘写 tenant_id** → 数据跨租户泄露。拦截器兜底 + 双租户互查测试，两手都要硬；
2. **线程池任务里读 TenantContext 拿到 null（或上个请求的租户）** → 上下文不跨线程，提交任务时显式携带；线程池复用导致"读到别人的租户"比 null 更危险；
3. **缓存 key 忘带租户前缀** → 租户A刷出来的数据给租户B看，数据库层查不出任何异常，极难排查；
4. **共享表用自增主键** → 租户反推平台全局业务量；
5. **大租户的慢查询拖垮共享库**（noisy neighbor）→ 配额+限流双保险，网关层就挡掉（滑动窗口限流器可以顺手回顾：[手写滑动窗口限流器](/2026/09/java-concurrency-02-sliding-window-limiter/)）；
6. **运营报表直接在主库跨租户 GROUP BY** → 一次报表拖垮全部租户；
7. **MQ 消费者不恢复租户上下文** → 消费端写库时 tenant_id 为空，拦截器要么报错要么写脏；
8. **租户升级独立库时只迁数据不改路由** → db_router 字段+统一路由层，迁移变成配置变更；
9. **只设计了"隔离"，没设计"计量"** → 计量是 SaaS 计费和配额的地基，表结构第一天就要留好。

## 📝 总结

多租户系统的设计可以浓缩成三句话：

- **拓扑**：隔离等级跟着钱走（定价分层映射部署形态），租户身份只信凭证不信参数；
- **库表**：共享表模式下，tenant_id 是主键、是索引、是缓存前缀、是 SQL 改写器的一切——四层都要对齐，漏一层就是事故；
- **报表**：高频计量 Redis 缓冲批量刷、租户报表只打预聚合、跨租户统计永远离开在线主库。

相关练习：这套系统里用到的限流与并发原语，见 [手写生产者-消费者](/2026/09/java-concurrency-01-producer-consumer/) 与 [手写滑动窗口限流器](/2026/09/java-concurrency-02-sliding-window-limiter/)。
