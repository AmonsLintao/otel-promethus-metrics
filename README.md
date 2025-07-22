---
title: "Prometheus 直接抓取 vs OpenTelemetry 导出：监控架构的选择"
tags:
  - prometheus
  - opentelemetry
  - metrics
  - monitoring
  - observability
head:
  - tag: meta
    attributes:
      name: keywords
      content: "prometheus, opentelemetry, metrics, monitoring, observability"
---

# Prometheus 直接抓取 vs OpenTelemetry 导出：监控架构的选择
> Pair with ClaudeCode, Aider, ClaudeCodeRouter, KimiK2

在实际监控系统设计中，选择合适的 metrics 采集方式是一个非常重要的决策。目前主要有两种方式：**metrics 直接被 Prometheus 抓取**和**通过 OpenTelemetry (OTel) 导出 metrics 再被 Prometheus 抓取**。本文将从架构、灵活性、兼容性、标准化和使用场景等方面进行详细对比分析。

- 你可以去这个地方找到配套的demo跑起来玩：https://github.com/AmonsLintao/otel-promethus-metrics, 包含:
    - 一个模拟发送 metrics 的前端
    - 一个模拟发送 metrics 的后端
    - otel collector + prometheus + grafana

## 架构区别

### Prometheus 直接抓取 metrics

这是最传统、最直接的方式：

- 应用暴露 `/metrics` HTTP 接口（通常使用 Prometheus SDK，例如 Go 的 `prometheus/client_golang`）
- Prometheus 主动抓取该接口（Pull 模式）

**示意图：**

```
[应用程序] --/metrics--> [Prometheus]
```

### OTel 导出 metrics 再被 Prometheus 抓取

现代可观测性架构中更灵活的做法：

- 应用使用 OpenTelemetry SDK 生成指标
- 通过 **OTel Collector 或 OTel SDK exporter** 暴露 `/metrics`
- Prometheus 抓取 OTel Collector 暴露的 metrics 接口

**示意图：**

```
[应用程序] --OTel SDK--> [OTel Collector] --/metrics--> [Prometheus]
```

## 灵活性和扩展性对比

| 方式                | 灵活性  | 说明                                                                       |
| ------------------- | ------- | -------------------------------------------------------------------------- |
| 直接暴露 `/metrics` | ⭐ 普通 | 不支持多种协议、多种后端                                                   |
| 通过 OTel Collector | 🌟 很高 | 可以同时导出到多个系统（Prometheus、OTLP、New Relic、Datadog、logging 等） |

OTel 支持"标准采集 + 分发"，可统一处理 metrics、logs、traces，不仅限于 Prometheus。

## 标准化程度

- **直接 metrics**：Prometheus 的格式是特有的（text-based exposition format）
- **OTel**：采用 OpenTelemetry 的标准（如 OTLP 协议），更偏向于"供应商无关"的可观测性架构

## 性能和开销

| 方式      | 性能     | 备注                                           |
| --------- | -------- | ---------------------------------------------- |
| 直接抓取  | 较高性能 | 少一层转发和抽象                               |
| OTel 采集 | 稍有开销 | 多一层 SDK + Collector，但可控制和缓冲采样数据 |

## 使用场景对比

| 使用场景                         | 推荐方式                   | 原因                 |
| -------------------------------- | -------------------------- | -------------------- |
| 小型系统、单一 Prometheus 环境   | 直接暴露 `/metrics`        | 简单、易用           |
| 多云、多后端、需要导出到多个系统 | OTel + Prometheus exporter | 灵活、统一标准       |
| 想统一指标、日志、链路追踪       | OTel 全家桶                | 架构清晰、可扩展性强 |

## 实际举例
> demo repo: https://github.com/AmonsLintao/otel-promethus-metrics

### 直接暴露 metrics（如 Go 应用）

```go
import "github.com/prometheus/client_golang/prometheus"
```

应用运行后直接在 `/metrics` 提供 Prometheus 可抓的格式。

### OTel 方式

```go
import go.opentelemetry.io/otel
import go.opentelemetry.io/otel/metric
```

然后由 OTel SDK/Collector 负责暴露和转发，甚至可以转为 OTLP，再由 Collector 转为 Prometheus 格式。

## Pull vs Push 模型的区别

需要注意的是，两种方式在数据传输模型上有本质区别：

### Prometheus 模型（Pull-based）

- 应用通过 SDK 暴露一个 HTTP endpoint，通常是 `/metrics`
- Prometheus 按配置的 scrape_interval 定期去拉（pull）这个 endpoint 的内容

### OTel 模型（Push-based）

- 应用使用 OTel SDK 采集 metrics（比如计数器、仪表等）
- 默认通过 OTLP 协议**主动推送（push）**到一个 OTel Collector 或其他接收端

## OTLP 协议详解

