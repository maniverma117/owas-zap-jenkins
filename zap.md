
# ✅ Final Polished `README.md`

Here’s your updated version with:

* ✅ Clear formatting for code vs explanation
* ✅ Better file name consistency
* ✅ Clarified instructions
* ✅ Minor grammar + markdown improvements

---

```markdown
# 🔐 ZAP SQL Injection Scan with Jenkins CI/CD Integration

This setup enables automated ZAP scans for SQL injection and other vulnerabilities. It uses:

- **ZAP running as a Docker container**
- **Python scan script**
- **Jenkins pipeline** to automate execution and archive reports

---

## 📁 Directory Structure

```

/var/lib/jenkins/zap-script/
├── grc\_zap\_sql\_scan.py       # Python scan script
└── docker-compose.yml        # ZAP service configuration

````

---

## 🚀 Setup Instructions

### ✅ 1. Prepare Files

Create the working directory:

```bash
mkdir -p /var/lib/jenkins/zap-script
cd /var/lib/jenkins/zap-script
````

Add these two files:

#### `grc_zap_sql_scan.py`

This Python script:

* Logs into your web app via form-based authentication
* Runs spider and active scan via ZAP API
* Saves `zap_report.html` and `zap_alerts.json` for auditing

✅ Make sure to update it with your own:

* `TARGET` URL
* `LOGIN_URL`
* `USERNAME` and `PASSWORD`
* `ZAP_API_KEY`

<details>
<summary>Click to expand full script</summary>

```python
from zapv2 import ZAPv2
import requests
import time
import json

TARGET = 'http://65.0.203.43'
LOGIN_URL = 'http://65.0.203.43/admin/login'
USERNAME = 'atishayjain+20@capitall.io'
PASSWORD = 'Aj@201997'
ZAP_API_KEY = 'test123'
ZAP_PROXY = 'http://localhost:8089'

ZAP = ZAPv2(apikey=ZAP_API_KEY, proxies={'http': ZAP_PROXY, 'https': ZAP_PROXY})

# Wait for ZAP to be ready
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

# Login to app via ZAP proxy
session = requests.Session()
session.proxies = {'http': ZAP_PROXY, 'https': ZAP_PROXY}
print("[+] Logging in to app...")

login_data = {'username': USERNAME, 'password': PASSWORD}
resp = session.post(LOGIN_URL, data=login_data)

if 'Welcome' in resp.text or resp.status_code in [200, 302]:
    print("[+] Login likely successful.")
else:
    print("[-] Login may have failed. Status:", resp.status_code)
    print(resp.text[:300])

time.sleep(3)

# Spider the target
print('[+] Starting spider scan...')
scan_id = ZAP.spider.scan(TARGET)
while int(ZAP.spider.status(scan_id)) < 100:
    print(f"  Spider progress: {ZAP.spider.status(scan_id)}%")
    time.sleep(2)

print("[+] Spidering complete.")
time.sleep(5)

# Active scan
print("[+] Starting active scan...")
ascan_id = ZAP.ascan.scan(TARGET)
while int(ZAP.ascan.status(ascan_id)) < 100:
    print(f"  Scan progress: {ZAP.ascan.status(ascan_id)}%")
    time.sleep(5)

print("[+] Active scan complete.")

# Save reports
with open('zap_report.html', 'w') as f:
    f.write(ZAP.core.htmlreport())

with open('zap_alerts.json', 'w') as f:
    json.dump(ZAP.core.alerts(baseurl=TARGET), f, indent=2)

print("[+] Reports saved: zap_report.html, zap_alerts.json")
```

</details>

---

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

### ✅ 2. Start ZAP in Background

```bash
cd /var/lib/jenkins/zap-script
docker compose up -d
```

This will:

* Run ZAP as a daemon
* Restart it on server reboot automatically

---

### ✅ 3. Jenkins Pipeline Configuration

Add this to your Jenkinsfile:

```groovy
pipeline {
    agent any

    environment {
        GITHUB_REPO_BRANCH = "${env.GIT_BRANCH ?: 'main'}"
        GIT_COMMIT_ID = "${env.GIT_COMMIT ?: 'N/A'}"
    }

    stages {
        stage('Initialize Build Metadata') {
            steps {
                script {
                    currentBuild.displayName = "App=[${JOB_BASE_NAME}] Branch=[${GITHUB_REPO_BRANCH}] CommitID=[${GIT_COMMIT_ID}]"
                    currentBuild.description = "Application [${JOB_BASE_NAME}] built from branch [${GITHUB_REPO_BRANCH}]"
                }
            }
        }

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

Test ZAP accessibility:

```bash
curl http://localhost:8089/JSON/core/view/version/?apikey=test123
```

Expected:

```json
{"version":"2.16.1"}
```

---

### 🧪 5. View Reports

After the Jenkins job finishes:

* `zap_report.html`: accessible via **Archived Artifacts**
* `zap_alerts.json`: useful for parsing, alerting, or analysis

---

### 🔁 Optional: Enable ZAP on Boot

To ensure ZAP auto-starts:

```bash
sudo systemctl enable docker
# or crontab/rc.local:
@reboot docker compose -f /var/lib/jenkins/zap-script/docker-compose.yml up -d
```

---

## 📌 Requirements

* Python 3.x with the following packages:

  ```bash
  pip3 install python-owasp-zap-v2.4 requests
  ```

* Docker + Docker Compose installed on the Jenkins host

---

## ✅ Summary

| Component     | Purpose                                  |
| ------------- | ---------------------------------------- |
| ZAP Docker    | Persistent security scanner daemon       |
| Python Script | Authenticated scan via API               |
| Jenkinsfile   | Automates scanning + reporting pipeline  |
| Reports       | HTML (manual view), JSON (machine-parse) |

---

You're now running a secure, automated DevSecOps pipeline. 🚀
Let me know if you want to extend this to:

* Fail builds on high-severity issues
* Notify via Slack/Teams
* Add OWASP Dependency Check too

```
```
