# LLM Dynamo on Azure AKS - Autoscaling with KEDA

## Overview
End-to-end runbook to provision an AKS cluster, deploy the NVIDIA Dynamo inference platform, and enable KEDA autoscaling driven by **Time-to-First-Token (TTFT)** latency metrics from Prometheus.


## Files

| File | Description |
|------|-------------|
| `agg.yaml` | `DynamoGraphDeployment` manifest defining the vLLM aggregated deployment with Frontend and Decode Worker services |
| `autoscale_ttf.yaml` | KEDA `ScaledObject` that autoscales the Decode Worker based on Time-To-First-Token (TTFT) p95 latency |
| `config_prome.yaml` | Prometheus collector `ConfigMap` for scraping metrics from the `dynamo-cloud` namespace |
| `dynamo-keda.ipynb` | Jupyter notebook end to end  |

## Inference Architecture

```
Client → LoadBalancer (port 8000)
           └─ Frontend pods ×2      (vllm-runtime 0.9.0, metrics :8000)
                   └─ VllmDecodeWorker pods ×2–4  (vllm-runtime 0.9.0, metrics :9090)
                              Model: Qwen/Qwen3-0.6B via dynamo.vllm
```


- **Frontend**: 2 replicas, exposes metrics on port `8000`
- **VllmDecodeWorker**: scales between 2–4 replicas based on TTFT p95, exposes metrics on port `9090`
- **Model**: `Qwen/Qwen3-0.6B`
- **Runtime image**: `nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.9.0`

## Scaling Policy

The `ScaledObject` (`autoscale_ttf.yaml`) scales the inference Worker using this Prometheus query:

```promql
histogram_quantile(0.95,
  sum(rate(dynamo_frontend_time_to_first_token_seconds_bucket[2m]))
  by (le)
)
```

## Autoscaling Signal
KEDA polls Prometheus every **30 s**. When the **TTFT p95 > 300 ms** (2-minute rolling window), it scales up `VllmDecodeWorker` replicas from a minimum of **2** to a maximum of **4**. Scale-down cooldown is **120 s**.

| Parameter | Value | Description |
|-----------|-------|-------------|
| Threshold | `0.3` (300ms) | Scale up when TTFT p95 exceeds this value |
| Polling interval | `30s` | How often KEDA checks the metric |
| Cooldown period | `120s` | Wait time before scaling down |
| Min replicas | `2` | Minimum Decode Worker pods |
| Max replicas | `4` | Maximum Decode Worker pods |



## End-to-end runbook
Check `dynamo-keda.ipynb` notebook 


## Prerequisites

- Azure Kubernetes cluster with GPU nodes
- [KEDA](https://keda.sh/docs/latest/deploy/) installed
- [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo) operator installed
- Prometheus stack deployed in the `monitoring` namespace


## Deployment

1. Apply the Prometheus collector configuration:
   ```bash
   kubectl apply -f config_prome.yaml
   ```

2. Deploy the vLLM aggregated graph:
   ```bash
   kubectl apply -f agg.yaml
   ```

3. Apply the KEDA autoscaler:
   ```bash
   kubectl apply -f autoscale_ttf.yaml
   ```

## License

SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.  
SPDX-License-Identifier: Apache-2.0
