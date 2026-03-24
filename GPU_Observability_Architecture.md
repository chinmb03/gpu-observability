# GPU Observability — Architecture & Flow

---

## What is GPU Observability?

GPU Observability is the practice of **continuously collecting, storing, visualizing, and alerting** on GPU hardware metrics in real time — so engineering teams always know the health, utilization, and performance of every GPU in their infrastructure.

---

## The Big Picture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA CENTER / CLOUD                               │
│                                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌───────────┐  │
│   │   GPU Node 1│    │   GPU Node 2│    │   GPU Node 3│    │  GPU Node │  │
│   │  (4x A100)  │    │  (4x A100)  │    │  (4x H100)  │    │    ...    │  │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └─────┬─────┘  │
│          │                  │                  │                  │        │
│          └──────────────────┴──────────────────┴──────────────────┘        │
│                                      │                                      │
│                               Metrics Scrape                                │
│                                      │                                      │
│                             ┌────────▼────────┐                            │
│                             │   Prometheus     │                            │
│                             │  (Time-Series DB)│                            │
│                             └────────┬────────┘                            │
│                                      │                                      │
│               ┌──────────────────────┼──────────────────────┐              │
│               │                      │                      │              │
│        ┌──────▼──────┐      ┌────────▼───────┐    ┌────────▼────────┐     │
│        │   Grafana   │      │  Spring Boot   │    │  Alertmanager   │     │
│        │  Dashboards │      │  Backend APIs  │    │  (Notifications)│     │
│        └─────────────┘      └────────┬───────┘    └─────────────────┘     │
│                                      │                                      │
│                             ┌────────▼────────┐                            │
│                             │  React Frontend │                            │
│                             │  (Live UI)      │                            │
│                             └─────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Layer-by-Layer Breakdown

### Layer 1 — GPU Hardware

```
┌──────────────────────────────────────────────────────┐
│                     GPU Node                         │
│                                                      │
│   ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐   │
│   │ GPU 0  │  │ GPU 1  │  │ GPU 2  │  │ GPU 3  │   │
│   │ A100   │  │ A100   │  │ A100   │  │ A100   │   │
│   └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘   │
│       │           │           │            │        │
│       └─────── NVLink Switch ─────────────┘        │
│                       │                             │
│                    PCIe Bus                         │
│                       │                             │
│               ┌───────▼──────┐                      │
│               │  NVIDIA      │                      │
│               │  Driver      │                      │
│               │  (NVML)      │                      │
│               └───────┬──────┘                      │
│                       │                             │
│               ┌───────▼──────┐                      │
│               │  DCGM Daemon │                      │
│               │ (nv-hostengine)                     │
│               └───────┬──────┘                      │
│                       │                             │
│               ┌───────▼──────┐                      │
│               │dcgm-exporter │  ◄── exposes         │
│               │  :9400       │      /metrics HTTP    │
│               └──────────────┘                      │
└──────────────────────────────────────────────────────┘
```

> **DCGM (Data Center GPU Manager)** sits between the NVIDIA driver and the monitoring stack. It polls hundreds of GPU hardware counters every few seconds and exposes them in Prometheus format.

---

### Layer 2 — Metrics Collection (Prometheus)

```
                    Every 15 seconds
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
  Node 1:9400      Node 2:9400      Node 3:9400
  /metrics         /metrics         /metrics
        │                │                │
        └────────────────┼────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │      Prometheus     │
              │                     │
              │  • Stores metrics   │
              │    as time-series   │
              │  • Supports PromQL  │
              │  • Retention: 30d   │
              │  • Port: 9090       │
              └─────────┬───────────┘
                        │
              ┌─────────▼───────────┐
              │   Long-term Storage │
              │  (Thanos / Cortex)  │
              │  Months of history  │
              │  for capacity plan  │
              └─────────────────────┘
```

---

### Layer 3 — Backend (Spring Boot Microservices)

