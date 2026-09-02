---
title: "AI应用练习01：LLM 使用入门——三个API原语、结构化输出与手写Agent循环"
date: 2026-09-02T22:30:00+08:00
slug: ai-app-01-llm-api
draft: false
tags: ["LLM", "OpenAI兼容API", "Function Calling", "Agent", "AI应用开发", "Python"]
categories: ["AI应用开发"]
series: ["AI应用开发练习"]
author: "lesshash"
description: "不用任何 SDK、纯 requests 裸调 OpenAI 兼容 API：对话/结构化输出/Function Calling 三个原语，附结构化输出对比实验的真实数据（10/10 vs 10/10）、一个手写 Agent 循环的完整运行记录，以及 WAF 拦 UA、tool_calls 消息顺序等高频坑。"
---

## 🎯 练习目标

所有 AI 应用——客服机器人、RAG 问答、自动化工作流——拆到底层都是同一副骨架：**LLM API 的三个原语**：

```
原语1  chat/completions     对话补全（一切的基础）
原语2  结构化输出            让模型输出程序能解析的 JSON
原语3  function calling     让模型调用你定义的函数（Agent 的心脏）
```

这次练习全程不用 openai SDK，纯 `requests`/`urllib` 裸调，把 SDK 帮你藏起来的东西亲手摸一遍。通道就用一条 OpenAI 兼容网关（DeepSeek、各类中转、自建网关都行），凭证进环境变量，绝不写死在代码里：

```bash
export LLM_BASE_URL="https://你的网关/v1"
export LLM_API_KEY="你的key"
```

## 🌟 原语一：对话补全——先搞懂"模型没有记忆"

```python
import os, json, urllib.request

def chat(messages, **kw):
    body = {"model": "gpt-4o-mini", "messages": messages, **kw}
    req = urllib.request.Request(
        os.environ["LLM_BASE_URL"] + "/chat/completions",
        data=json.dumps(body).encode(),
        headers={"Authorization": f"Bearer {os.environ['LLM_API_KEY']}",
                 "Content-Type": "application/json",
                 "User-Agent": "llm-lab/1.0"})
    with urllib.request.urlopen(req, timeout=90) as r:
        return json.load(r)

resp = chat([
    {"role": "system",  "content": "你是一位毒舌但专业的代码评审官"},
    {"role": "user",    "content": "帮我看看这段代码：while(true){} 有什么问题"},
    {"role": "assistant", "content": "这会100%吃满一个CPU核心，且永远不退出。"},
    {"role": "user",    "content": "那怎么改？"},
])
print(resp["choices"][0]["message"]["content"])
```

三个角色的分工：**system** 定人设和规则（每次请求都生效）、**user** 是输入、**assistant** 是模型历史回复。

最重要的一课：**HTTP 是无状态的，模型没有记忆**。所谓"多轮对话"，是你把历史 messages 全量重发一遍——第 4 条消息能接上第 3 条的语境，不是因为服务端记得，而是因为上下文每次都完整重传。推论也很直接：

- 对话越长，每次请求的 token 越多，越慢越贵；
- "记忆"要自己做：超出上下文窗口就得裁剪/摘要历史消息。

### 两个参数，一道分水岭

- **temperature**：采样随机性。0 = 尽量确定（分类、抽取、代码），越高越发散（文案、头脑风暴）；
- **max_tokens**：输出上限。它是**护栏不是精度**——防止模型跑飞烧钱，不是"要多少给多少"。

### 真实工程坑：网关 WAF 拦默认 UA

上面代码里 `"User-Agent": "llm-lab/1.0"` 不是摆设：我用的网关 WAF 会拦 Python-urllib 的默认 UA，不自定义就 403。另外该网关的 `/v1/models` 也被关了（403），模型名只能硬编码。这类"兼容网关各有各的脾气"的问题，SDK 用户很少碰到，裸调一次全见识了——排查思路就是拿 curl 对照试，先分清是代码问题还是网关问题。

## 🌟 原语二：结构化输出——模型说话给程序听

业务里要的几乎从来不是"一段话"，而是能进数据库的 JSON。两种做法：

1. **prompt-only**：system 里写"只输出 JSON，不要任何其他文字"；
2. **response_format**：请求体加 `response_format={"type": "json_object"}`。

