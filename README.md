# Symplichain Hackathon Submission
**Platform:** SymFlow — Logistics Management  
**Date:** April 2026  
**Stack:** Django · React · Celery · Redis · AWS · PostgreSQL

---

## Table of Contents
1. [Part 1 — Shared Gateway: Rate Limiting Architecture](#part-1)
2. [Part 2 — Mobile Architecture](#part-2)
3. [Part 3 — CI/CD Pipeline with GitHub Actions](#part-3)
4. [Part 4 — Debugging the Monday Photo Upload Outage](#part-4)

---

## Part 1 — Shared Gateway: Rate Limiting Architecture <a name="part-1"></a>

### Problem Summary

25 customers collectively generate **>20 requests/second**, but the upstream external API enforces a hard cap of **3 requests/second**. Naively forwarding requests would cause mass rejections. We need a queuing + rate-limiting layer that is fair, resilient to failures, and observable.

---

### Architecture Overview

```
Customers (25)
  │
  ▼
[Django REST API]  — validates & enqueues requests
  │
  ▼
[Redis (ElastiCache)]  — per-customer queues
  Customer_A: [r1, r2 … r100]
  Customer_B: [r1]
  Customer_C: [r1, r2]
  │
  ▼
[Round-Robin Dispatcher (Celery Beat)]
  │  Picks 1 req from each customer cyclically
  ▼
[Celery Worker Pool]  — rate_limit = "3/s" (global)
  │
  ▼
[External API]  — max 3 req/sec
  │
  ▼
[Result → RDS / Response to Client]
```

---

### Design Decisions

#### 1. Per-Customer Redis Queues

Rather than a single global queue (which lets a high-volume customer like Customer A starve everyone else), each customer gets their own Redis list key: `queue:requests:<customer_id>`. Django enqueues jobs with `rpush`; the dispatcher pops with `lpop`.

#### 2. Token Bucket Rate Limiting (3 req/sec)

A Token Bucket is stored in Redis as an atomic counter. Every 333ms a Celery Beat task refills one token (up to a max of 3). Workers check-and-decrement the bucket before each API call, blocking if empty.

```python
# redis_rate_limiter.py (conceptual)
import redis, time

r = redis.Redis.from_url(settings.REDIS_URL)
BUCKET_KEY = "token_bucket:external_api"
MAX_TOKENS = 3

def acquire_token(timeout=5):
    deadline = time.time() + timeout
    while time.time() < deadline:
        tokens = r.get(BUCKET_KEY)
        if tokens and int(tokens) > 0:
            if r.decr(BUCKET_KEY) >= 0:
                return True   # token acquired
        time.sleep(0.05)
    raise Exception("Rate limit timeout — token unavailable")

# Celery Beat refill task (every 333ms)
@app.task
def refill_token_bucket():
    r.set(BUCKET_KEY, MAX_TOKENS, ex=2)  # atomic reset, 2s expiry
```

#### 3. Fairness via Round-Robin Dispatcher

A Celery Beat task runs every ~333ms, collects all active customer queue keys, and dispatches exactly one request from each in rotation — ensuring no single customer monopolises the allowance.

```python
# dispatcher.py (conceptual)
@app.task
def round_robin_dispatch():
    active = r.smembers("active_customers")
    for customer_id in active:
        raw = r.lpop(f"queue:requests:{customer_id}")
        if raw:
            payload = json.loads(raw)
            call_external_api.apply_async(args=[payload], rate_limit="3/s")
        else:
            r.srem("active_customers", customer_id)
```

#### 4. Failure Handling — Exponential Backoff

If the external API returns a 5xx error or times out, Celery's `autoretry_for` enforces delays of 1s → 2s → 4s → 8s (max 5 retries), preventing thundering-herd on recovery.

```python
@app.task(
    bind=True,
    autoretry_for=(ExternalAPIException,),
    retry_backoff=True,       # exponential: 1s, 2s, 4s, 8s…
    retry_backoff_max=60,     # cap at 60 seconds
    max_retries=5
)
def call_external_api(self, payload):
    response = requests.post(EXTERNAL_API_URL, json=payload, timeout=10)
    if response.status_code >= 500:
        raise ExternalAPIException(f"API error: {response.status_code}")
    return response.json()
```

> **Why this approach?** Using Redis + Celery avoids introducing new technology. The token bucket is O(1) and atomic. Per-customer queues provide fairness guarantees. Exponential backoff protects the external API after outages.

---

### Observability

- **Celery Flower** dashboard for real-time task queue depth and failure rates.
- **CloudWatch Custom Metrics** — publish queue depth per customer every 60s.
- **Alerting** — SNS alert if queue depth > 500 or retry count spikes.

---

## Part 2 — Mobile Architecture: SymFlow Mobile App <a name="part-2"></a>

### Stack Decision: React Native ✅

| Criterion | React Native | Native (Kotlin/Swift) |
|---|---|---|
| Code sharing with web | ✅ Reuse business logic, API clients, state | ❌ Full rewrite |
| Team ramp-up | ✅ Existing React engineers | ❌ Hire Android + iOS devs |
| Performance (logistics ops) | ✅ Adequate for forms, maps, camera | ✅ Marginally better |
| Time-to-market | ✅ Single codebase → faster shipping | ❌ 2× dev time |
| Cost | ✅ Lower | ❌ Higher |

**Verdict:** For a fast-moving logistics startup where the web team already writes React, React Native is the pragmatic choice. Native would only be preferred if intensive real-time graphics or Bluetooth hardware integration were required.

---

### Architecture Overview

```
┌─────────────────────────────────────────┐
│           React Native App              │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │ Screens  │  │  State   │  │  API  │ │
│  │(Driver   │  │(Zustand/ │  │Client │ │
│  │Dashboard,│  │ Redux)   │  │(Axios)│ │
│  │ Upload,  │  └────┬─────┘  └───┬───┘ │
│  │ Manifest)│       │            │     │
│  └──────────┘       ▼            ▼     │
│        ┌─────────────────────────────┐  │
│        │    Shared Business Logic    │  │
│        │  (validation, formatters)   │  │
│        └─────────────────────────────┘  │
└─────────────────────────────────────────┘
          │ HTTPS (JWT auth)
          ▼
  [Django REST API on EC2]
          │
   ┌──────┴────────┐
   │               │
  [RDS]       [S3 + CloudFront]
                   │
              [Celery → Bedrock]
```

---

### Key Screens & Features

#### 1. Driver Dashboard
- Today's manifest with delivery stops — sortable, filterable.
- Tap a stop to expand → address, contact, special notes.
- Large, thumb-friendly action buttons (minimum 48×48dp touch targets).

#### 2. Delivery Confirmation (Low-Friction UX)

Truck drivers are often gloved, rushed, and in low-light conditions. The confirmation flow is designed around these constraints:

- **Photo Upload:** One-tap camera button → auto-upload to S3 → Celery triggers Bedrock for AI validation.
- **Large Swipe Gesture:** "Swipe right to confirm delivery" — far more reliable than checkboxes with gloves.
- **Voice Commands via SymAI:** "Mark delivered" / "Report issue" — integrates with AWS Transcribe + Bedrock, keeping hands free.
- **Offline Mode:** Actions queued locally (AsyncStorage sync queue) and replayed when connectivity restores.

#### 3. Issue Reporting
- Pre-populated issue types (Damaged, Missing, Wrong address) — minimises typing.
- Photo evidence attached directly from the report screen.
- Push notification to dispatcher immediately via FCM.

---

### Performance Optimisations

- **Image compression** on-device before S3 upload (React Native Image Manipulator).
- **FlatList with windowing** for large manifest lists — prevents memory bloat.
- **Background fetch** (Expo TaskManager) to pre-load next day's manifests overnight.
- **CloudFront CDN** for all static assets and map tiles — low-latency even on 3G.

> **SymAI Integration:** Voice command → AWS Transcribe → intent classification via Bedrock → action dispatch to the same Django REST endpoints used by the web app. No separate mobile AI pipeline needed.

---

## Part 3 — CI/CD Pipeline with GitHub Actions <a name="part-3"></a>

### Pipeline Design

```
Developer → git push → GitHub
                           │
              ┌────────────┴────────────┐
              │ push to `staging`       │ push to `main`
              ▼                         ▼
     [staging-deploy.yml]     [production-deploy.yml]
              │                         │
         Run Tests                 Run Tests
              │                         │
         Build React              Build React
              │                         │
         S3 Sync (staging)       S3 Sync (prod)
              │                         │
         SSH → EC2 staging       SSH → EC2 prod
           migrate + restart       migrate + restart
```

---

### `staging-deploy.yml`

```yaml
name: Deploy to Staging

on:
  push:
    branches:
      - staging

env:
  AWS_REGION: ap-south-1
  S3_BUCKET_STAGING: symplichain-frontend-staging
  EC2_HOST_STAGING: ${{ secrets.EC2_HOST_STAGING }}

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: symplichain_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ["5432:5432"]
      redis:
        image: redis:7
        ports: ["6379:6379"]
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set Up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install Python Dependencies
        run: pip install -r requirements.txt

      - name: Run Django Tests
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/symplichain_test
          REDIS_URL: redis://localhost:6379/0
          DJANGO_SETTINGS_MODULE: config.settings.test
        run: python manage.py test --verbosity=2

  deploy:
    name: Deploy to Staging EC2
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Set Up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: frontend/package-lock.json

      - name: Install & Build React Frontend
        working-directory: ./frontend
        run: |
          npm ci
          REACT_APP_ENV=staging npm run build

      - name: Sync Frontend to S3 (Staging)
        run: |
          aws s3 sync frontend/build/ s3://${{ env.S3_BUCKET_STAGING }}/ \
            --delete \
            --cache-control "public,max-age=31536000,immutable"
          aws s3 cp frontend/build/index.html \
            s3://${{ env.S3_BUCKET_STAGING }}/index.html \
            --cache-control "no-cache"

      - name: Invalidate CloudFront Cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_DIST_ID_STAGING }} \
            --paths "/*"

      - name: Deploy Backend to EC2 (Staging)
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST_STAGING }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            set -e
            cd /srv/symplichain
            echo "=== Pulling latest code ==="
            git fetch origin staging
            git reset --hard origin/staging
            echo "=== Installing Python dependencies ==="
            source venv/bin/activate
            pip install -r requirements.txt --quiet
            echo "=== Running database migrations ==="
            python manage.py migrate --noinput
            echo "=== Collecting static files ==="
            python manage.py collectstatic --noinput
            echo "=== Restarting services ==="
            sudo systemctl restart gunicorn
            sudo systemctl restart celery
            sudo systemctl restart celerybeat
            echo "=== Health check ==="
            sleep 5
            curl -f http://localhost:8000/health/ || exit 1
            echo "Staging deployment successful!"
```

---

### `production-deploy.yml`

```yaml
name: Deploy to Production

on:
  push:
    branches:
      - main

jobs:
  test:
    # Same test job as staging (postgres + redis services, Django tests)
    # ... (identical to staging test job above)

  deploy:
    name: Deploy to Production EC2
    needs: test
    runs-on: ubuntu-latest
    environment: production    # Requires manual approval in GitHub UI

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      - name: Install & Build React Frontend
        working-directory: ./frontend
        run: |
          npm ci
          REACT_APP_ENV=production npm run build

      - name: Sync Frontend to S3 (Production)
        run: |
          aws s3 sync frontend/build/ s3://symplichain-frontend-prod/ \
            --delete \
            --cache-control "public,max-age=31536000,immutable"
          aws s3 cp frontend/build/index.html \
            s3://symplichain-frontend-prod/index.html \
            --cache-control "no-cache"

      - name: Invalidate CloudFront Cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_DIST_ID_PRODUCTION }} \
            --paths "/*"

      - name: Deploy Backend to EC2 (Production)
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST_PRODUCTION }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY_PROD }}
          script: |
            set -e
            cd /srv/symplichain
            git fetch origin main
            git reset --hard origin/main
            source venv/bin/activate
            pip install -r requirements.txt --quiet
            python manage.py migrate --noinput
            python manage.py collectstatic --noinput
            sudo systemctl reload gunicorn
            sudo systemctl restart celery celerybeat
            sleep 5
            curl -f https://app.symplichain.com/health/ || exit 1
            echo "Production deployment successful!"
```

---

### Required GitHub Secrets

| Secret Name | Purpose |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user with S3 + CloudFront permissions |
| `AWS_SECRET_ACCESS_KEY` | IAM secret |
| `EC2_HOST_STAGING` / `_PRODUCTION` | EC2 public IP or DNS |
| `EC2_SSH_KEY` / `EC2_SSH_KEY_PROD` | Private key for SSH access |
| `CF_DIST_ID_STAGING` / `_PRODUCTION` | CloudFront distribution IDs |

> **Bonus — Docker:** Containerising Django + Celery with Docker means `pip install` on EC2 becomes `docker pull` — dependencies are baked into the image at build time, guaranteeing environment parity between staging and production.

---

## Part 4 — Debugging the Monday Photo Upload Outage <a name="part-4"></a>

### Data Flow Under Investigation

```
Mobile App
   │ POST /api/v1/deliveries/{id}/photo/
   ▼
[EC2 — Django/Gunicorn]
   │ upload file to S3
   ▼
[S3 Bucket — symplichain-photos]
   │ triggers Celery task
   ▼
[Celery Worker (Redis Queue)]
   │ calls AWS Bedrock (AI validation)
   ▼
[AWS Bedrock]
   │ returns labels / validation result
   ▼
[RDS — PostgreSQL]  ← writes DeliveryPhoto record
```

> **Golden Rule:** Follow the data path — don't guess, bisect. Each layer either processed the request or it didn't.

---

### Step 1 — Did the request reach Django?

SSH into EC2 and inspect Gunicorn/Nginx logs.

```bash
# SSH in
ssh -i symplichain.pem ubuntu@<EC2_IP>

# Check Gunicorn access log — look for the photo endpoint
tail -n 200 /var/log/gunicorn/access.log | grep "photo"

# Check for Django-level errors
tail -n 200 /var/log/gunicorn/error.log

# Also check Nginx (sits in front of Gunicorn)
tail -n 200 /var/log/nginx/error.log
```

**Possible findings:**
- `413 Request Entity Too Large` → Nginx `client_max_body_size` is too small. Fix: increase in `/etc/nginx/sites-available/symplichain`.
- `504 Gateway Timeout` → Gunicorn worker timed out during S3 upload. Fix: make upload async via Celery.
- No log entry at all → Request never reached EC2; check load balancer / security group rules.

---

### Step 2 — Did the file successfully land in S3?

```bash
# Check if files are arriving in S3 (most recent first)
aws s3 ls s3://symplichain-photos/ --recursive | sort | tail -20

# Filter by today's date prefix
aws s3 ls s3://symplichain-photos/2026/04/07/

# Search Django logs for S3-specific errors
grep -i "s3\|botocore\|NoSuchBucket\|AccessDenied" \
  /var/log/gunicorn/error.log | tail -50
```

**Possible findings:**
- `AccessDenied` → EC2 IAM Role lost S3 write permission (was a policy changed Monday?).
- `NoSuchBucket` → Bucket name misconfiguration in environment variables.
- Files present in S3 → problem is downstream in Celery.

---

### Step 3 — Is the Celery task being enqueued and processed?

```bash
# Check Celery Flower dashboard (if running)
# Open: http://<EC2_IP>:5555

# CLI inspection
celery -A config inspect active
celery -A config inspect reserved
celery -A config inspect stats

# Check Celery worker logs
tail -n 200 /var/log/celery/worker.log | grep -E "ERROR|FAILED|photo"

# Check Redis queue depth (are tasks piling up?)
redis-cli -h <ELASTICACHE_ENDPOINT> LLEN celery
```

**Possible findings:**
- Tasks queued but not processing → Worker crashed. Run `sudo systemctl status celery`.
- Tasks failing immediately → Python traceback in worker logs — fix the bug.
- Queue is empty → Task was never enqueued; check the Django view's `.apply_async()` call.

---

### Step 4 — Is AWS Bedrock timing out or erroring?

```bash
# CloudWatch — check Bedrock invocation logs
aws logs filter-log-events \
  --log-group-name /aws/bedrock/modelinvocations \
  --start-time $(date -d "monday 00:00" +%s000) \
  --filter-pattern "ERROR"

# Search Celery logs for Bedrock-specific errors
grep -i "bedrock\|ThrottlingException\|ModelError\|timeout" \
  /var/log/celery/worker.log | tail -50
```

**Possible findings:**
- `ThrottlingException` → Bedrock quota exceeded on Monday's traffic spike. Request a quota increase via AWS Service Quotas.
- `ReadTimeoutError` → Bedrock slow to respond; add retry with backoff in the Celery task.
- No Bedrock errors → problem is in the final RDS write.

---

### Step 5 — Is RDS accepting connections and writes?

```bash
# Check RDS CloudWatch metrics:
# RDS > Databases > symplichain-prod
# Key metrics: DatabaseConnections, FreeStorageSpace, CPUUtilization

# CLI — check connection count
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=symplichain-prod \
  --start-time 2026-04-07T00:00:00 \
  --end-time 2026-04-07T23:59:59 \
  --period 300 --statistics Maximum

# Direct DB connection test from EC2
psql $DATABASE_URL -c "SELECT count(*) FROM deliveries_deliveryphoto LIMIT 1;"

# Check Django logs for DB errors
grep -i "OperationalError\|too many connections\|psycopg" \
  /var/log/gunicorn/error.log | tail -30
```

**Possible findings:**
- `too many connections` → Connection pool exhausted. Implement **PgBouncer** as a connection pooler, or reduce `CONN_MAX_AGE` in Django settings.
- `FreeStorageSpace = 0` → RDS disk full; increase storage or purge old data.
- High RDS CPU → Slow/blocking query; run `pg_stat_activity` to identify it.

---

### Summary Decision Tree

```
Request fails
   │
   ├─ No Django log entry     → Check Nginx / Security Groups / Load Balancer
   │
   ├─ Django 4xx/5xx error    → Fix in Django view (permissions, serializer, file size)
   │
   ├─ S3 AccessDenied/error   → Fix IAM Role policy on EC2
   │
   ├─ Celery task fails
   │     ├─ Bedrock Throttling → Add retry + backoff, request quota increase
   │     └─ RDS error         → Fix connection pool (PgBouncer) or disk space
   │
   └─ All logs clean          → Check Django transaction rollback logic
```

---

### Post-Mortem Actions

1. Add a **CloudWatch alarm** for Celery queue depth > 100.
2. Add a **dedicated log line** after each pipeline stage (S3 save → task enqueue → Bedrock call → RDS write) — makes future debugging a log grep, not a treasure hunt.
3. Set up a **dead-letter queue** in Redis/Celery for permanently failed tasks with SNS alerting.

---

*End of Submission — Symplichain Hackathon | April 2026*