当应用使用 OpenTelemetry 推送 metrics 到 OTel Collector 时，它使用的是 **OTLP（OpenTelemetry Protocol）协议**，通过 **gRPC 或 HTTP/Protobuf** 传输。

### Metric 的 payload 格式
> check the frontend metrics send to otel collector

OTLP 是基于 Protocol Buffers（proto3）定义的。metrics 的 payload 是一个 `ExportMetricsServiceRequest` 消息体。

**一个 Gauge Metric 的 JSON 表示形式：**

```json
{
  "resourceMetrics": [
    {
      "resource": {
        "attributes": [
          { "key": "service.name", "value": { "stringValue": "my-app" } }
        ]
      },
      "scopeMetrics": [
        {
          "metrics": [
            {
              "name": "http_requests_total",
              "description": "Total HTTP requests",
              "unit": "1",
              "gauge": {
                "dataPoints": [
                  {
                    "attributes": [
                      { "key": "method", "value": { "stringValue": "GET" } },
                      { "key": "status", "value": { "stringValue": "200" } }
                    ],
                    "timeUnixNano": "1718600000000000000",
                    "asDouble": 158.0
                  }
                ]
              }
            }
          ]
        }
      ]
    }
  ]
}
```

## OTel vs Prometheus 数据模型的差异

**OTel 的 metrics 数据模型比 Prometheus 更丰富**，当通过 `prometheus` exporter 将其导出为 `/metrics` 供 Prometheus 拉取时，**会发生信息丢失或降级**。

### 为什么 OTel 的 metrics 更丰富？

| 特性                                                 | OTel 支持 | Prometheus 支持 | 说明                                                       |
| ---------------------------------------------------- | --------- | --------------- | ---------------------------------------------------------- |
| 时间戳                                               | ✅        | ❌（非原生）    | OTel 的每个 data point 带时间戳，Prometheus 不支持显式指定 |
| 多值聚合指标（Sum, Histogram, ExponentialHistogram） | ✅        | ✅（部分）      | Prometheus 不支持 OTel 的 `ExponentialHistogram`           |
| Metric scope（哪个 SDK 或库生成）                    | ✅        | ❌              | OTel 保留 instrumentation scope 信息                       |
| 资源属性（service.name, host.name 等）               | ✅        | ❌              | Prometheus 不支持结构化资源属性，必须编码到 label          |
| 统一关联 tracing/logging                             | ✅        | ❌              | Prometheus 只处理 metrics，OTel 可以跨 domain              |

### 导出到 Prometheus 时的信息丢失

| OTel 信息             | Prometheus 映射方式或结果                 |
| --------------------- | ----------------------------------------- |
| 时间戳                | **丢弃**，Prometheus 使用 scrape 时间     |
| Scope / Library Name  | **丢弃**                                  |
| Resource 属性         | 可选择作为 label 注入，**部分保留**       |
| ExponentialHistogram  | **无法导出**，需转为普通 histogram 或忽略 |
| Exemplars（trace id） | 通常不导出，或需特殊配置                  |

## 自部署场景的考虑

在自部署 Prometheus + Grafana 的场景中，metrics 通常必须先导出到 Prometheus，Grafana 再通过 Prometheus 数据源读取可视化。这是目前最常见、也是最兼容的方式。

**标准流程：**

```text
[应用] → (OTLP Push) → [OTel Collector]
                            ↓ Prometheus Exporter
                     [Prometheus ← /metrics Scrape]
                            ↓
                     [Grafana ← Prometheus 数据源]
```

### 优化建议

如果希望保留更多 OTel 的语义信息，可以考虑：

1. **注入关键 resource 属性到 label**（通过 OTel Processor 实现）
2. **使用 Grafana Agent** 替代 Prometheus（兼容 Prometheus + OTel）
3. **追加 OTLP exporter** 到其他后端作为扩展

## 总结

| 对比项   | 直接被 Prometheus 抓取 | 经 OTel 导出后抓取           |
| -------- | ---------------------- | ---------------------------- |
| 架构     | 简单直接               | 更灵活、可扩展               |
| 标准     | Prometheus 专有格式    | OTel 标准，支持多后端        |
| 性能     | 更高一些               | 多一层开销                   |
| 可组合性 | 一对一                 | 一对多（OTLP、logs、traces） |
| 推荐场景 | 小项目/只用 Prometheus | 多系统/现代观测平台          |

**选择建议：**

- 如果你只用 Prometheus 做指标采集，直接暴露 metrics 是最简单的方式
- 如果你希望统一处理 metrics、logs、traces，并可能迁移到其他后端系统（如 Grafana Cloud、New Relic、DataDog），那使用 OTel 更具优势
- 在自部署情况下，Prometheus 是必须环节，但仍然可以通过配置 OTel processor 和精心设计来最大限度保留信息