### 对比实验（真实数据）

任务：分析商品评论，输出 `{"情绪":"正面|中性|负面","关键词":[...],"是否建议跟进":bool}`。每方式跑 10 次，gpt-4o-mini，temperature=0，总耗时 163 秒：

```
[方式1: 仅靠 prompt 说『只输出JSON』] 成功 10/10
  样例: {"情绪":"中性","关键词":["物流快","包装破损","客服处理","补偿券","凑合"],"是否建议跟进":true}

[方式2: response_format json_object] 成功 10/10
  样例: {"情绪":"中性","关键词":["物流快","包装破损","客服处理","优惠券补偿"],"是否建议跟进":true}
```

诚实结论：**强模型 + 简单任务，prompt-only 也能 100%**。response_format 的价值场景是弱模型、复杂嵌套 schema、下游解析零容忍（一条失败就丢数据）时——约束在解码层生效，比"求"模型听话硬气得多。

另一个真实观察：temperature=0 下两次运行的**关键词仍然不同**（"补偿券" vs "优惠券补偿"）。temperature=0 不等于完全确定——网关路由到不同副本、浮点并行归约顺序都会引入抖动。想完全可复现，得自己缓存结果，别指望模型。

### 兜底解析：真实业务都会做

不管用哪种方式，解析前都要做一次"剥壳"：

```python
def try_parse(text):
    t = text.strip()
    m = re.search(r"\{.*\}", t, re.S)   # 抠出最外层花括号，剥掉 ```json 围栏和废话
    if not m:
        return None
    try:
        return json.loads(m.group(0))
    except Exception:
        return None
```

## 🌟 原语三：Function Calling——手写最小 Agent

这是三个原语里最容易被神化的一个。先说破本质：

> **模型从头到尾只是"输出了一段 JSON 说它想调什么"。执行永远发生在你的 Python 进程里。**

所以你可以在执行层做权限校验、参数白名单、SQL 预编译——tool 层是企业 AI 安全的第一道闸门。

### 工具说明书（给模型看的 JSON Schema）

```python
TOOLS = [
    {"type": "function", "function": {
        "name": "query_job",
        "description": "按岗位类别查询在招职位列表，返回职位名/地点/薪资/要求",
        "parameters": {"type": "object", "properties": {
            "category": {"type": "string",
                         "enum": ["IT互联网", "餐饮零售", "教育培训"]}},
            "required": ["category"]}}},
    {"type": "function", "function": {
        "name": "count_subs",
        "description": "统计订阅了某类岗位的求职者人数",
        "parameters": {"type": "object", "properties": {
            "category": {"type": "string",
                         "enum": ["IT互联网", "餐饮零售", "教育培训"]}},
            "required": ["category"]}}},
]

def query_job(category: str) -> dict:
    with sqlite3.connect("zhaopin.db") as con:
        rows = con.execute(
            "SELECT title,location,sal_key,requirements FROM jobs "
            "WHERE category=? AND status='active'", (category,)).fetchall()
    return [{"title": r[0], "location": r[1], "salary": r[2],
             "requirements": r[3]} for r in rows]

FUNCTIONS = {"query_job": query_job, "count_subs": count_subs}
```

注意 `FUNCTIONS` 这张注册表：模型只知道工具名字，真正执行靠它映射到 Python 函数。**永远不要 eval 模型给的函数名**。

### Agent 循环（灵魂在此）

```python
def agent_run(question: str, max_turns: int = 5) -> str:
    messages = [
        {"role": "system",
         "content": "你是职鹊求职平台的客服助手。涉及在招职位或订阅情况时，"
                    "必须先调用工具查询真实数据，不许凭空编造。"},
        {"role": "user", "content": question},
    ]
    for turn in range(max_turns):
        resp = chat(messages, tools=TOOLS)
        msg = resp["choices"][0]["message"]

        if not msg.get("tool_calls"):          # 没有工具调用 = 最终回答，唯一出口
            return msg["content"]

        messages.append(msg)                    # ① 先追加带 tool_calls 的 assistant 消息

        for tc in msg["tool_calls"]:            # ② 列表！模型可能一次要调多个工具
            name = tc["function"]["name"]
            args = json.loads(tc["function"]["arguments"])   # ③ 是 JSON 字符串不是 dict
            result = FUNCTIONS[name](**args)                  # 本地执行
            messages.append({"role": "tool", "tool_call_id": tc["id"],
                             "content": json.dumps(result, ensure_ascii=False)})
    return "（达到最大轮数限制，强制退出）"
