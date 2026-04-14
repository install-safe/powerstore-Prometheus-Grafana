# Dell PowerStore 用開源工具打造企業級即時儀表板與 REST API 踩坑記
## Prometheus + Grafana + Metrics Exporter 完整整合實錄

---

## 前言

身為 IT 維運人員，你是否曾經面臨這樣的情境：Dell PowerStore 儲存設備運行中，卻只能透過 PowerStore Manager 網頁介面手動確認狀態，無法與其他基礎設施整合到同一個監控平台？

我們也遇到了同樣的問題。在一次基礎設施監控統一化的專案中，我們嘗試將 Dell PowerStore 1000T 的運作狀態整合進現有的 Prometheus + Grafana 監控 stack。這篇文章完整記錄了整個過程——包括踩到的坑、找到的解法，以及最終建立出的儀表板。

希望這篇文章能讓後來的工程師少走一些彎路。

---

## 環境說明

| 項目 | 內容 |
|------|------|
| 儲存設備 | Dell PowerStore 1000T |
| PowerStore IP | 192.168.236.80 |
| 監控主機 | Ubuntu 22.04（192.168.55.4） |
| 監控工具 | Prometheus + Grafana + Docker |

---

## 架構概覽

整合後的資料流如下：

```
Dell PowerStore
      ↓ REST API
PowerStore Metrics Exporter（Port 9010）
      ↓ /metrics/{IP}/{category}
Prometheus（Port 9090）
      ↓ 資料來源
Grafana（Port 3000）
      ↓
即時儀表板
```

這個架構完全基於開源工具，不需要額外購買商業監控軟體授權。

---

## 第一坑：REST API 認證

### 問題描述

一開始我們以為 PowerStore REST API 和一般 API 一樣，直接帶上帳密就能呼叫：

```bash
# 這樣行不通！
curl -sk -u admin:'Password123!' \
  "https://192.168.236.80/api/rest/metrics_query" \
  -H "Accept: application/json"
```

結果一直收到這個錯誤：

```json
{
  "messages": [{
    "severity": "Error",
    "code": "0xE09040010001",
    "message_l10n": "You do not have the required authorization to perform this operation."
  }]
}
```

明明帳號是 Administrator 權限，卻一直回傳授權失敗。

### 根本原因

