# Model Router

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License">
  <img src="https://img.shields.io/badge/Hermes-%3E%3D2.0.0-orange.svg" alt="Hermes">
  <img src="https://img.shields.io/badge/version-2.2.0-blue.svg" alt="Version">
</p>

多模型智能路由 v2.2 — 时间感知路由 + 成本监控 + 影子流量 + 路由矩阵配置化，在成本和性能间自动取得平衡。

## 核心能力

- **时间感知路由**：按时间段自动切换模型池，高峰用廉价模型，低谷用顶级模型
- **成本监控**：实时追踪 token 消耗和费用，提供 `cost_dashboard` 工具
- **影子流量**：新模型静默评估，不影响生产流量
- **路由矩阵**：YAML 声明式路由规则，无需改代码
- **速率限制**：`rate_limit_status` 检查当前模型配额状态

## 安装

### 前置条件

- Python 3.10+
- [Hermes Agent](https://github.com/weksbwrx62862/hermes) >= 2.0.0

### 从源码安装

```bash
git clone https://github.com/weksbwrx62862/model-router.git
cd model-router
pip install -e .
```

### 依赖

```bash
pip install pyyaml requests
```

## 使用

### Hermes 插件模式

```yaml
# hermes_config.yaml
plugins:
  - name: model-router
    path: ./model-router
```

### 路由规则配置

```yaml
# route_rules.yaml
routes:
  - name: daytime-default
    time_range: "08:00-20:00"
    priority: [deepseek-v4-flash, gpt-4o-mini]
  - name: nighttime-premium
    time_range: "20:00-08:00"
    priority: [gpt-4o, deepseek-v4-pro]
```

## 提供的工具

| 工具 | 功能 |
|------|------|
| `model_route` | 根据任务和预算选择最佳模型 |
| `model_pool` | 查看可用模型池 |
| `model_balance` | 查询当前余额/配额 |
| `time_info` | 获取当前时间段信息 |
| `model_route_for_cron` | 定时任务专用路由 |
| `rate_limit_status` | 速率限制状态查询 |
| `cost_dashboard` | 成本仪表盘 |

## 提供的钩子

| 钩子 | 说明 |
|------|------|
| `pre_llm_call` | LLM 调用前自动路由 |
| `post_llm_call` | LLM 调用后统计成本 |

## 项目结构

```
model-router/
├── plugin.yaml              # 插件声明
├── __init__.py              # 主入口 + 路由引擎
├── cost_monitor.py          # 成本监控面板
├── V2.2_IMPROVEMENTS.md     # v2.2 改进说明
└── V2.3_IMPROVEMENTS.md     # v2.3 规划
```

## 开发

```bash
git clone https://github.com/weksbwrx62862/model-router.git
cd model-router
pip install -e .
# 通过 Hermes 运行时测试
```

## License

MIT