```

### 真实运行记录

```
=== 测试1: 单工具 ===
-- 第1轮 finish_reason=tool_calls
-- 第2轮 finish_reason=stop
您好！目前 IT互联网 类的在招职位如下：
**Java后端工程师**
- 📍 工作地点：新加坡
- 💰 薪资：10000-20000
- 📋 要求：3年以上经验，熟悉Spring/微服务；英语可沟通

=== 测试2: 并行工具调用 ===
-- 第1轮 finish_reason=tool_calls
-- 第2轮 finish_reason=stop
（一轮回来了两个 tool_calls：query_job + count_subs 同时执行，
 回答里既有职位表格又有订阅人数）
```

看 `finish_reason` 的轮转就是 Agent 的心跳：`tool_calls → stop`。测试 2 验证了并行工具调用——模型在**一轮**里同时要了两个工具，代码里必须 for 循环逐个处理、逐个回填。

### 五个高频坑（全部踩过/防过）

1. **忘了先把带 tool_calls 的 assistant 消息 append 进 messages** 就直接 append tool 消息 → API 400：`An assistant message with 'tool_calls' must be followed by tool messages`。顺序必须是 `assistant(tool_calls) → tool → tool → ...`，每个 tool 消息靠 `tool_call_id` 对号入座；
2. **tool_calls 是列表**，漏处理一个就 400；
3. **arguments 是 JSON 字符串**不是 dict，直接 `**` 解包会炸，先 `json.loads`；
4. **tool 消息的 content 必须是字符串**，函数返回 dict/list 要 `json.dumps`（`ensure_ascii=False` 保留中文，省 token 又好读）；
5. **必须有 max_turns 上限**：模型陷入"调工具→不满意→再调"死循环时，你的 Agent 会一直烧钱。所有框架都有这个护栏，名字通常叫 max_iterations。

## 📊 三个原语怎么组合成应用

```
客服机器人    = 原语1(多轮) + 原语2(工单落库)
数据问答      = 原语3(查SQL/查API) + 原语1(组织语言)
RAG 问答     = 检索(传统代码) + 原语1(基于资料回答)
自动化工作流  = 原语3(N个工具) + 循环 + 护栏     ← 上面手写的 agent_run 就是它
```

下一篇就用这套地基搭 RAG：[AI应用练习02：RAG 使用实战——让92篇文章变成可问答的知识库](/2026/09/ai-app-02-rag/)。

## ⚠️ 踩坑清单

1. **key 写死在代码里** → 环境变量/配置中心，代码进 git 的那一刻 key 就算泄露；
2. **兼容网关的隐性脾气**：WAF 拦默认 UA、/v1/models 403、base_url 里嵌 key——拿 curl 对照排查，先分清代码问题还是网关问题；
3. **以为模型有记忆** → 无状态，历史每次重传，长对话要自己做裁剪/摘要；
4. **裸信 temperature=0 会确定性输出** → 网关路由、采样实现都会引入抖动，可复现靠缓存；
5. **结构化输出不做解析兜底** → 剥 markdown 围栏 + 正则抠 JSON + 失败重试，三件套缺一不可；
6. **eval 模型返回的函数名/路径** → 白名单注册表 + 预编译 SQL，tool 层就是安全边界；
7. **Agent 循环不加轮数上限** → 死循环烧钱，max_turns 是钱和安全的双保险。

## 📝 练习收获

- LLM 应用的地基就三个原语，SDK/LangChain 只是包装——裸调一遍，框架报错时才知道错在哪层；
- Function Calling 的本质是"模型出主意，你的进程动手"——安全和幂等都在你手里；
- 结构化输出的正确姿势：schema 明确 + response_format 兜底 + 解析三件套；
- 所有"智能"都在 prompt 和工具说明书里，所有"安全"都在你的执行层里。