```
┌────────────────────────────────────────────────────────────────┐
│                    Spring Boot Backend                         │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              API Gateway (Spring Cloud Gateway)          │ │
│  │         Single entry point · Auth · Rate limiting        │ │
│  └───────────────────────┬──────────────────────────────────┘ │
│                          │                                     │
│           ┌──────────────┼──────────────┐                     │
│           │              │              │                      │
│  ┌────────▼────────┐  ┌──▼──────────┐  ┌▼──────────────────┐ │
│  │  GPU Metrics    │  │   Alert     │  │   Config Server   │ │
│  │  Service        │  │   Service   │  │  (Centralised     │ │
│  │                 │  │             │  │   Config)         │ │
│  │ • Queries       │  │ • Evaluates │  │                   │ │
│  │   Prometheus    │  │   thresholds│  └───────────────────┘ │
│  │ • REST API      │  │ • Broadcasts│                        │
│  │ • SSE streaming │  │   WebSocket │                        │
│  └─────────────────┘  └─────────────┘                        │
└────────────────────────────────────────────────────────────────┘
```

---

### Layer 4 — Frontend (React)

```
┌────────────────────────────────────────────────────────────────┐
│                      React Frontend                            │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                     Dashboard                            │ │
│  │                                                          │ │
│  │  ┌─────────────────┐       ┌──────────────────────────┐ │ │
│  │  │   GPU Cards     │       │     Alert Panel          │ │ │
│  │  │  ┌──────┐       │       │                          │ │ │
│  │  │  │GPU 0 │ Util  │       │  🔴 CRITICAL: XID Error  │ │ │
│  │  │  │      │ Temp  │       │  🟠 WARNING:  High Temp  │ │ │
│  │  │  │      │ Power │       │  🟡 INFO:     Low Util   │ │ │
│  │  │  └──────┘ VRAM  │       │                          │ │ │
│  │  │  ┌──────┐       │       └──────────────────────────┘ │ │
│  │  │  │GPU 1 │  ...  │                                    │ │
│  │  │  └──────┘       │       ┌──────────────────────────┐ │ │
│  │  └─────────────────┘       │   NVLink Topology        │ │ │
│  │                            │                          │ │ │
│  │  ┌─────────────────┐       │  GPU0 ←→ GPU1            │ │ │
│  │  │  Time-Series    │       │    ↕         ↕           │ │ │
│  │  │  Charts         │       │  GPU2 ←→ GPU3            │ │ │
│  │  │  (Utilization,  │       │                          │ │ │
│  │  │   Memory, Temp) │       └──────────────────────────┘ │ │
│  │  └─────────────────┘                                    │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

---

## End-to-End Data Flow

```
┌──────────┐     poll      ┌──────────┐    scrape    ┌─────────────┐
│   GPU    │─────────────▶│   DCGM   │─────────────▶│  Prometheus │
│ Hardware │   (NVML API) │  Exporter│  every 15s   │             │
└──────────┘              │  :9400   │              └──────┬──────┘
                          └──────────┘                     │
                                                    PromQL query
                                                           │
                                                  ┌────────▼───────┐
                                                  │  Spring Boot   │
                                                  │  GPU Metrics   │
                                                  │  Service       │
                                                  └────────┬───────┘
                                                           │
                              ┌────────────────────────────┤
                              │                            │
                      ┌───────▼──────┐           ┌────────▼──────┐
                      │  REST API    │           │  SSE Stream   │
                      │  (snapshot)  │           │  (live push   │
                      │              │           │   every 5s)   │
                      └───────┬──────┘           └────────┬──────┘
                              │                            │
                              └────────────┬───────────────┘
                                           │
                                  ┌────────▼────────┐
                                  │  React Frontend │
                                  │                 │
                                  │  • GPU Cards    │
                                  │  • Live Charts  │
                                  │  • Topology Map │
                                  └─────────────────┘


 Alert Path (parallel):

 Prometheus ──▶ Spring Boot Alert Service ──▶ WebSocket (STOMP)
                  (checks thresholds                  │
                   every 15s)                ┌────────▼────────┐
                                             │  React Alert    │
                                             │  Panel          │
                                             │  (live feed)    │
                                             └─────────────────┘
