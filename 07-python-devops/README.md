# 🐍 Python for DevOps

## Why Python in DevOps?

- Automate repetitive ops tasks (cleanup, reporting, health checks)
- Interact with cloud APIs (Boto3 for AWS, google-cloud-python, azure-sdk)
- Build custom CLI tools
- Parse logs, process YAML/JSON configs
- Write glue code between systems

---

## 📋 Cheatsheet

### Working with Files & Paths
```python
from pathlib import Path

# Pathlib (modern, preferred over os.path)
p = Path("/var/log/app")
p.mkdir(parents=True, exist_ok=True)

for log_file in p.glob("*.log"):
    print(log_file.name, log_file.stat().st_size)

config = Path("config.yaml").read_text()
Path("output.json").write_text('{"key": "value"}')
```

### YAML / JSON
```python
import yaml, json

# Load YAML config
with open("config.yaml") as f:
    config = yaml.safe_load(f)     # ALWAYS use safe_load

# Parse JSON response
import requests
response = requests.get("https://api.example.com/health")
data = response.json()

# Dump back
print(yaml.dump(config, default_flow_style=False))
print(json.dumps(data, indent=2))
```

### Subprocess (Run Shell Commands)
```python
import subprocess

# Preferred: subprocess.run
result = subprocess.run(
    ["kubectl", "get", "pods", "-n", "prod"],
    capture_output=True, text=True, check=True
)
print(result.stdout)

# Run with shell (use sparingly — injection risk)
result = subprocess.run(
    "docker ps | grep myapp",
    shell=True, capture_output=True, text=True
)

# Stream output in real-time
with subprocess.Popen(
    ["kubectl", "logs", "-f", "mypod"],
    stdout=subprocess.PIPE, text=True
) as proc:
    for line in proc.stdout:
        print(line, end="")
```

### AWS with Boto3
```python
import boto3

# EC2
ec2 = boto3.client("ec2", region_name="us-east-1")
instances = ec2.describe_instances(
    Filters=[{"Name": "instance-state-name", "Values": ["running"]}]
)
for r in instances["Reservations"]:
    for i in r["Instances"]:
        name = next((t["Value"] for t in i.get("Tags", []) if t["Key"] == "Name"), "N/A")
        print(f"{i['InstanceId']} | {i['InstanceType']} | {name}")

# S3
s3 = boto3.client("s3")
s3.upload_file("report.csv", "my-bucket", "reports/2024/report.csv")

# SSM Parameter Store
ssm = boto3.client("ssm")
param = ssm.get_parameter(Name="/prod/db/password", WithDecryption=True)
db_pass = param["Parameter"]["Value"]
```

---

## 🛠️ Hands-On Project 1: Multi-Service Health Checker

```python
#!/usr/bin/env python3
"""
health_checker.py
Checks HTTP endpoints and sends Slack alert if any are down.
Usage: python health_checker.py --config services.yaml
"""

import argparse
import sys
import time
from dataclasses import dataclass

import requests
import yaml


@dataclass
class ServiceResult:
    name: str
    url: str
    ok: bool
    status_code: int | None
    response_time_ms: float | None
    error: str | None


def check_service(name: str, url: str, timeout: int = 10) -> ServiceResult:
    try:
        start = time.time()
        resp = requests.get(url, timeout=timeout, allow_redirects=True)
        elapsed = (time.time() - start) * 1000
        return ServiceResult(
            name=name, url=url,
            ok=resp.status_code < 400,
            status_code=resp.status_code,
            response_time_ms=round(elapsed, 2),
            error=None
        )
    except requests.exceptions.Timeout:
        return ServiceResult(name=name, url=url, ok=False,
                             status_code=None, response_time_ms=None, error="Timeout")
    except requests.exceptions.ConnectionError as e:
        return ServiceResult(name=name, url=url, ok=False,
                             status_code=None, response_time_ms=None, error=str(e))


def send_slack_alert(webhook_url: str, failed: list[ServiceResult]) -> None:
    message = {
        "text": "🚨 *Health Check Alert*",
        "blocks": [
            {
                "type": "section",
                "text": {"type": "mrkdwn", "text": "🚨 *Services Down*"}
            },
            *[
                {
                    "type": "section",
                    "text": {
                        "type": "mrkdwn",
                        "text": f"*{s.name}*\n`{s.url}`\nError: {s.error or s.status_code}"
                    }
                }
                for s in failed
            ]
        ]
    }
    requests.post(webhook_url, json=message, timeout=5)


def main():
    parser = argparse.ArgumentParser(description="Service health checker")
    parser.add_argument("--config", default="services.yaml")
    parser.add_argument("--slack-webhook", default=None)
    args = parser.parse_args()

    with open(args.config) as f:
        config = yaml.safe_load(f)

    results = [check_service(s["name"], s["url"]) for s in config["services"]]

    for r in results:
        status = "✅" if r.ok else "❌"
        time_str = f"{r.response_time_ms}ms" if r.response_time_ms else "N/A"
        print(f"{status} {r.name:<20} {r.url:<40} {time_str}")

    failed = [r for r in results if not r.ok]
    if failed:
        print(f"\n{len(failed)} service(s) are DOWN!")
        if args.slack_webhook:
            send_slack_alert(args.slack_webhook, failed)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

**`services.yaml`:**
```yaml
services:
  - name: API Gateway
    url: https://api.example.com/health
  - name: Auth Service
    url: https://auth.example.com/ping
  - name: Database Proxy
    url: http://db-proxy.internal:5432/health
```

---

## 🛠️ Hands-On Project 2: Log Analyzer

Parses application logs, extracts error patterns, and generates a summary report.

📁 See [`scripts/log_analyzer.py`](./scripts/log_analyzer.py)

---

## 🎯 Key Interview Questions

1. **When would you use `subprocess.run` vs `subprocess.Popen`?**  
   `run` for simple commands where you wait for output. `Popen` for streaming output or long-running processes.

2. **How do you securely handle credentials in Python scripts?**  
   Use environment variables, AWS Secrets Manager, or Vault. Never hardcode. Use `python-dotenv` for local dev.

3. **What is Boto3 and how does it authenticate?**  
   AWS SDK for Python. Auth order: env vars → `~/.aws/credentials` → IAM instance role → IAM task role (ECS/K8s).

4. **How do you make a Python script a proper CLI tool?**  
   Use `argparse` or `click`, add a `setup.py`/`pyproject.toml` with an entry point, use `#!/usr/bin/env python3`.

5. **How would you parallelize checking 100 endpoints?**  
   Use `concurrent.futures.ThreadPoolExecutor` (I/O-bound). For CPU-bound: `ProcessPoolExecutor`.
