# Docker 結構說明
## 補充文件：Dell PowerStore 監控 Stack

---

## 目錄結構

```
~/idrac-monitoring/
│
├── docker-compose.yml                          # 主要編排檔案
│
├── prometheus/
│   ├── prometheus.yml                          # Prometheus 主設定
│   └── targets.yml                             # 動態 target 清單
│
├── idrac_exporter/
│   └── config.yml                              # iDRAC exporter 設定（帳密）
│
├── ipmi_exporter/
│   └── ipmi_config.yml                         # IPMI exporter 設定
│
├── snmp_exporter/
│   └── snmp.yml                                # SNMP exporter 設定（Juniper）
│
└── powerstore-metrics-exporter/                # Clone 自 GitHub
    ├── build/
    │   └── powerstore-metrics-exporter         # Static compiled binary
    ├── bulk/                                   # Bulk API 暫存目錄
    ├── https/                                  # TLS 憑證目錄（選用）
    ├── config.yml                              # PowerStore 連線設定
    └── Dockerfile                              # 用於 build image
```

---

## docker-compose.yml 完整內容

```yaml
services:

  # ── iDRAC Exporter ──────────────────────────────────────────
  idrac-exporter:
    image: mrlhansen/idrac_exporter:latest
    container_name: idrac-exporter
    restart: unless-stopped
    ports:
      - "9348:9348"
    volumes:
      - ./idrac_exporter/config.yml:/etc/idrac_exporter/config.yml:ro

  # ── IPMI Exporter ───────────────────────────────────────────
  ipmi-exporter:
    image: prometheuscommunity/ipmi-exporter:latest
    container_name: ipmi-exporter
    restart: unless-stopped
    ports:
      - "9290:9290"
    volumes:
      - ./ipmi_exporter/ipmi_config.yml:/config.yml:ro
    privileged: true                            # 存取本機 IPMI 設備需要

  # ── PowerStore Exporter ─────────────────────────────────────
  powerstore-exporter:
    image: powerstore-exporter:latest           # 本地 build（非 Docker Hub）
    container_name: powerstore-exporter
    restart: unless-stopped
    ports:
      - "9010:9010"
    volumes:
      - ./powerstore-metrics-exporter/config.yml:/powerstore_exporter/config.yml:ro

  # ── SNMP Exporter ───────────────────────────────────────────
  snmp-exporter:
    image: prom/snmp-exporter:latest
    container_name: snmp-exporter
    restart: unless-stopped
    ports:
      - "9116:9116"
    volumes:
      - ./snmp_exporter/snmp.yml:/etc/snmp_exporter/snmp.yml:ro

  # ── Prometheus ──────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/targets.yml:/etc/prometheus/targets.yml:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=90d"    # 資料保留 90 天
      - "--web.enable-lifecycle"               # 啟用 /reload API

  # ── Grafana ─────────────────────────────────────────────────
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  prometheus_data:                              # Prometheus 時序資料持久化
  grafana_data:                                 # Grafana 設定持久化
```

---

## Container 說明

| Container | Image | Port | 用途 |
|-----------|-------|------|------|
| idrac-exporter | mrlhansen/idrac_exporter | 9348 | 抓取 Dell iDRAC 用電量、溫度、PSU 狀態 |
| ipmi-exporter | prometheuscommunity/ipmi-exporter | 9290 | 抓取舊型伺服器 IPMI 用電量（如 NX3230） |
| powerstore-exporter | powerstore-exporter（本地） | 9010 | 抓取 PowerStore 硬體狀態與效能指標 |
| snmp-exporter | prom/snmp-exporter | 9116 | 抓取 Juniper 交換器用電量、溫度 |
| prometheus | prom/prometheus | 9090 | 時序資料庫，統一收集所有 exporter 資料 |
| grafana | grafana/grafana | 3000 | 視覺化儀表板 |

---

## Prometheus 設定（prometheus.yml）

