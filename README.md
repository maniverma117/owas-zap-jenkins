# 🔐 ZAP SQL Injection Scan with Jenkins CI/CD Integration

This setup enables automated ZAP scans for SQL injection and other vulnerabilities. It uses:

* **ZAP running as a Docker container**
* **Python scan script**
* **Jenkins pipeline** to automate execution and archive reports

---

### 📁 Directory Structure

```
/var/lib/jenkins/zap-script/
├── grc_zap_sql_scan.py       # Python scan script
└── docker-compose.yml        # ZAP service configuration
```

---

## 🚀 Setup Instructions

### ✅ 1. Place the files

Create the working directory:

```bash
mkdir -p /var/lib/jenkins/zap-script
cd /var/lib/jenkins/zap-script
```

Copy these two files:

#### `zap_sql_scan.py`

```bash

from zapv2 import ZAPv2
import requests
import time
import json

# -------------------------
# CONFIGURATION
# -------------------------
TARGET = 'http://65.0.203.43'
LOGIN_URL = 'http://65.0.203.43/admin/login'
USERNAME = 'atishayjain+20@capitall.io'
PASSWORD = 'Aj@201997'
ZAP_API_KEY = 'test123'
ZAP_PROXY = 'http://localhost:8089'

# -------------------------
# ZAP SETUP
# -------------------------
ZAP = ZAPv2(apikey=ZAP_API_KEY, proxies={'http': ZAP_PROXY, 'https': ZAP_PROXY})

# -------------------------
# Wait for ZAP to be ready
# -------------------------
for i in range(30):
    try:
        if ZAP.core.version:
            print(f"[+] ZAP is ready: version {ZAP.core.version}")
            break
    except:
        print("[-] Waiting for ZAP to start...")
        time.sleep(2)
else:
    print("[-] ZAP did not become ready in time.")
    exit(1)

# -------------------------
# Login through ZAP Proxy
# -------------------------
session = requests.Session()
session.proxies = {'http': ZAP_PROXY, 'https': ZAP_PROXY}

print("[+] Logging in to app...")
login_data = {
    'username': USERNAME,
    'password': PASSWORD
}
resp = session.post(LOGIN_URL, data=login_data)

if 'Welcome' in resp.text or resp.status_code in [200, 302]:
    print("[+] Login likely successful.")
else:
    print("[-] Login may have failed. Status:", resp.status_code)
    print(resp.text[:300])

time.sleep(3)

# -------------------------
# Spider the target
# -------------------------
print('[+] Starting spider scan...')
scan_id = ZAP.spider.scan(TARGET)
while int(ZAP.spider.status(scan_id)) < 100:
    print(f"  Spider progress: {ZAP.spider.status(scan_id)}%")
    time.sleep(2)
print("[+] Spidering complete.")

time.sleep(5)

# -------------------------
# Active scan
# -------------------------
print("[+] Starting active scan...")
ascan_id = ZAP.ascan.scan(TARGET)
while int(ZAP.ascan.status(ascan_id)) < 100:
    print(f"  Scan progress: {ZAP.ascan.status(ascan_id)}%")
    time.sleep(5)
print("[+] Active scan complete.")

# -------------------------
# Save report
# -------------------------
print("[+] Writing report files...")

with open('zap_report.html', 'w') as f:
    f.write(ZAP.core.htmlreport())

with open('zap_alerts.json', 'w') as f:
    json.dump(ZAP.core.alerts(baseurl=TARGET), f, indent=2)

print("[+] Reports saved: zap_report.html, zap_alerts.json")

```

The working Python script that:

* Logs into your app
* Runs spider and active scan via ZAP API
* Saves `zap_report.html` and `zap_alerts.json`

> ✅ Make sure it has correct target URL, login URL, username, password, and ZAP API key.

#### `docker-compose.yml`

```yaml
version: '3.8'

services:
  zap:
    image: ghcr.io/zaproxy/zaproxy:stable
    container_name: zap
    restart: always
    ports:
      - "8089:8089"
    command: >
      zap.sh -daemon -port 8089 -host 0.0.0.0
      -config api.key=test123
      -config api.addrs.addr.name=.*
      -config api.addrs.addr.regex=true
```

---

### ✅ 2. Start ZAP as a background service

```bash
cd /var/lib/jenkins/zap-script
docker compose up -d
```

ZAP will now:

* Run as a background daemon
* Auto-start on server reboot

---

### ✅ 3. Jenkins Pipeline Configuration

Add the following stages to your `Jenkinsfile`:

```groovy
pipeline {
    agent any

    stages {
        stage('Run ZAP Scan') {
            steps {
                sh '''
                rm -f zap_report.html zap_alerts.json
                cp /var/lib/jenkins/zap-script/grc_zap_sql_scan.py .
                python3 grc_zap_sql_scan.py
                '''
            }
        }

        stage('Archive ZAP Reports') {
            steps {
                archiveArtifacts artifacts: 'zap_report.html,zap_alerts.json', fingerprint: true
            }
        }
    }
}
```

---

### ✅ 4. Validate API Access

Ensure ZAP is reachable:

```bash
curl http://localhost:8089/JSON/core/view/version/?apikey=test123
```

Expected output:

```json
{"version":"2.16.1"}
```

---

### 🧪 5. View Reports

After the Jenkins job runs:

* `zap_report.html` will be available in Jenkins under “Archived Artifacts”
* `zap_alerts.json` can be parsed or forwarded to other systems

---

### 🔁 Optional: Enable Auto-Start on Reboot

```bash
sudo systemctl enable docker
# or add manual restart to crontab or rc.local:
@reboot docker compose -f /var/lib/jenkins/zap-script/docker-compose.yml up -d
```

---

## 📌 Requirements

* Python 3.x with `python-owasp-zap-v2.4` installed:

  ```bash
  pip3 install python-owasp-zap-v2.4 requests
  ```
* Docker & Docker Compose installed on the Jenkins host

---

## ✅ Summary

| Component     | Purpose                             |
| ------------- | ----------------------------------- |
| `ZAP Docker`  | Persistent security scanner daemon  |
| Python script | Authenticated spider + active scan  |
| Jenkinsfile   | Automates scan + archives report    |
| Reports       | HTML (for humans), JSON (for tools) |

---

You're fully production-ready with this 👏
