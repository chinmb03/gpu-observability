# GPU Observability with DCGM Exporter, Spring Boot Microservices & React UI

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Why GPU Observability Matters](#2-why-gpu-observability-matters)
3. [What is DCGM?](#3-what-is-dcgm)
4. [Full Architecture Overview](#4-full-architecture-overview)
5. [Setting Up DCGM Exporter](#5-setting-up-dcgm-exporter)
6. [Key Metrics to Monitor](#6-key-metrics-to-monitor)
7. [Spring Boot Microservices Implementation](#7-spring-boot-microservices-implementation)
   - [Project Setup](#71-project-setup)
   - [GPU Metrics Service](#72-gpu-metrics-service)
   - [Alert Service (WebSocket)](#73-alert-service-websocket)
   - [API Gateway](#74-api-gateway-spring-cloud-gateway)
   - [application.yml](#75-applicationyml)
8. [React Frontend Implementation](#8-react-frontend-implementation)
   - [Project Structure](#81-project-structure)
   - [SSE Hook — Live GPU Data](#82-sse-hook--live-gpu-data)
   - [WebSocket Alert Hook](#83-websocket-alert-hook)
   - [Dashboard Page](#84-gpu-dashboard-page)
   - [GPU Card Component](#85-gpu-card-component)
   - [Alert Panel Component](#86-alert-panel-component)
9. [Live Data Visualization — Prometheus & PromQL](#9-live-data-visualization--prometheus--promql)
10. [GPU-to-GPU Communication & NVLink Monitoring](#10-gpu-to-gpu-communication--nvlink-monitoring)
11. [Alerting Rules](#11-alerting-rules)
12. [Data Flow Summary](#12-data-flow-summary)
13. [Best Practices](#13-best-practices)
14. [Interview-Ready Talking Points](#14-interview-ready-talking-points)

---

## 1. Introduction

Modern AI/ML workloads — large language model training, inference, computer vision pipelines — are GPU-intensive. A cluster of NVIDIA A100s running a distributed training job can consume tens of thousands of dollars per day. Without observability, you are flying blind: you don't know if your GPUs are actually doing work, if memory is leaking, if thermals are approaching throttle limits, or if expensive NVLink bandwidth is being wasted.

**GPU Observability** is the practice of continuously collecting, storing, visualizing, and alerting on hardware and software metrics from GPUs in real time.

---

## 2. Why GPU Observability Matters

### 2.1 Cost Efficiency
GPUs are the most expensive compute resource in ML infrastructure. A GPU running at 0% utilization is money being burned. Observability tells you:
- Which GPUs are idle vs. saturated
- Whether workloads are batch-scheduled efficiently
- If multi-tenancy is under- or over-provisioned

### 2.2 Performance Debugging
Silent performance degradation is common:
- **Memory fragmentation** causing OOM crashes
- **Thermal throttling** silently reducing clock speeds
- **PCIe/NVLink bottlenecks** starving GPU compute of data
- **Stragglers** in data-parallel training (one slow GPU holds back all others)

### 2.3 Hardware Health & Reliability
GPUs fail. DCGM tracks:
- ECC (error-correcting code) memory errors — soft errors (correctable) and hard errors (uncorrectable, fatal)
- Retired pages — VRAM pages taken offline by the driver due to repeated errors
- XID errors — NVIDIA's GPU error codes (e.g., XID 79 = GPU has fallen off the bus)

### 2.4 Capacity Planning
Aggregate utilization trends over time guide:
- When to purchase more GPUs
- Which GPU SKU is right for upcoming workloads (A100 80GB vs H100 vs L40S)
- Reserved vs. on-demand cloud GPU decisions

### 2.5 SLA / SLO Enforcement
Inference services promise latency SLAs. GPU saturation directly causes latency spikes. Observability closes the loop between infrastructure state and service-level objectives.

---

## 3. What is DCGM?

**DCGM (Data Center GPU Manager)** is NVIDIA's open-source library and daemon for managing and monitoring NVIDIA GPUs in data center environments.

| Component | Role |
|-----------|------|
| `nv-hostengine` | Background daemon that polls GPU hardware via NVML |
| `dcgmi` | CLI tool for querying DCGM interactively |
| `libdcgm` | C library for programmatic access |
| `dcgm-exporter` | Prometheus-compatible HTTP exporter of DCGM metrics |

DCGM sits between the NVIDIA driver (NVML) and your monitoring stack, providing:
- **Field IDs** — hundreds of hardware counters (utilization, memory, clocks, NVLink, PCIe, power, temperature, ECC)
- **Health checks** — automated pass/fail diagnostics per GPU
- **Job statistics** — per-job GPU utilization tracking (useful for multi-tenant clusters)
- **Policy management** — automated actions on error conditions

---

## 4. Full Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          GPU Nodes                                       │
│   NVIDIA GPU → DCGM Daemon → dcgm-exporter (:9400/metrics)             │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │ scrape (Prometheus)
┌─────────────────────────────▼───────────────────────────────────────────┐
│                     Backend (Spring Boot Microservices)                  │
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  GPU Metrics     │  │  Alert           │  │  Auth / Gateway      │  │
│  │  Service         │  │  Service         │  │  (Spring Cloud GW)   │  │
│  │  (REST + SSE)    │  │  (WebSocket)     │  │                      │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │ REST / SSE / WebSocket
┌─────────────────────────────▼───────────────────────────────────────────┐
│                        React Frontend                                    │
│   Dashboard | Live Charts | GPU Topology | Alert Panel                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Microservices Breakdown

| Service | Responsibility | Tech |
|---------|---------------|------|
| `gpu-metrics-service` | Polls Prometheus, exposes REST + SSE | Spring Boot + WebFlux |
| `alert-service` | Evaluates thresholds, sends alerts via WebSocket | Spring Boot + WebSocket |
| `api-gateway` | Single entry point, routing, auth | Spring Cloud Gateway |
| `config-server` | Centralized config | Spring Cloud Config |

---

## 5. Setting Up DCGM Exporter

### 5.1 Standalone (Bare Metal)

```bash
# Install DCGM
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list \
  | sudo tee /etc/apt/sources.list.d/nvidia-container.list

sudo apt-get update && sudo apt-get install -y datacenter-gpu-manager

# Start the DCGM host engine
sudo systemctl start nvidia-dcgm

# Run dcgm-exporter
docker run -d \
  --gpus all \
  --cap-add SYS_ADMIN \
  -p 9400:9400 \
  nvcr.io/nvidia/k8s/dcgm-exporter:3.3.0-3.2.0-ubuntu22.04
```

Access metrics at: `http://localhost:9400/metrics`

### 5.2 Kubernetes (DaemonSet via Helm)

```bash
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update

helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  --namespace monitoring \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.interval=15s
```

### 5.3 Custom Metrics Configuration (`default-counters.csv`)

```csv
# Format: FieldId, PromType, Help
DCGM_FI_DEV_GPU_UTIL,                gauge,   GPU utilization (%)
DCGM_FI_DEV_MEM_COPY_UTIL,           gauge,   Memory bandwidth utilization (%)
DCGM_FI_DEV_FB_FREE,                 gauge,   Framebuffer memory free (MiB)
DCGM_FI_DEV_FB_USED,                 gauge,   Framebuffer memory used (MiB)
DCGM_FI_DEV_GPU_TEMP,                gauge,   GPU temperature (°C)
DCGM_FI_DEV_POWER_USAGE,             gauge,   Power draw (W)
DCGM_FI_DEV_SM_CLOCK,                gauge,   SM clock frequency (MHz)
DCGM_FI_DEV_ECC_SBE_VOL_TOTAL,       counter, ECC single-bit errors (volatile)
DCGM_FI_DEV_ECC_DBE_VOL_TOTAL,       counter, ECC double-bit errors (volatile)
DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL,  counter, NVLink total bandwidth (MB/s)
DCGM_FI_DEV_PCIE_TX_THROUGHPUT,      counter, PCIe TX throughput (KB/s)
DCGM_FI_DEV_PCIE_RX_THROUGHPUT,      counter, PCIe RX throughput (KB/s)
DCGM_FI_DEV_XID_ERRORS,              counter, XID error count
```

---

## 6. Key Metrics to Monitor

| Metric | DCGM Field | What to Watch |
|--------|------------|---------------|
| **GPU Utilization** | `DCGM_FI_DEV_GPU_UTIL` | Should be >80% during training; 0% = idle waste |
| **Memory Free** | `DCGM_FI_DEV_FB_FREE` | Low free memory = OOM risk |
| **Memory Used** | `DCGM_FI_DEV_FB_USED` | Watch for monotonic growth = memory leak |
| **GPU Temperature** | `DCGM_FI_DEV_GPU_TEMP` | A100 throttles at ~83°C; alert at 80°C |
| **Power Usage** | `DCGM_FI_DEV_POWER_USAGE` | Near TDP = fully loaded; far below = underutilized |
| **SM Clock** | `DCGM_FI_DEV_SM_CLOCK` | Drop in clock = thermal throttling |
| **NVLink Bandwidth** | `DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL` | Saturation signals all-reduce bottleneck |
| **PCIe Throughput** | `DCGM_FI_DEV_PCIE_TX/RX_THROUGHPUT` | Low = CPU↔GPU data starvation |
| **ECC Errors (SBE)** | `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL` | Rising rate = degrading VRAM |
| **ECC Errors (DBE)** | `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL` | Any DBE = immediate action needed |
| **XID Errors** | `DCGM_FI_DEV_XID_ERRORS` | Non-zero = driver/hardware fault |
| **Retired Pages** | `DCGM_FI_DEV_RETIRED_DBE` | > threshold = RMA the GPU |

---

## 7. Spring Boot Microservices Implementation

### 7.1 Project Setup

#### Maven Parent POM

```xml
<!-- pom.xml (parent) -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.3</version>
</parent>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

### 7.2 GPU Metrics Service

#### Dependencies

```xml
<!-- gpu-metrics-service/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

#### Model Class

```java
// GpuMetrics.java
@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class GpuMetrics {
    private String nodeId;
    private int gpuIndex;
    private String gpuName;
    private Instant timestamp;

    // Core metrics
    private double utilizationPercent;
    private double memoryUsedMib;
    private double memoryFreeMib;
    private double temperatureCelsius;
    private double powerWatts;
    private double smClockMhz;

    // Health metrics
    private long eccSingleBitErrors;
    private long eccDoubleBitErrors;
    private long xidErrors;

    // Interconnect
    private double nvlinkBandwidthMBps;
    private double pcieTxKBps;
    private double pcieRxKBps;

    // Derived
    public double getMemoryUtilizationPercent() {
        double total = memoryUsedMib + memoryFreeMib;
        return total > 0 ? (memoryUsedMib / total) * 100 : 0;
    }
}
```

#### Prometheus Query Client

```java
// PrometheusQueryClient.java
@Component
public class PrometheusQueryClient {

    private final WebClient webClient;

    public PrometheusQueryClient(@Value("${prometheus.base-url}") String baseUrl) {
        this.webClient = WebClient.builder()
                .baseUrl(baseUrl)
                .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                .build();
    }

    public Mono<Double> queryScalar(String promql) {
        return webClient.get()
                .uri(uriBuilder -> uriBuilder
                        .path("/api/v1/query")
                        .queryParam("query", promql)
                        .build())
                .retrieve()
                .bodyToMono(PrometheusResponse.class)
                .map(response -> response.getData().getResult())
                .filter(results -> !results.isEmpty())
                .map(results -> Double.parseDouble(results.get(0).getValue()[1]))
                .onErrorReturn(0.0);
    }

    public Mono<List<double[]>> queryRange(String promql, Instant start, Instant end, String step) {
        return webClient.get()
                .uri(uriBuilder -> uriBuilder
                        .path("/api/v1/query_range")
                        .queryParam("query", promql)
                        .queryParam("start", start.getEpochSecond())
                        .queryParam("end", end.getEpochSecond())
                        .queryParam("step", step)
                        .build())
                .retrieve()
                .bodyToMono(PrometheusRangeResponse.class)
                .map(this::extractTimeSeries);
    }
}
```

#### Metrics Aggregation Service

```java
// GpuMetricsAggregatorService.java
@Service
@Slf4j
public class GpuMetricsAggregatorService {

    private final PrometheusQueryClient prometheusClient;

    public Flux<GpuMetrics> fetchAllGpuMetrics(String nodeId) {
        String baseQuery = String.format("{instance=\"%s\"}", nodeId);

        return prometheusClient
                .queryVector("DCGM_FI_DEV_GPU_UTIL" + baseQuery)
                .flatMapMany(results -> Flux.fromIterable(results))
                .flatMap(result -> buildGpuMetrics(nodeId, result));
    }

    private Mono<GpuMetrics> buildGpuMetrics(String nodeId, VectorResult result) {
        int gpuIndex = Integer.parseInt(result.getMetric().get("gpu"));
        String gpuName = result.getMetric().getOrDefault("modelName", "Unknown");
        String gpuFilter = String.format("{instance=\"%s\",gpu=\"%d\"}", nodeId, gpuIndex);

        return Mono.zip(
                prometheusClient.queryScalar("DCGM_FI_DEV_GPU_UTIL" + gpuFilter),
                prometheusClient.queryScalar("DCGM_FI_DEV_FB_USED" + gpuFilter),
                prometheusClient.queryScalar("DCGM_FI_DEV_FB_FREE" + gpuFilter),
                prometheusClient.queryScalar("DCGM_FI_DEV_GPU_TEMP" + gpuFilter),
                prometheusClient.queryScalar("DCGM_FI_DEV_POWER_USAGE" + gpuFilter),
                prometheusClient.queryScalar("DCGM_FI_DEV_SM_CLOCK" + gpuFilter),
                prometheusClient.queryScalar("DCGM_FI_DEV_ECC_DBE_VOL_TOTAL" + gpuFilter),
                prometheusClient.queryScalar("DCGM_FI_DEV_XID_ERRORS" + gpuFilter)
        ).map(tuple -> GpuMetrics.builder()
                .nodeId(nodeId)
                .gpuIndex(gpuIndex)
                .gpuName(gpuName)
                .timestamp(Instant.now())
                .utilizationPercent(tuple.getT1())
                .memoryUsedMib(tuple.getT2())
                .memoryFreeMib(tuple.getT3())
                .temperatureCelsius(tuple.getT4())
                .powerWatts(tuple.getT5())
                .smClockMhz(tuple.getT6())
                .eccDoubleBitErrors(tuple.getT7().longValue())
                .xidErrors(tuple.getT8().longValue())
                .build()
        );
    }
}
```

#### REST + SSE Controller

```java
// GpuMetricsController.java
@RestController
@RequestMapping("/api/v1/gpu")
@CrossOrigin(origins = "${frontend.allowed-origins}")
public class GpuMetricsController {

    private final GpuMetricsAggregatorService metricsService;

    // Snapshot of all GPUs on a node
    @GetMapping("/nodes/{nodeId}/metrics")
    public Flux<GpuMetrics> getNodeMetrics(@PathVariable String nodeId) {
        return metricsService.fetchAllGpuMetrics(nodeId);
    }

    // Time-series for charts
    @GetMapping("/nodes/{nodeId}/gpu/{gpuIndex}/history")
    public Mono<List<double[]>> getGpuHistory(
            @PathVariable String nodeId,
            @PathVariable int gpuIndex,
            @RequestParam(defaultValue = "DCGM_FI_DEV_GPU_UTIL") String metric,
            @RequestParam(defaultValue = "1h") String range,
            @RequestParam(defaultValue = "30s") String step) {
        String promql = String.format("%s{instance=\"%s\",gpu=\"%d\"}", metric, nodeId, gpuIndex);
        Instant end = Instant.now();
        Instant start = end.minus(parseDuration(range));
        return metricsService.queryRange(promql, start, end, step);
    }

    // SSE: live streaming every 5 seconds
    @GetMapping(value = "/nodes/{nodeId}/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<List<GpuMetrics>>> streamMetrics(@PathVariable String nodeId) {
        return Flux.interval(Duration.ofSeconds(5))
                .flatMap(tick -> metricsService.fetchAllGpuMetrics(nodeId).collectList())
                .map(metrics -> ServerSentEvent.<List<GpuMetrics>>builder()
                        .id(String.valueOf(System.currentTimeMillis()))
                        .event("gpu-metrics")
                        .data(metrics)
                        .build())
                .doOnError(e -> log.error("SSE stream error for node {}: {}", nodeId, e.getMessage()));
    }

    // SSE: live NVLink topology
    @GetMapping(value = "/nodes/{nodeId}/nvlink/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<NvLinkTopology>> streamNvLinkTopology(@PathVariable String nodeId) {
        return Flux.interval(Duration.ofSeconds(10))
                .flatMap(tick -> metricsService.fetchNvLinkTopology(nodeId))
                .map(topology -> ServerSentEvent.<NvLinkTopology>builder()
                        .event("nvlink-topology")
                        .data(topology)
                        .build());
    }
}
```

---

### 7.3 Alert Service (WebSocket)

#### Alert Service

```java
// AlertService.java
@Service
@Slf4j
public class AlertService {

    private final SimpMessagingTemplate messagingTemplate;
    private final GpuMetricsAggregatorService metricsService;

    private static final double TEMP_WARN_THRESHOLD = 80.0;
    private static final double UTIL_LOW_THRESHOLD = 10.0;

    @Scheduled(fixedDelay = 15000)
    public void evaluateAlerts() {
        metricsService.fetchAllNodes()
                .flatMap(nodeId -> metricsService.fetchAllGpuMetrics(nodeId))
                .subscribe(metrics -> {
                    checkTemperature(metrics);
                    checkEccErrors(metrics);
                    checkUtilization(metrics);
                    checkXidErrors(metrics);
                });
    }

    private void checkTemperature(GpuMetrics m) {
        if (m.getTemperatureCelsius() > TEMP_WARN_THRESHOLD) {
            GpuAlert alert = GpuAlert.builder()
                    .severity(m.getTemperatureCelsius() > 85 ? Severity.CRITICAL : Severity.WARNING)
                    .type(AlertType.HIGH_TEMPERATURE)
                    .nodeId(m.getNodeId())
                    .gpuIndex(m.getGpuIndex())
                    .message(String.format("GPU %d on %s at %.1f°C",
                            m.getGpuIndex(), m.getNodeId(), m.getTemperatureCelsius()))
                    .timestamp(Instant.now())
                    .build();
            messagingTemplate.convertAndSend("/topic/alerts", alert);
        }
    }

    private void checkEccErrors(GpuMetrics m) {
        if (m.getEccDoubleBitErrors() > 0) {
            GpuAlert alert = GpuAlert.builder()
                    .severity(Severity.CRITICAL)
                    .type(AlertType.ECC_DOUBLE_BIT_ERROR)
                    .nodeId(m.getNodeId())
                    .gpuIndex(m.getGpuIndex())
                    .message("Double-bit ECC error detected — GPU may need replacement")
                    .timestamp(Instant.now())
                    .build();
            messagingTemplate.convertAndSend("/topic/alerts", alert);
        }
    }

    private void checkXidErrors(GpuMetrics m) {
        if (m.getXidErrors() > 0) {
            GpuAlert alert = GpuAlert.builder()
                    .severity(Severity.CRITICAL)
                    .type(AlertType.XID_ERROR)
                    .nodeId(m.getNodeId())
                    .gpuIndex(m.getGpuIndex())
                    .message(String.format("XID error on GPU %d — driver/hardware fault", m.getGpuIndex()))
                    .timestamp(Instant.now())
                    .build();
            messagingTemplate.convertAndSend("/topic/alerts", alert);
        }
    }
}
```

#### WebSocket Configuration

```java
// WebSocketConfig.java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws/gpu-alerts")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

---

### 7.4 API Gateway (Spring Cloud Gateway)

```yaml
# application.yml (api-gateway)
spring:
  cloud:
    gateway:
      routes:
        - id: gpu-metrics-service
          uri: lb://gpu-metrics-service
          predicates:
            - Path=/api/v1/gpu/**
          filters:
            - StripPrefix=0
            - name: CircuitBreaker
              args:
                name: gpuMetricsCB
                fallbackUri: forward:/fallback/metrics

        - id: alert-service
          uri: lb://alert-service
          predicates:
            - Path=/ws/**
          filters:
            - StripPrefix=0

      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "http://localhost:3000"
            allowedMethods: ["GET", "POST", "OPTIONS"]
            allowedHeaders: "*"
```

---

### 7.5 application.yml

```yaml
# gpu-metrics-service/src/main/resources/application.yml
spring:
  application:
    name: gpu-metrics-service

server:
  port: 8081

prometheus:
  base-url: http://prometheus:9090

frontend:
  allowed-origins: http://localhost:3000

management:
  endpoints:
    web:
      exposure:
        include: health,prometheus,info
  metrics:
    export:
      prometheus:
        enabled: true
```

---

## 8. React Frontend Implementation

### 8.1 Project Structure

```
src/
├── api/
│   ├── gpuMetricsApi.ts        # REST + SSE calls
│   └── alertWebSocket.ts       # STOMP WebSocket client
├── components/
│   ├── GpuCard.tsx             # Per-GPU metric card
│   ├── UtilizationGauge.tsx    # Gauge chart (recharts)
│   ├── MemoryBar.tsx           # Memory bar
│   ├── TimeSeriesChart.tsx     # Line chart for history
│   ├── NvLinkTopology.tsx      # GPU-to-GPU topology graph
│   └── AlertPanel.tsx          # Live alert feed
├── hooks/
│   ├── useGpuStream.ts         # SSE hook
│   └── useAlertSocket.ts       # WebSocket hook
├── pages/
│   └── Dashboard.tsx
└── store/
    └── gpuStore.ts             # Zustand store
```

### Install Dependencies

```bash
npm install @stomp/stompjs sockjs-client recharts zustand axios
npm install -D @types/sockjs-client
```

---

### 8.2 SSE Hook — Live GPU Data

```typescript
// hooks/useGpuStream.ts
import { useState, useEffect, useRef } from 'react';
import { GpuMetrics } from '../types/gpu';

export function useGpuStream(nodeId: string) {
  const [metrics, setMetrics] = useState<GpuMetrics[]>([]);
  const [connected, setConnected] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const eventSourceRef = useRef<EventSource | null>(null);

  useEffect(() => {
    if (!nodeId) return;

    const url = `/api/v1/gpu/nodes/${nodeId}/stream`;
    const eventSource = new EventSource(url);
    eventSourceRef.current = eventSource;

    eventSource.addEventListener('gpu-metrics', (event) => {
      try {
        const data: GpuMetrics[] = JSON.parse(event.data);
        setMetrics(data);
        setConnected(true);
        setError(null);
      } catch (e) {
        setError('Failed to parse GPU metrics');
      }
    });

    eventSource.onerror = () => {
      setConnected(false);
      setError('SSE connection lost — reconnecting...');
    };

    return () => {
      eventSource.close();
    };
  }, [nodeId]);

  return { metrics, connected, error };
}
```

---

### 8.3 WebSocket Alert Hook

```typescript
// hooks/useAlertSocket.ts
import { useState, useEffect } from 'react';
import { Client } from '@stomp/stompjs';
import SockJS from 'sockjs-client';
import { GpuAlert } from '../types/alert';

export function useAlertSocket() {
  const [alerts, setAlerts] = useState<GpuAlert[]>([]);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    const client = new Client({
      webSocketFactory: () => new SockJS('/ws/gpu-alerts'),
      onConnect: () => {
        setConnected(true);
        client.subscribe('/topic/alerts', (message) => {
          const alert: GpuAlert = JSON.parse(message.body);
          setAlerts(prev => [alert, ...prev].slice(0, 100));
        });
      },
      onDisconnect: () => setConnected(false),
      reconnectDelay: 5000,
    });

    client.activate();
    return () => { client.deactivate(); };
  }, []);

  return { alerts, connected };
}
```

---

### 8.4 GPU Dashboard Page

```tsx
// pages/Dashboard.tsx
import React, { useState } from 'react';
import { useGpuStream } from '../hooks/useGpuStream';
import { useAlertSocket } from '../hooks/useAlertSocket';
import GpuCard from '../components/GpuCard';
import AlertPanel from '../components/AlertPanel';
import NvLinkTopology from '../components/NvLinkTopology';

export default function Dashboard() {
  const [selectedNode, setSelectedNode] = useState('gpu-node-1');
  const { metrics, connected, error } = useGpuStream(selectedNode);
  const { alerts } = useAlertSocket();

  return (
    <div className="dashboard">
      <header>
        <h1>GPU Observability</h1>
        <span className={`status ${connected ? 'live' : 'disconnected'}`}>
          {connected ? '● LIVE' : '○ Reconnecting...'}
        </span>
        <NodeSelector value={selectedNode} onChange={setSelectedNode} />
      </header>

      {error && <Banner type="error" message={error} />}

      <SummaryRow metrics={metrics} />

      <div className="gpu-grid">
        {metrics.map(gpu => (
          <GpuCard key={`${gpu.nodeId}-${gpu.gpuIndex}`} gpu={gpu} />
        ))}
      </div>

      <NvLinkTopology nodeId={selectedNode} />

      <AlertPanel alerts={alerts} />
    </div>
  );
}
```

---

### 8.5 GPU Card Component

```tsx
// components/GpuCard.tsx
import React from 'react';
import { GpuMetrics } from '../types/gpu';

interface Props { gpu: GpuMetrics; }

export default function GpuCard({ gpu }: Props) {
  const tempColor = gpu.temperatureCelsius > 80 ? '#ef4444'
                  : gpu.temperatureCelsius > 70 ? '#f97316' : '#22c55e';

  return (
    <div className={`gpu-card ${gpu.xidErrors > 0 ? 'gpu-card--error' : ''}`}>
      <div className="gpu-card__header">
        <span className="gpu-name">GPU {gpu.gpuIndex} — {gpu.gpuName}</span>
        <span className="gpu-node">{gpu.nodeId}</span>
      </div>

      <div className="gpu-card__metrics">
        <MetricGauge
          label="GPU Util"
          value={gpu.utilizationPercent}
          max={100}
          unit="%"
          color={gpu.utilizationPercent < 10 ? '#f97316' : '#6366f1'}
        />

        <div className="metric">
          <label>VRAM</label>
          <MemoryBar used={gpu.memoryUsedMib} free={gpu.memoryFreeMib} />
          <span>
            {gpu.memoryUsedMib.toFixed(0)} / {(gpu.memoryUsedMib + gpu.memoryFreeMib).toFixed(0)} MiB
          </span>
        </div>

        <div className="metric">
          <label>Temp</label>
          <span style={{ color: tempColor, fontWeight: 'bold' }}>
            {gpu.temperatureCelsius.toFixed(1)} °C
          </span>
        </div>

        <div className="metric">
          <label>Power</label>
          <span>{gpu.powerWatts.toFixed(0)} W</span>
        </div>

        <div className="health-badges">
          {gpu.eccDoubleBitErrors > 0 && <Badge type="critical" label="ECC DBE" />}
          {gpu.xidErrors > 0 && <Badge type="critical" label="XID Error" />}
          {gpu.eccSingleBitErrors > 0 && <Badge type="warning" label={`SBE: ${gpu.eccSingleBitErrors}`} />}
        </div>
      </div>
    </div>
  );
}
```

---

### 8.6 Alert Panel Component

```tsx
// components/AlertPanel.tsx
import React from 'react';
import { GpuAlert, Severity } from '../types/alert';

const severityStyles: Record<Severity, string> = {
  CRITICAL: 'alert--critical',
  WARNING:  'alert--warning',
  INFO:     'alert--info',
};

export default function AlertPanel({ alerts }: { alerts: GpuAlert[] }) {
  return (
    <div className="alert-panel">
      <h2>Live Alerts <span className="badge">{alerts.length}</span></h2>
      <div className="alert-list">
        {alerts.length === 0 && <p className="no-alerts">No active alerts</p>}
        {alerts.map((alert, i) => (
          <div key={i} className={`alert-item ${severityStyles[alert.severity]}`}>
            <span className="alert-severity">{alert.severity}</span>
            <span className="alert-message">{alert.message}</span>
            <span className="alert-node">GPU {alert.gpuIndex} @ {alert.nodeId}</span>
            <span className="alert-time">{new Date(alert.timestamp).toLocaleTimeString()}</span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 9. Live Data Visualization — Prometheus & PromQL

### Prometheus Scrape Config

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'dcgm-exporter'
    scrape_interval: 15s
    static_configs:
      - targets: ['gpu-node-1:9400', 'gpu-node-2:9400']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
```

### Useful PromQL Queries

**Average GPU utilization across all GPUs:**
```promql
avg(DCGM_FI_DEV_GPU_UTIL) by (instance, gpu)
```

**Memory utilization percentage:**
```promql
(DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE)) * 100
```

**Power efficiency proxy:**
```promql
DCGM_FI_DEV_GPU_UTIL / DCGM_FI_DEV_POWER_USAGE
```

**NVLink bandwidth rate:**
```promql
rate(DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL[1m])
```

**ECC double-bit error detection:**
```promql
increase(DCGM_FI_DEV_ECC_DBE_VOL_TOTAL[5m]) > 0
```

**Grafana Official Dashboard:** Import ID **12239** from grafana.com

---

## 10. GPU-to-GPU Communication & NVLink Monitoring

In a 4-GPU A100 NVSwitch topology, NVLink bandwidth between GPUs looks like:
- GPU 0 → GPU 1: ~11,104 MB/s
- GPU 0 → GPU 2: ~5,348 MB/s
- GPU 2 → GPU 3: ~5,945 MB/s

This matters for **distributed training** because:
- All-reduce collectives (PyTorch DDP, Megatron-LM) move gradients across all GPUs every training step
- NVLink (600 GB/s on A100 NVSwitch) is ~10x faster than PCIe (64 GB/s)
- Asymmetric NVLink bandwidth indicates topology-aware scheduling isn't being used

### Monitor Per-Lane NVLink:

```promql
# DCGM fields for per-lane bandwidth:
# DCGM_FI_DEV_NVLINK_BANDWIDTH_L0 through L17

rate(DCGM_FI_DEV_NVLINK_BANDWIDTH_L0{instance="gpu-node-1",gpu="0"}[1m])
```

Alert if any lane shows zero while the GPU is active — indicates a dead NVLink lane that silently halves all-reduce performance.

---

## 11. Alerting Rules

```yaml
# alertmanager-rules.yml
groups:
  - name: gpu-alerts
    rules:
      - alert: GPUHighTemperature
        expr: DCGM_FI_DEV_GPU_TEMP > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "GPU {{ $labels.gpu }} on {{ $labels.instance }} is at {{ $value }}°C"

      - alert: GPUDoubleBitECCError
        expr: increase(DCGM_FI_DEV_ECC_DBE_VOL_TOTAL[5m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Double-bit ECC error on GPU {{ $labels.gpu }} — hardware failure imminent"

      - alert: GPULowUtilization
        expr: avg_over_time(DCGM_FI_DEV_GPU_UTIL[30m]) < 10
        for: 30m
        labels:
          severity: info
        annotations:
          summary: "GPU {{ $labels.gpu }} has been under 10% utilization for 30 minutes"

      - alert: GPUXIDError
        expr: increase(DCGM_FI_DEV_XID_ERRORS[5m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "XID error detected on GPU {{ $labels.gpu }}"

      - alert: GPUInferenceSaturation
        expr: quantile_over_time(0.95, DCGM_FI_DEV_GPU_UTIL[5m]) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "GPU {{ $labels.gpu }} P95 utilization > 85% — inference latency SLO at risk"
```

---

## 12. Data Flow Summary

```
DCGM Exporter → Prometheus
                    ↓
        Spring Boot WebFlux (PrometheusQueryClient)
                    ↓
        GpuMetricsAggregatorService
           ↙                    ↘
  REST Controller            Alert Service
  SSE /stream                (threshold eval every 15s)
       ↓                          ↓
  React useGpuStream         WebSocket STOMP
  EventSource hook           /topic/alerts
       ↓                          ↓
  GpuCard components         AlertPanel component
  (live re-render)           (live alert feed)
```

---

## 13. Best Practices

| Practice | Why |
|----------|-----|
| Use 15s scrape interval | Balances freshness vs. cardinality cost; use 5s only for debugging |
| Label by job/pod | Use DCGM's job stats API to attribute utilization per ML job, not just per GPU |
| Monitor retired pages weekly | Retired pages grow silently; GPU should be RMA'd before they cause OOM |
| Track SM clock vs. base clock ratio | Ratio < 0.9 = thermal or power throttle |
| Topology-aware placement | Always co-locate communicating ranks on NVLink-connected GPUs |
| Enable MIG observability | For A100/H100 with Multi-Instance GPU, DCGM exports per-MIG-instance metrics |
| Store long-term in Thanos/Cortex | GPU health trends for capacity planning require months of data |
| Correlate with job scheduler | Join DCGM metrics with Slurm/Kubernetes job IDs for per-workload accounting |
| Use WebFlux (reactive) over MVC | SSE and async Prometheus queries benefit from non-blocking I/O |
| Redis pub/sub for scale-out | For multi-instance SSE fan-out across Spring Boot replicas |

---