查閱 [Dell PowerStore REST API Developers Guide](https://www.dell.com/support/manuals/zh-tw/powerstoreOS) 後才發現：

> PowerStore API 使用 **cookie-based 認證**，且 POST/PATCH/DELETE 操作前必須先透過 GET 請求取得 **CSRF token（DELL-EMC-TOKEN）**。

這和一般 REST API 的 Basic Auth 完全不同，文件中明確說明：

> *Before issuing any REST call which changes the state of the object (such as POST, PATCH or DELETE) send a GET request to receive a CSRF token as response header named DELL-EMC-TOKEN.*

### 正確的認證流程

**Step 1：先用 GET 請求取得 cookie 和 CSRF token**

```bash
curl -sk -c /tmp/pst_cookie.txt \
  -u admin:'Password123!' \
  -X GET \
  "https://192.168.236.80/api/rest/appliance?select=id" \
  -H "Accept: application/json" \
  -D /tmp/pst_headers.txt

# 從 response header 取出 CSRF token
TOKEN=$(grep -i "DELL-EMC-TOKEN" /tmp/pst_headers.txt | awk '{print $2}' | tr -d '\r')
```

**Step 2：帶上 cookie 和 CSRF token 呼叫 API**

```bash
curl -sk -b /tmp/pst_cookie.txt \
  "https://192.168.236.80/api/rest/metrics/generate" \
  -H "Content-Type: application/json" \
  -H "DELL-EMC-TOKEN: $TOKEN" \
  -X POST \
  -d '{"entity":"performance_metrics_by_appliance","entity_id":"A1","interval":"Five_Mins"}'
```

這樣就能正確取得 Appliance 效能指標了。

---

## 第二坑：API 端點路徑錯誤

解決認證問題後，我們嘗試呼叫 metrics API，卻又碰壁：

```bash
# 這個路徑不存在！
curl ... "https://192.168.236.80/api/rest/metrics_query"
# 回傳：No API path found that matches request
```

### 解法

透過解析 PowerStore 提供的 OpenAPI 文件找到正確路徑：

```bash
curl -sk -b /tmp/pst_cookie.txt \
  "https://192.168.236.80/api/rest/openapi.json" \
  | python3 -c "
import json,sys
d=json.load(sys.stdin)
print([k for k in d['paths'].keys() if 'metric' in k.lower()])
"
```

正確的 metrics 端點是：

```
POST /api/rest/metrics/generate
```

而非 `/api/rest/metrics_query`。

---

## 第三坑：用電量資料不在 REST API 裡

這是最令人意外的發現。我們原本期望透過 REST API 取得 PowerStore 的即時用電量（Watts），但仔細檢查 `MetricsEntityEnum` 的所有可用 entity 後確認：

**PowerStore REST API 的 metrics 端點完全不提供用電量資料。**

可查詢的 entity 只包含：

| Entity | 說明 |
|--------|------|
| performance_metrics_by_appliance | IOPS、延遲、頻寬 |
| performance_metrics_by_volume | Volume 效能 |
| space_metrics_by_appliance | 容量使用率 |
| copy_metrics_by_appliance | 複製效能 |

如果需要用電量資料，替代方案有：
- **iDRAC**：若 PowerStore controller node 有獨立 iDRAC，可透過 Redfish API 取得
- **智慧型 PDU**：從機架 PDU 直接抓取整體用電量
- **SNMP**：PowerStore 的 SNMP 僅支援 Trap（被動通知），不支援主動 poll

---

## 解決方案：Dell 官方開源 Metrics Exporter

就在我們準備自己撰寫爬蟲腳本時，找到了 Dell 官方維護的開源工具：

**[Metrics Exporter for Dell PowerStore](https://github.com/dell/powerstore-metrics-exporter)**

這個工具透過 PowerStore REST API 收集指標，並以 Prometheus 格式對外暴露，完全符合我們的需求。

### 可收集的指標類別

| 端點路徑 | 說明 |
|---------|------|
| `/metrics/{IP}/hardware` | PSU、風扇、電池、硬碟健康狀態 |
| `/metrics/{IP}/appliance` | IOPS、延遲、頻寬、CPU 使用率 |
| `/metrics/{IP}/volume` | Volume 效能 |
| `/metrics/{IP}/capacity` | 容量使用率 |
| `/metrics/{IP}/cluster` | Cluster 資訊 |
| `/metrics/{IP}/port` | 網路埠效能 |
| `/metrics/{IP}/nas` | NAS 效能 |
| `/metrics/{IP}/file` | File System 效能 |

---

## 部署步驟

### 前置條件

- Ubuntu 22.04
- Docker + Docker Compose
- Go 1.18+（用於編譯）

### Step 1：Clone Repo

```bash
git clone https://github.com/dell/powerstore-metrics-exporter.git
cd powerstore-metrics-exporter
```

### Step 2：編譯（注意！必須用 Static Build）

這是另一個坑：直接用 `go build` 編譯出來的 binary 是動態連結，無法在 Dockerfile 使用的 busybox container 內執行，會出現：

```
exec /powerstore_exporter/powerstore-metrics-exporter: no such file or directory
```

必須改用 static build：

```bash
mkdir -p build bulk https
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -a -ldflags '-extldflags "-static"' \
  -o build/powerstore-metrics-exporter .

# 確認是 static 版本
file build/powerstore-metrics-exporter
# 應該顯示：statically linked
```

### Step 3：設定 config.yml

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
  - ip: 192.168.236.80        # PowerStore 管理 IP
    user: admin
    password: your_password
    apiVersion: v3            # PowerStore OS 3.6+ 建議用 v3
    apiLimit: 5000
    bulkCollector: false      # PowerStore OS 4.1+ 需設為 true
```

> **注意**：建議在 PowerStore 建立專用的 operator 帳號，不要直接使用 admin。

### Step 4：Build Docker Image

```bash
docker build --no-cache -t powerstore-exporter:latest .
```

### Step 5：加入 docker-compose.yml

```yaml
services:
  powerstore-exporter:
    image: powerstore-exporter:latest
    container_name: powerstore-exporter
    restart: unless-stopped
    ports:
      - "9010:9010"
    volumes:
      - ./powerstore-metrics-exporter/config.yml:/powerstore_exporter/config.yml:ro
```

啟動：

```bash
docker compose up -d powerstore-exporter
```

### Step 6：確認 Exporter 正常運作

```bash
# 確認 hardware 指標
curl -s http://192.168.55.4:9010/metrics/192.168.236.80/hardware | grep -v "^#"

# 確認 appliance 效能指標
curl -s http://192.168.55.4:9010/metrics/192.168.236.80/appliance | grep -v "^#"
```

正常的話會看到類似：

```
powerstore_hardware_Power_Supply_state{IP="192.168.236.80",appliance_id="A1",name="BaseEnclosure-NodeA-PSU0"} 1
powerstore_hardware_Fan_state{IP="192.168.236.80",appliance_id="A1",name="BaseEnclosure-NodeA-FanModule0"} 1
powerstore_perf_avg_read_iops{IP="192.168.236.80",appliance_id="A1",...} 8.6
powerstore_perf_avg_write_latency{IP="192.168.236.80",appliance_id="A1",...} 534.2
```

---

## 整合進 Prometheus

在 `prometheus.yml` 加入兩個 job：

```yaml
scrape_configs:
  - job_name: "powerstore"
    static_configs:
      - targets:
          - "192.168.55.4:9010"
    metrics_path: /metrics/192.168.236.80/hardware

  - job_name: "powerstore_appliance"
    static_configs:
      - targets:
          - "192.168.55.4:9010"
    metrics_path: /metrics/192.168.236.80/appliance
```

不需重啟 Prometheus，直接 reload：

```bash
curl -X POST http://192.168.55.4:9090/-/reload
```

到 `http://192.168.55.4:9090/targets` 確認兩個 job 都是 UP 狀態。

---

## Grafana 儀表板建置

### 可監控的關鍵指標

**Hardware 硬體健康指標（1=正常，0=異常）**

| 指標名稱 | 說明 |
|----------|------|
| `powerstore_hardware_Power_Supply_state` | PSU 電源供應器狀態 |
| `powerstore_hardware_Fan_state` | 風扇狀態 |
| `powerstore_hardware_Battery_state` | 電池（UPS）狀態 |
| `powerstore_hardware_Drive_state` | 硬碟健康狀態 |
| `powerstore_hardware_drive_size` | 硬碟容量（bytes） |

**Appliance 效能指標**

| 指標名稱 | 說明 |
|----------|------|
| `powerstore_perf_avg_read_iops` | 平均讀取 IOPS |
| `powerstore_perf_avg_write_iops` | 平均寫入 IOPS |
| `powerstore_perf_avg_total_iops` | 平均總 IOPS |
| `powerstore_perf_avg_read_latency` | 平均讀取延遲（μs） |
| `powerstore_perf_avg_write_latency` | 平均寫入延遲（μs） |
| `powerstore_perf_avg_read_bandwidth` | 平均讀取頻寬（Bytes/s） |
| `powerstore_perf_avg_write_bandwidth` | 平均寫入頻寬（Bytes/s） |
| `powerstore_perf_avg_io_workload_cpu_utilization` | IO CPU 使用率 |
| `powerstore_appliance` | Appliance 整體健康狀態（0=正常） |

### 儀表板內容

我們建立的 Dashboard（UID：`dell-powerstore-v1`）包含以下 Panel：

**健康狀態區（一眼看出異常）**
- PSU 電源供應器狀態（綠色=正常，紅色=異常）
- 風扇狀態（支援多顆風扇個別顯示）
- 電池狀態
- 正常硬碟數量
- Appliance 整體健康狀態

**效能趨勢區（歷史走勢分析）**
- IOPS 趨勢（Read / Write / Total）
- 延遲趨勢（Read / Write / 平均，單位 μs）
- 頻寬趨勢（Read / Write / Total，單位 Bytes/s）
- IO CPU 使用率

**硬體資訊區**
- 硬碟容量列表（含型號、容量大小）

---

## 結語

這次整合 Dell PowerStore 到 Prometheus + Grafana 的過程，踩到了三個主要的坑：

1. **REST API 認證**：必須使用 cookie-based 認證 + CSRF token，不能只用 Basic Auth
2. **API 端點路徑**：正確的 metrics 端點是 `/api/rest/metrics/generate`
3. **Static Build**：Exporter 的 Go binary 必須靜態編譯才能在 busybox container 運行

最終透過 Dell 官方開源的 [Metrics Exporter](https://github.com/dell/powerstore-metrics-exporter)，成功將 PowerStore 的硬體健康狀態與效能指標整合進現有的監控平台，讓維運團隊不需要開啟多個管理介面，在同一個 Grafana 儀表板就能掌握 PowerStore 的即時狀態。

如果你也在規劃 Dell 儲存設備的監控整合，希望這篇文章能節省你不少時間。

---

## 參考資料

- [Metrics Exporter for Dell PowerStore（GitHub）](https://github.com/dell/powerstore-metrics-exporter)
- [Dell PowerStore REST API Developers Guide](https://www.dell.com/support/manuals/zh-tw/powerstore)
- [Prometheus 官方文件](https://prometheus.io/docs/)
- [Grafana 官方文件](https://grafana.com/docs/)
