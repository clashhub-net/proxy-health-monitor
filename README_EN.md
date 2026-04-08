# Proxy Health Monitor

> Node health checking, failover strategies and rotation management

[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

**[Chinese](README.md)** | **[English](README_EN.md)**

---

## 🩺 Quick Health Check

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
        echo "[FAIL] $node - Timeout" | tee -a $LOG_FILE
    fi
}

# Test your proxy nodes
check_node "socks5://127.0.0.1:1080"
check_node "socks5://127.0.0.1:1081"
check_node "socks5://127.0.0.1:1082"
```

---

## 🔄 Failover Configuration

### Clash URL-Test (Auto Select)

```yaml
# Add to Clash config
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

### Clash Fallback (Auto Switch)

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

## 📡 Speed Testing

```bash
#!/bin/bash
# Compare download speeds across nodes

TEST_URL="http://cachefly.cachefly.net/10mb.test"

for port in 1080 1081 1082; do
    echo "--- Port $port ---"
    curl -x socks5://127.0.0.1:$port -o /dev/null -s \
         -w "Speed: %{speed_download} B/s | Time: %{time_total}s\n" \
         --max-time 30 $TEST_URL
done
```

---

## 📊 Monitoring with Prometheus

```python
#!/usr/bin/env python3
# proxy_exporter.py

from prometheus_client import start_http_server, Gauge
import subprocess, time

proxy_latency = Gauge('proxy_node_latency', 'Latency', ['node'])
proxy_status = Gauge('proxy_node_up', 'Status', ['node'])

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

## 🎯 Best Practices

| Practice | Description |
|----------|-------------|
| **URL-Test interval** | 300s (5min) for production, 60s for critical |
| **Tolerance** | 50ms to prevent constant switching |
| **Fallback chain** | Always have 2-3 backup nodes |
| **Health check URL** | Use `gstatic.com/generate_204` (fast, reliable) |
| **DNS check** | Verify DNS isn't leaking via `dnsleaktest.com` |

---

## 🔗 Related Resources

- [ClashVIP](https://clashvip.net) - Clash configuration guides
- [ClashHub](https://clashhub.net) - Proxy rules and lists
- [VPSVIP](https://vpsvip.net) - VPS recommendations

---

## 📄 License

MIT License