```

---

## Communication Protocols

```
┌─────────────────────────────────────────────────────────────────┐
│                   Protocol Decision Map                         │
│                                                                 │
│                                                                 │
│   GPU Metrics (Live, Continuous)                                │
│   ─────────────────────────────                                 │
│   Spring Boot ─────── SSE ──────────────▶ React                │
│                (Server-Sent Events)                             │
│                Server pushes every 5s                           │
│                One-way stream                                   │
│                Auto-reconnects                                  │
│                                                                 │
│   Alerts (Real-time, Bidirectional)                             │
│   ──────────────────────────────────                            │
│   Spring Boot ──── WebSocket ───────────▶ React                │
│                 (STOMP over SockJS)                             │
│                Full-duplex channel                              │
│                Pub/Sub topics                                   │
│                User can ACK alerts                              │
│                                                                 │
│   Historical Data (On-demand)                                   │
│   ──────────────────────────                                    │
│   React ───── HTTP REST GET ─────────────▶ Spring Boot         │
│               (Request/Response)                                │
│               Chart time-range queries                          │
│               Paginated, cached                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## What Metrics Flow Through the System

```
                        GPU Hardware
                            │
            ┌───────────────┼───────────────┐
            │               │               │
     ┌──────▼──────┐ ┌──────▼──────┐ ┌─────▼───────┐
     │Performance  │ │   Health    │ │ Interconnect│
     │  Metrics    │ │   Metrics   │ │  Metrics    │
     │             │ │             │ │             │
     │ • GPU Util  │ │ • ECC SBE   │ │ • NVLink BW │
     │ • VRAM Used │ │ • ECC DBE   │ │ • PCIe TX   │
     │ • SM Clock  │ │ • XID Error │ │ • PCIe RX   │
     │ • Power (W) │ │ • Temp (°C) │ │ • NVSwitch  │
     │ • Mem BW    │ │ • Retired   │ │   Bandwidth │
     │             │ │   Pages     │ │             │
     └──────┬──────┘ └──────┬──────┘ └─────┬───────┘
            │               │               │
            └───────────────┼───────────────┘
                            │
                     DCGM Exporter
                            │
                        Prometheus
                            │
                    Spring Boot APIs
                            │
                      React Dashboard
```

---

## Alert Severity Flow

```
                         Metric Breaches Threshold
                                   │
                    ┌──────────────▼──────────────┐
                    │     Alert Service            │
                    │    (evaluates every 15s)     │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
       ┌──────▼──────┐    ┌────────▼──────┐    ┌───────▼──────┐
       │    INFO     │    │   WARNING     │    │   CRITICAL   │
       │             │    │               │    │              │
       │ • Low GPU   │    │ • Temp > 80°C │    │ • ECC DBE    │
       │   util for  │    │ • High memory │    │ • XID Error  │
       │   30 mins   │    │   pressure    │    │ • Temp > 85°C│
       │             │    │ • P95 util    │    │ • GPU offline│
       │  → Dashboard│    │   > 85% (SLO) │    │              │
       │    badge    │    │               │    │  → Page on-  │
       │             │    │  → Slack /    │    │    call now  │
       └─────────────┘    │    Email      │    │  → WebSocket │
                          └───────────────┘    │    push      │
                                               └──────────────┘
```

---

