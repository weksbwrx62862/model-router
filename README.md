# Model Router

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue.svg" alt="Python">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License">
  <img src="https://img.shields.io/badge/Hermes-%3E%3D2.0.0-orange.svg" alt="Hermes">
</p>

多模型智能路由 v2.2 — 时间感知路由 + 成本监控 + 影子流量 + 路由矩阵配置化。

## 核心能力

- 时间感知路由（按时间段自动切换模型池）
- 成本监控面板（实时追踪 token 消耗和费用）
- 影子流量（新模型静默评估）
- 路由矩阵配置化（YAML 声明式路由规则）

## 快速开始

```yaml
plugins:
  - name: model-router
    path: ./model-router
```

## License

MIT