# Wallarm Solutions Engineer Challenge

This project implements a **local, self-hosted Wallarm node** using Docker Compose, a simple backend (`httpbin`), and **GoTestWAF** to simulate attack traffic and generate local reports.

---

## 1. Deploy a Wallarm Filtering Node

**Deployment Choice:** Docker Compose (local)  
**Why:** Simple, reproducible, and provides full control over configuration and logs without external exposure.

### Services

- `wallarm-node` – Wallarm NGINX-based filtering node  
- `httpbin` – lightweight backend API  
- `gotestwaf` – on-demand container for attack simulation and reporting

### Environment Variables (`compose/.env`)

```bash
WALLARM_API_TOKEN=<Deploy token>
WALLARM_API_HOST=audit.api.wallarm.com
WALLARM_LABELS=group=local-lab
WALLARM_MODE=block
NGINX_BACKEND=httpbin
```

### Start Services
```bash
cd compose
docker compose up -d httpbin wallarm-node
```

### Verify Node Registration
In the Wallarm Console, go to:

```
Settings → Nodes → Regular Nodes
```

You should see your node with the label `group=local-lab`.

---

## 2. Set Up a Backend Origin

**Backend:** [`kennethreitz/httpbin`](https://hub.docker.com/r/kennethreitz/httpbin)

### Verify Connectivity
```bash
curl -i http://localhost:8080/get
```

**Expected Output:**
```
HTTP/1.1 200 OK
Content-Type: application/json
```

This confirms traffic passes through the Wallarm proxy to the backend.

---

## 3. Generate Traffic Using GoTestWAF

### Monitoring vs Blocking Mode
- **Monitoring:** All requests return 200; GoTestWAF won’t detect a WAF.  
- **Blocking:** Requests like SQL injection payloads return 403/406/429 and are logged as attacks.

### Manual Block Test
```bash
# URL-encoded quotes
curl -i 'http://localhost:8080/get?user=admin%27%20OR%20%271%27=%271'
# Expect 403/406/429 in block mode
```

### Run GoTestWAF and Save Local Reports
```bash
docker compose run --rm gotestwaf \
  --url=http://wallarm-node:80 \
  --blockStatusCodes 403,406,429 \
  --reportFormat html,pdf,csv \
  --reportPath /app/reports \
  --reportName gotestwaf-2025-10-27-0836 \
  --includePayloads \
  --noEmailReport
```

### Reports Generated
```
reports/
├── gotestwaf-2025-10-27-0836.csv
├── gotestwaf-2025-10-27-0836.html
└── gotestwaf-2025-10-27-0836.pdf
```

### Findings
- WAF pre-check returned `blocked=true` with HTTP 403.  
- REST traffic was successfully intercepted.  
- Attacks appeared in the Wallarm Console under **Events → Attacks**.

---

## 4. Document Your Process

### Overview Summary
This local Docker deployment demonstrates:

- Successful node registration with the correct API host (`audit.api.wallarm.com`)
- Inline attack blocking in **block** mode
- Local, reproducible report generation using **GoTestWAF**

### Issues and Resolutions

| Issue | Root Cause | Resolution |
|-------|-------------|------------|
| Node not visible in Console | Wrong API host | Tenant URL was `my.audit.wallarm.com`; updated to `WALLARM_API_HOST=audit.api.wallarm.com`. |
| No blocking despite `.env` | Hard-coded mode in Compose file | Updated Compose to use `WALLARM_MODE: "${WALLARM_MODE:-monitoring}"` and recreated container. |
| GoTestWAF connection refused | Node not ready / network timing | Added healthcheck and `depends_on`. |
| Report write error | Missing volume mapping | Added `../reports:/app/reports` and set `--reportPath /app/reports`. |

---

## 5. Additional API Discovery
I wanted to see what API discovery would look like with a sample application like crAPI. A seperate docker enviornment was spun up where we proxied API traffic through a Wallarm node. Discovery results captured in a screenshot

## Screenshots

| Description | File |
|--------------|------|
| Node registered | `docs/node.png` |
| Attacks (monitoring) | `docs/monitoring.png` |
| Attacks (blocking) | `docs/blocked.png` |
| Proxy/endpoint test | `docs/endpoint.png` |
| cpAPI API Discovery | `docs/crapi-discovery.png` |

---

## Summary

This challenge demonstrates a **fully functional Wallarm Docker node** protecting a backend and integrated with Wallarm Cloud for telemetry.  
In **block mode**, GoTestWAF confirms real attack blocking and report generation, with artifacts stored under `reports/`.
