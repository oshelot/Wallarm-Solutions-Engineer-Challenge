Wallarm Solutions Engineer Challenge

This project implements a local, self-hosted Wallarm Filtering Node running in Docker, connected to a simple backend (httpbin) and tested with GoTestWAF attack simulation traffic.
The goal was to deploy, verify, and document a working Wallarm environment that detects and blocks malicious requests.

🧩 1️⃣ Deploy a Wallarm Filtering Node

Deployment choice: Docker Compose (local, offline-friendly)
Reasoning: Easy to replicate, minimal infrastructure, full control over environment variables and logs.

Steps performed

Created a new Compose stack with three services:

wallarm-node: Wallarm NGINX-based filtering node

httpbin: simple HTTP backend target

gotestwaf: on-demand container for traffic simulation

Configured environment variables via compose/.env:

WALLARM_API_TOKEN=<Deploy token>
WALLARM_API_HOST=audit.api.wallarm.com
WALLARM_LABELS=group=local-lab
WALLARM_MODE=block
NGINX_BACKEND=httpbin


Verified connectivity and successful registration in the Wallarm Console → Settings → Nodes → Regular Nodes.

Troubleshooting highlight
Initially, the node didn’t appear in the UI. I suspected a token type issue, but the real cause was an incorrect API host.
Documentation listed:

us1.api.wallarm.com – US Cloud

api.wallarm.com – EU Cloud
My tenant used my.audit.wallarm.com, so the correct endpoint was audit.api.wallarm.com.
Updating WALLARM_API_HOST resolved registration immediately.

🌐 2️⃣ Set Up a Backend Origin

Backend: kennethreitz/httpbin

Configuration

Added as a Compose service, exposed internally on port 80.

The Wallarm node forwards traffic to it via NGINX_BACKEND=httpbin.

Verified reachability:

curl -i http://localhost:8080/get


Result: HTTP 200 response confirms Wallarm proxy → backend path is healthy.

🚦 3️⃣ Generate Traffic Using GoTestWAF

Tool: Wallarm GoTestWAF

Execution: containerized via Compose network.

Monitoring vs. Blocking

In monitoring, all requests returned 200 → no blocks detected.

After switching to blocking mode, requests such as

curl -i 'http://localhost:8080/get?user=admin%27%20OR%20%271%27=%271'


returned 403, confirming attack blocking.

Report Generation

Executed:

docker compose run --rm gotestwaf \
  --url=http://wallarm-node:80 \
  --blockStatusCodes 403,406,429 \
  --reportFormat html,pdf,csv \
  --reportPath /app/reports \
  --reportName gotestwaf-2025-10-27-0836 \
  --includePayloads \
  --noEmailReport


Reports generated:

reports/
├── gotestwaf-2025-10-27-0836.csv
├── gotestwaf-2025-10-27-0836.html
└── gotestwaf-2025-10-27-0836.pdf


Findings

WAF pre-check detected blocking (blocked=true, HTTP 403).

REST traffic protected; GraphQL/gRPC endpoints skipped (not present).

Attacks surfaced in Wallarm Console’s Events → Attacks dashboard.

🧠 4️⃣ Document Your Process
Overview Summary

A self-contained local Docker deployment replicates a typical on-prem Wallarm installation.
It demonstrates:

Node registration and heartbeat with Wallarm Cloud

Real-time detection and blocking of simulated OWASP attacks

Local report generation and validation workflow

Key Issues & Resolutions
Issue	Root Cause	Resolution
Node not visible in Console	Wrong WALLARM_API_HOST	Updated to audit.api.wallarm.com
No blocks in GoTestWAF	Node stuck in monitoring	Fixed docker-compose.yml to load .env variable
“Connection refused” from GoTestWAF	Race condition / network timing	Added healthcheck and dependency logic
Report creation failed	No mounted /app/reports	Added volume mapping to Compose
📸 Screenshots
Description	File
Node registered in Console	docs/node.png
Attacks view (monitoring)	docs/monitoring.png
Attacks view (blocking)	docs/blocked.png
Endpoint confirmation	docs/endpoint.png
✅ Summary

This challenge successfully demonstrates a fully functional Wallarm filtering node, deployed locally with Docker, connected to Wallarm Cloud (audit.api.wallarm.com), and verified with GoTestWAF.
Blocking mode operated as expected, generating valid reports and attack telemetry visible in the dashboard.
