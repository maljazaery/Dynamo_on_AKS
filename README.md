# aks-dynamo-guide

End-to-end tutorials for running [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo) on **Azure Kubernetes Service (AKS)**, integrated with **Azure Managed Prometheus** and **Grafana**, showcasing different inference serving architectures and autoscaling strategies for LLM workloads.

---

## Overview

| # | tutorial | Strategy | Status |
|---|------|----------|--------|
| 1 | [Autoscaling Aggregated Serving with KEDA](#1-autoscaling-aggregated-serving-with-keda) | Metric-driven HPA via KEDA + Prometheus TTFT | ✅ Complete |
| 2 | [Autoscaling Disaggregated Serving with Dynamo Planner](#2-autoscaling-disaggregated-serving-with-dynamo-planner) | Disaggregated Prefill/Decode with Dynamo planner | 🚧 In Progress |
| 3 | [KV Cache Routing](#3-kv-cache-routing) | Intelligent KV cache-aware request routing | 🚧 In Progress |

---

## Architecture

All demos share a common base infrastructure on Azure:

```
┌─────────────────────────────────────────────────────────┐
│                    Azure AKS Cluster                    │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │                                                  │   │
│  │                 Dynamo operator                  │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │      Azure GPU Node Pool (ND / NC / NV)          │   │
│  │   Frontend pods  →  Prefill / Decode Workers     │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
         │                          │
         ▼                          ▼
┌─────────────────┐      ┌──────────────────────┐
│  Azure Managed  │      │   Azure Managed      │
│  Prometheus     │────▶ │   Grafana            │
└─────────────────┘      └──────────────────────┘
```


## Demos

### 1. Autoscaling Aggregated Serving with KEDA

**Folder:** [`autoscaling_agg_keda/`](./autoscaling_agg_keda/)

**What it demonstrates:**

Deploys Dynamo in **aggregated serving** mode (combined Prefill + Decode per pod) and autoscales the number of Worker replicas using **KEDA** triggered by **Time-to-First-Token (TTFT)** latency sourced from Prometheus.

**Architecture:**

```
Client → LoadBalancer :8000
           └─ Frontend pods ×2
                   └─ VllmDecodeWorker pods ×2–4  (autoscaled)
```

**Autoscaling Signal:**


When **TTFT p95 > 300 ms**, KEDA scales up Decode Workers from **2 → 4** replicas. Scale-down cooldown is 120 s.


> See [autoscaling_agg_keda/dynamo-keda.ipynb](./autoscaling_agg_keda/dynamo-keda.ipynb) for the full step-by-step guide.

---

### 2. Autoscaling Disaggregated Serving with Dynamo Planner

> **Status: 🚧 In Progress**

**What it demonstrates:**

Deploys Dynamo in **disaggregated serving** mode, separating **Prefill** and **Decode** workers onto dedicated pods. The **Dynamo Planner** continuously monitors queue depths and GPU utilization to dynamically rebalance the Prefill-to-Decode worker ratio without requiring an external autoscaler.

**Architecture:**

```
Client → LoadBalancer :8000
           └─ Frontend pods
                   ├─ Prefill Worker pods  (planner-managed)
                   └─ Decode Worker pods   (planner-managed)
```

**Key concepts:**
- Disaggregated Prefill/Decode for improved GPU memory efficiency
- Dynamo Planner controls replica ratios based on live load signals
- Integration with Azure Managed Prometheus for observability

---

### 3. KV Cache Routing

> **Status: 🚧 In Progress**

**What it demonstrates:**

Implements **KV cache-aware request routing** to maximize KV cache reuse across Decode Worker pods. Requests with overlapping prompt prefixes are steered to the worker that already holds the relevant KV cache blocks, reducing redundant computation and improving throughput.

**Key concepts:**
- Prefix-aware load balancing for LLM inference
- Reduced KV cache eviction under high-concurrency workloads
- Grafana dashboards for cache hit rate and routing efficiency

---




## Contributing

Contributions and issue reports are welcome. Please open a GitHub Issue or Pull Request.