## Kubernetes Deployment Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                         │
│                                                                 │
│   monitoring namespace                                          │
│   ┌────────────────────────────────────────────────────────┐   │
│   │                                                        │   │
│   │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │   │
│   │  │  Prometheus  │  │   Grafana    │  │Alertmanager │  │   │
│   │  │  (StatefulSet│  │ (Deployment) │  │(Deployment) │  │   │
│   │  │   1 replica) │  │              │  │             │  │   │
│   │  └──────────────┘  └──────────────┘  └─────────────┘  │   │
│   │                                                        │   │
│   └────────────────────────────────────────────────────────┘   │
│                                                                 │
│   gpu-observability namespace                                   │
│   ┌────────────────────────────────────────────────────────┐   │
│   │                                                        │   │
│   │  ┌──────────────┐  ┌──────────────┐                   │   │
│   │  │ API Gateway  │  │ GPU Metrics  │                   │   │
│   │  │ (Deployment) │  │   Service    │                   │   │
│   │  │  2 replicas  │  │  (Deployment)│                   │   │
│   │  └──────────────┘  │  3 replicas  │                   │   │
│   │                    └──────────────┘                   │   │
│   │  ┌──────────────┐                                     │   │
│   │  │Alert Service │                                     │   │
│   │  │ (Deployment) │                                     │   │
│   │  │  2 replicas  │                                     │   │
│   │  └──────────────┘                                     │   │
│   └────────────────────────────────────────────────────────┘   │
│                                                                 │
│   GPU Worker Nodes  (DaemonSet — 1 pod per node)               │
│   ┌────────────────────────────────────────────────────────┐   │
│   │                                                        │   │
│   │  ┌────────────┐  ┌────────────┐  ┌────────────┐       │   │
│   │  │dcgm-export │  │dcgm-export │  │dcgm-export │  ...  │   │
│   │  │  node-1    │  │  node-2    │  │  node-3    │       │   │
│   │  └────────────┘  └────────────┘  └────────────┘       │   │
│   │                                                        │   │
│   └────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Why Each Component Exists

```
┌─────────────────────┬──────────────────────────────────────────────┐
│ Component           │ Why It's Needed                              │
├─────────────────────┼──────────────────────────────────────────────┤
│ NVIDIA DCGM         │ Only trusted source for deep GPU hardware    │
│                     │ counters — ECC, XID, NVLink, clocks          │
├─────────────────────┼──────────────────────────────────────────────┤
│ dcgm-exporter       │ Bridges DCGM to Prometheus format            │
│                     │ DaemonSet ensures every GPU node is covered  │
├─────────────────────┼──────────────────────────────────────────────┤
│ Prometheus          │ Stores time-series, enables range queries,   │
│                     │ multi-node federation, PromQL language        │
├─────────────────────┼──────────────────────────────────────────────┤
│ Spring Boot         │ Business logic layer — threshold evaluation, │
│ Microservices       │ API abstraction, auth, SSE/WebSocket mgmt    │
├─────────────────────┼──────────────────────────────────────────────┤
│ API Gateway         │ Single host for React to talk to, handles   │
│                     │ CORS, auth, rate-limiting, circuit-breaking  │
├─────────────────────┼──────────────────────────────────────────────┤
│ React + SSE         │ Live dashboard without polling overhead;     │
│                     │ browser auto-reconnects on network drop      │
├─────────────────────┼──────────────────────────────────────────────┤
│ WebSocket (STOMP)   │ Instant alert delivery to UI; future:        │
│                     │ user ACK, suppress, escalate from UI         │
├─────────────────────┼──────────────────────────────────────────────┤
│ Alertmanager        │ Deduplicates, groups, silences, and routes   │
│                     │ alerts to PagerDuty / Slack / email          │
├─────────────────────┼──────────────────────────────────────────────┤
│ Grafana             │ Long-term trend dashboards, cross-team       │
│                     │ visibility, SLO burn-rate charts             │
└─────────────────────┴──────────────────────────────────────────────┘
```

---

## Summary Flow (One Line Per Layer)

```
  GPU Hardware
      │  generates raw signals (utilization, temp, memory, errors)
      ▼
  DCGM Daemon
      │  collects via NVML driver API every few seconds
      ▼
  dcgm-exporter
      │  exposes as Prometheus /metrics endpoint on :9400
      ▼
  Prometheus
      │  scrapes all GPU nodes, stores time-series, evaluates rules
      ▼
  Spring Boot Services
      │  query Prometheus via PromQL, apply business logic, stream data
      ▼
  React Frontend
      │  SSE for live GPU cards, WebSocket for real-time alerts
      ▼
  On-call Engineer
        gets notified → investigates → resolves
```