```yaml
global:
  scrape_interval: 60s
  evaluation_interval: 60s

scrape_configs:

  # Dell iDRAC（12 台伺服器，動態 target）
  - job_name: "idrac"
    metrics_path: /metrics
    params:
      target: ["__address__"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: idrac-exporter:9348
    file_sd_configs:
      - files:
          - /etc/prometheus/targets.yml
        refresh_interval: 30s

  # IPMI（舊型伺服器 NX3230）
  - job_name: "ipmi"
    metrics_path: /metrics
    params:
      module: [default]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: ipmi-exporter:9290
    static_configs:
      - targets:
          - 192.168.50.253

  # PowerStore 硬體狀態
  - job_name: "powerstore"
    static_configs:
      - targets:
          - "192.168.55.4:9010"
    metrics_path: /metrics/192.168.236.80/hardware

  # PowerStore Appliance 效能
  - job_name: "powerstore_appliance"
    static_configs:
      - targets:
          - "192.168.55.4:9010"
    metrics_path: /metrics/192.168.236.80/appliance

  # Juniper 交換器（SNMP）
  - job_name: "snmp_juniper"
    static_configs:
      - targets:
          - 192.168.50.247
    metrics_path: /snmp
    params:
      module: [juniper]
      auth: [public_v2]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 192.168.55.4:9116
```

---

## 各設定檔說明

### idrac_exporter/config.yml
```yaml
username: root
password: calvin
metrics:
  - power
  - thermal
  - memory
  - storage
  - system
```

### ipmi_exporter/ipmi_config.yml
```yaml
modules:
  default:
    user: root
    pass: calvin
    driver: LAN_2_0
    collectors:
      - dcmi
      - ipmi
      - sel
```

### snmp_exporter/snmp.yml
```yaml
auths:
  public_v2:
    community: public
    version: 2

modules:
  juniper:
    walk:
      - 1.3.6.1.4.1.2636.3.1.13.1.5   # 元件名稱
      - 1.3.6.1.4.1.2636.3.1.13.1.20  # 用電量（Watts）
      - 1.3.6.1.4.1.2636.3.1.13.1.21  # 溫度
    metrics:
      - name: juniper_component_power_watts
        oid: 1.3.6.1.4.1.2636.3.1.13.1.20
        type: gauge
        help: Power consumption in Watts
        indexes:
          - labelname: jnxContentsContainerIndex
            type: gauge
          - labelname: jnxContentsL1Index
            type: gauge
          - labelname: jnxContentsL2Index
            type: gauge
          - labelname: jnxContentsL3Index
            type: gauge
      - name: juniper_component_temperature
        oid: 1.3.6.1.4.1.2636.3.1.13.1.21
        type: gauge
        help: Temperature in Celsius
        indexes:
          - labelname: jnxContentsContainerIndex
            type: gauge
          - labelname: jnxContentsL1Index
            type: gauge
          - labelname: jnxContentsL2Index
            type: gauge
          - labelname: jnxContentsL3Index
            type: gauge
```

### powerstore-metrics-exporter/config.yml
```yaml
exporter:
  port: 9010
  reqLimit: 200
  bulkDir: ./bulk/
  bulkCron: "*/5 * * * *"
  https:
    enable: false
log:
  type: logfmt
  path: ./powerstoreExporter.out.log
  level: info
storageList:
  - ip: 192.168.236.80
    user: admin
    password: your_password
    apiVersion: v3
    apiLimit: 5000
    bulkCollector: false
```

---

## 常用管理指令

```bash
# 啟動所有 container
docker compose up -d

# 查看所有 container 狀態
docker compose ps

# 查看特定 container log
docker logs powerstore-exporter --tail 20
docker logs prometheus --tail 20

# 重啟特定 container
docker compose restart powerstore-exporter

# Prometheus 熱重載（不需重啟）
curl -X POST http://192.168.55.4:9090/-/reload

# 新增 iDRAC target（不需重啟）
# 編輯 prometheus/targets.yml 後執行上方 reload 指令

# 停止所有 container
docker compose down
```

---

## 注意事項

| 項目 | 說明 |
|------|------|
| powerstore-exporter image | 需本地 build，無法直接從 Docker Hub pull |
| ipmi-exporter | 需要 `privileged: true` 才能存取 IPMI 設備 |
| Prometheus 資料保留 | 預設 90 天，可依磁碟空間調整 `--storage.tsdb.retention.time` |
| PowerStore Exporter binary | 必須用 static build，否則在 busybox container 內無法執行 |
| 新增 iDRAC 監控目標 | 只需修改 `targets.yml`，Prometheus 每 30 秒自動 reload |
