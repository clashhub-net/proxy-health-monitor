# 代理节点健康监控

> 节点检测、故障切换和负载均衡管理

[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

**[English](README_EN.md)**

---

## 🩺 快速健康检测

```bash
#!/bin/bash
# proxy-health-check.sh

TIMEOUT=10
LOG_FILE="health_$(date +%Y%m%d_%H%M).log"

check_node() {
    local node=$1
    local result=$(curl -x "$node" -o /dev/null -s -w "%{http_code}:%{time_total}" \
             --max-time $TIMEOUT https://www.google.com 2>&1)
    
    if [[ "$result" == *"200"* ]]; then
        local latency=$(echo "$result" | cut -d: -f2)
        echo "[OK] $node - ${latency}s" | tee -a $LOG_FILE
    else
        echo "[FAIL] $node - 超时" | tee -a $LOG_FILE
    fi
}

# 测试你的代理节点
check_node "socks5://127.0.0.1:1080"
check_node "socks5://127.0.0.1:1081"
check_node "socks5://127.0.0.1:1082"
```

---

## 🔄 故障切换配置

### Clash URL-Test（自动选节点）

```yaml
# 添加到 Clash 配置
proxy-groups:
  - name: "Auto-Select"
    type: url-test
    proxies:
      - US-East
      - Japan-Tokyo
      - Singapore
      - Germany
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
```

### Clash Fallback（自动故障切换）

```yaml
proxy-groups:
  - name: "Proxy"
    type: fallback
    proxies:
      - Primary
      - Backup-1
      - Backup-2
    url: "http://www.gstatic.com/generate_204"
    interval: 60
```

---

## 📡 测速脚本

```bash
#!/bin/bash
# 多节点速度对比

TEST_URL="http://cachefly.cachefly.net/10mb.test"

for port in 1080 1081 1082; do
    echo "--- 端口 $port ---"
    curl -x socks5://127.0.0.1:$port -o /dev/null -s \
         -w "速度: %{speed_download} B/s | 耗时: %{time_total}s\n" \
         --max-time 30 $TEST_URL
done
```

---

## 📊 Prometheus 监控

```python
#!/usr/bin/env python3
# proxy_exporter.py

from prometheus_client import start_http_server, Gauge
import subprocess, time

proxy_latency = Gauge('proxy_node_latency', '延迟', ['node'])
proxy_status = Gauge('proxy_node_up', '状态', ['node'])

NODES = {'us-east': 1080, 'japan': 1081, 'singapore': 1082}

def check(name, port):
    try:
        r = subprocess.run(
            ['curl', '-x', f'socks5://127.0.0.1:{port}', '-o', '/dev/null',
             '-s', '-w', '%{http_code}:%{time_total}', '--max-time', '10',
             'https://www.google.com'],
            capture_output=True, text=True, timeout=15)
        code, lat = r.stdout.split(':')
        proxy_latency.labels(node=name).set(float(lat))
        proxy_status.labels(node=name).set(1 if code == '200' else 0)
    except:
        proxy_status.labels(node=name).set(0)

if __name__ == '__main__':
    start_http_server(9876)
    while True:
        for n, p in NODES.items():
            check(n, p)
        time.sleep(30)
```

---

## 🎯 最佳实践

| 实践 | 说明 |
|------|------|
| **检测间隔** | 生产环境 300s，关键业务 60s |
| **容差值** | 设置 50ms 避免频繁切换 |
| **备用链** | 至少准备 2-3 个备用节点 |
| **检测地址** | 用 `gstatic.com/generate_204`（快速可靠） |
| **DNS 检测** | 定期访问 `dnsleaktest.com` 确认无泄漏 |

---

## 🔗 相关资源

- [ClashVIP](https://clashvip.net) - Clash 配置教程
- [ClashHub](https://clashhub.net) - 规则和代理列表
- [VPSVIP](https://vpsvip.net) - VPS 推荐

---

## 📄 许可证

MIT 许可证
