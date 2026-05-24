<p align="center">
  <h1 align="center">🔀 Model Router</h1>
  <p align="center"><strong>多模型智能路由 v2.2 — 时间感知路由 + 成本监控 + 影子流量 + 路由矩阵配置化</strong></p>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/Hermes-%3E%3D2.0.0-orange.svg" alt="Hermes">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License">
  <img src="https://img.shields.io/badge/version-2.2.0-blue.svg" alt="Version">
  <img src="https://img.shields.io/badge/kind-backend-purple.svg" alt="Kind">
</p>

Model Router 是 [Hermes Agent](https://github.com/weksbwrx62862/hermes) 的后端路由插件，通过时间感知、任务分类、成本评分和降级链等多维策略，在多个 LLM Provider 之间自动选择最优模型，在成本和性能间取得平衡。支持 MiMo / DeepSeek / NVIDIA NIM 等多 Provider 多密钥轮转，内置令牌桶限流与冷却黑名单机制，确保高可用。

## 功能矩阵

| 能力 | 说明 | 对应工具 / 钩子 |
|------|------|-----------------|
| 时间感知路由 | 按时段自动切换模型池，非高峰优先 MiMo（0.8x 系数），高峰优先 DeepSeek | `pre_llm_call` / `time_info` |
| 成本监控 | 实时追踪 token 消耗与费用，JSONL 持久化日志 | `cost_dashboard` |
| 影子流量 | 1% 抽样静默评估新模型，不影响生产流量 | `ShadowEvaluator` |
| 路由矩阵 | YAML/JSON 声明式路由规则，60s 热更新 | `RoutingMatrix` |
| 速率限制 | Provider 级令牌桶限流 + 冷却黑名单 | `rate_limit_status` |
| 多密钥轮转 | MiMo 3 Key / NVIDIA 5 Key Round-Robin + 健康检测 | 内置 |
| 长上下文绕路 | 超 200K tokens 优选大窗口模型，超 800K 直选最大 | 内置 |
| 任务分类 | 8 类任务 × 3 级复杂度自动识别 | `TaskClassifier` |
| 降级链 | 限流 / 冷却自动降级到备选模型 | `pre_llm_call` |
| AMA 联动 | 接收 Adaptive Multi-Agent 任务权重，动态覆盖策略 | `set_task_weight` |

## 架构图

```
                          ┌─────────────────────────────────────────┐
                          │            Hermes Agent                 │
                          └─────────────┬───────────────────────────┘
                                        │
                            ┌───────────▼───────────┐
                            │    pre_llm_call Hook   │
                            └───────────┬───────────┘
                                        │
                 ┌──────────────────────▼──────────────────────┐
                 │            Model Router Engine               │
                 │                                              │
                 │  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
                 │  │ 时间感知  │  │ 任务分类  │  │ AMA 联动   │ │
                 │  └────┬─────┘  └────┬─────┘  └─────┬─────┘ │
                 │       │             │               │        │
                 │       └──────┬──────┘───────────────┘        │
                 │              ▼                               │
                 │  ┌─────────────────────┐                     │
                 │  │   路由矩阵 (JSON)    │◄── 60s 热更新       │
                 │  └─────────┬───────────┘                     │
                 │            ▼                                  │
                 │  ┌─────────────────────┐                     │
                 │  │  多维评分 + 限流检查  │                     │
                 │  │  (cost/speed/quality)│                     │
                 │  └─────────┬───────────┘                     │
                 │            ▼                                  │
                 │  ┌─────────────────────┐                     │
                 │  │  降级链 + Key 轮转    │                     │
                 │  └─────────┬───────────┘                     │
                 └────────────┼──────────────────────────────────┘
                              │
                 ┌────────────▼──────────────────────────────────┐
                 │           Monkey-Patch Layer                   │
                 │   _build_api_kwargs → 自动切换 Provider/Key   │
                 └────────────┬──────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │   MiMo   │   │ DeepSeek │   │NVIDIA NIM│
        │ 3 Keys   │   │  OpenAI  │   │ 5 Keys   │
        └──────────┘   └──────────┘   └──────────┘
                              │
                 ┌────────────▼──────────────────────────────────┐
                 │         post_llm_call Hook                     │
                 │   CostMonitor 记录成本 + ShadowEvaluator 抽样  │
                 └───────────────────────────────────────────────┘
```

## 快速开始

### 前置条件

- Python 3.10+
- [Hermes Agent](https://github.com/weksbwrx62862/hermes) >= 2.0.0
- PyYAML

### 安装

```bash
git clone https://github.com/weksbwrx62862/model-router.git
cd model-router
pip install -e .
```

### 最小配置

1. 在 `~/.hermes/config.yaml` 中注册插件并配置 Provider：

```yaml
plugins:
  - name: model-router
    path: ./model-router

providers:
  mimo:
    api_key: ${MIMO_API_KEY}
    base_url: https://api.mimo.dev/v1
    models:
      - mimo-v2.5-pro
      - mimo-v2.5
  openai:
    api_key: ${OPENAI_API_KEY}
    base_url: https://api.openai.com/v1
    models:
      - deepseek-v4-pro
      - deepseek-v4-flash
  nvidia-nim:
    api_key: ${NVIDIA_NIM_API_KEY}
    base_url: https://integrate.api.nvidia.com/v1
    models:
      - deepseek-ai/deepseek-r1
      - meta/llama-4-maverick

plugins:
  model-router:
    strategy: auto
    peak_hours:
      off_peak_start: 0
      off_peak_end: 8
```

2. 在 `~/.hermes/.env` 中设置 API Key：

```bash
MIMO_API_KEY=sk-xxx
MIMO_API_KEY_2=sk-xxx
MIMO_API_KEY_3=sk-xxx
NVIDIA_NIM_API_KEY=nvapi-xxx
```

3. 启动 Hermes Agent，插件自动加载。

## 核心功能详解

### 时间感知路由

根据北京时间自动切换模型池优先级：

| 时段 | 范围 | 优先模型 | 原因 |
|------|------|----------|------|
| 非高峰 | 00:00 - 08:00 | MiMo | 0.8x 系数优惠，成本更低 |
| 高峰 | 08:00 - 00:00 | DeepSeek / NVIDIA NIM | 质量优先 |

通过 `time_info` 工具查看当前时段：

```json
{"name": "time_info", "arguments": {}}
```

### 成本监控

每次 LLM 调用后自动记录 token 消耗和费用，日志持久化到 `~/.hermes/model_router_costs.jsonl`。

通过 `cost_dashboard` 查看统计：

```
======================================================================
Model Router 成本监控
======================================================================
今日成本: $0.0234

Model                          Calls     Cost($)     AvgLat(ms)
----------------------------------------------------------------
mimo/mimo-v2.5-pro             42        0.0180      1203.5
nvidia-nim/deepseek-v4-flash   18        0.0024      856.2

影子流量对比记录: 3
======================================================================
```

### 影子流量

1% 抽样同时调用更贵的模型，静默对比质量，不影响生产流量。对比记录保存到 `~/.hermes/model_router_shadow.jsonl`。

### 路由矩阵

路由策略从代码中抽离，支持 JSON 热更新。配置文件：`~/.hermes/routing_matrix.json`

```json
{
  "('classify', 'light')": "cheapest",
  "('code', 'heavy')": "smartest",
  "('complex_reasoning', 'medium')": "smartest"
}
```

8 种任务类型 × 3 级复杂度 → 4 种策略映射（`cheapest` / `balanced` / `smartest` / `auto`），60 秒自动刷新。

### 速率限制

Provider 级令牌桶限流，超限自动降级到备选模型并加入冷却黑名单（默认 10 秒）。

```json
{"name": "rate_limit_status", "arguments": {"provider": "nvidia-nim"}}
```

### 长上下文绕路

| 阈值 | 策略 | 行为 |
|------|------|------|
| > 200K tokens | `long_context_prefer` | 优选 context_window >= 阈值的模型 |
| > 800K tokens | `long_context_bypass` | 直选最大上下文模型，跳过常规评分 |

### 多密钥轮转

- MiMo：3 个 API Key Round-Robin 轮转 + 健康检测
- NVIDIA NIM：5 个 API Key Round-Robin 轮转
- 不健康 Key（失败率 > 50%）自动跳过，5 分钟冷却后重试

### AMA 联动

接收 [Adaptive Multi-Agent](https://github.com/weksbwrx62862/adaptive-multi-agent) 下发的任务权重，动态覆盖路由策略：

| AMA 评分 | 覆盖策略 |
|----------|----------|
| ≤ 3 | `cheapest` |
| 4 - 6 | `auto` |
| ≥ 7 | `smartest` |

## 技术栈

| 类别 | 技术 |
|------|------|
| 语言 | Python 3.10+ |
| 插件框架 | Hermes Plugin SDK >= 2.0.0 |
| 配置解析 | PyYAML |
| HTTP 客户端 | OpenAI SDK (兼容模式) |
| 日志格式 | JSONL |
| 并发模型 | threading + Lock |
| 限流算法 | 令牌桶 (Token Bucket) |
| 路由策略 | 多维评分 (cost/speed/quality) + 时间感知 + 任务分类 |

## 项目结构

```
model-router/
├── plugin.yaml              # 插件声明 (name, version, tools, hooks)
├── __init__.py              # 主入口 + 路由引擎 + Hook/Tool 注册
├── cost_monitor.py          # 成本监控 / 影子流量 / 路由矩阵 / 任务分类器
├── V2.2_IMPROVEMENTS.md     # v2.2 改进说明
├── V2.3_IMPROVEMENTS.md     # v2.3 规划
├── .gitignore
└── README.md
```

运行时数据文件：

```
~/.hermes/
├── config.yaml                    # Hermes 全局配置 (含 providers + 插件配置)
├── .env                           # API Key 环境变量
├── model_router_costs.jsonl       # 成本日志
├── model_router_shadow.jsonl      # 影子流量日志
└── routing_matrix.json            # 路由矩阵配置 (60s 热更新)
```

## 开发指南

### 环境搭建

```bash
git clone https://github.com/weksbwrx62862/model-router.git
cd model-router
pip install -e .
```

### 添加新 Provider

1. 在 `~/.hermes/config.yaml` 的 `providers` 下添加 Provider 配置
2. 在 `__init__.py` 的 `_infer_attrs()` 中添加模型属性推断规则
3. 如需多 Key 轮转，添加环境变量名列表并实现 `_pick_xxx_key()`

### 添加新工具

1. 在 `__init__.py` 中实现 `handle_xxx(args, **kwargs) -> str`
2. 在 `register()` 中调用 `ctx.register_tool()` 注册
3. 在 `plugin.yaml` 的 `provides_tools` 列表中添加工具名

### 修改评分规则

评分参数在 `_DEFAULT_SCORING` 字典中定义，可通过 `config.yaml` 覆盖：

```yaml
plugins:
  model-router:
    scoring:
      off_peak_mimo_bonus: 30
      peak_deepseek_bonus: 25
      task_code_bonus: 15
```

### 测试

通过 Hermes Agent 运行时测试，使用工具调用验证路由行为：

```json
{"name": "model_route", "arguments": {"query": "帮我分析这段代码的性能瓶颈"}}
```

## 路线图

- [x] v2.0 — 时间感知路由 + 多密钥轮转 + 令牌桶限流
- [x] v2.1 — AMA 联动 + 执行反馈 + 降级链
- [x] v2.2 — 成本监控 + 影子流量 + 路由矩阵配置化 + 任务分类器
- [ ] v2.3 — 长上下文自动绕路 + 模型上下文窗口感知 + 长文档评分加成
- [ ] v3.0 — 自动质量评估（影子流量结果自动打分）+ 成本预算告警 + 多租户支持

## 常见问题

**Q: 如何切换路由策略？**

使用 `model_balance` 工具或修改 `config.yaml`：

```yaml
plugins:
  model-router:
    strategy: smartest  # cheapest | fastest | smartest | auto
```

**Q: 所有 Provider 都被限流了怎么办？**

Model Router 内置保底策略：当所有 Provider 都被限流时，选择评分最高的模型继续服务，确保不中断。

**Q: 影子流量会影响生产性能吗？**

不会。影子流量默认 1% 抽样率，且是异步记录，不阻塞主请求路径。

**Q: 路由矩阵修改后需要重启吗？**

不需要。路由矩阵每 60 秒自动刷新，修改 JSON 文件后最多 1 分钟生效。

**Q: 如何查看某个 Provider 的限流状态？**

```json
{"name": "rate_limit_status", "arguments": {"provider": "nvidia-nim"}}
```

**Q: 支持哪些任务类型？**

8 种：`classify` / `extract` / `simple_qa` / `long_doc` / `code` / `math` / `complex_reasoning` / `agent`

## Contributing

1. Fork 本仓库
2. 创建 Feature Branch：`git checkout -b agent/<task-id>-<brief-description>`
3. 提交变更：遵循 [Conventional Commits](https://www.conventionalcommits.org/) 规范
4. 推送分支：`git push origin agent/<task-id>-<brief-description>`
5. 创建 Pull Request

## License

[MIT](https://opensource.org/licenses/MIT)

## Security

- API Key 仅通过环境变量注入，绝不硬编码到代码或日志中
- 日志中的 Key 自动脱敏（`sk-xxx***xxx` 格式）
- `.trae/` 目录已加入 `.gitignore`，禁止提交到仓库
- 如发现安全漏洞，请通过 GitHub Issues 私密报告

## 致谢

- [Hermes Agent](https://github.com/weksbwrx62862/hermes) — 插件运行框架
- [Adaptive Multi-Agent](https://github.com/weksbwrx62862/adaptive-multi-agent) — 任务权重联动
- 路由设计参考：《15 分钟搭建多模型智能路由系统》

<p align="center">
  <em>智能路由，成本与性能的最优解</em>
</p>
