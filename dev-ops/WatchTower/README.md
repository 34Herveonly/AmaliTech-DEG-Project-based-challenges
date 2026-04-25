# Reyla Logistics Observability Stack

This repository contains the completed observability challenge for Reyla Logistics. I've wired up the three existing services (order, tracking, and notification) with a full monitoring stack using Prometheus, Grafana, and Alertmanager. 

## Architecture Diagram

Here is a high-level view of how the observability stack fits together:

```text
[ Node.js Services ]
  ├─ order-service (:3001)
  ├─ tracking-service (:3002)
  └─ notification-service (:3003)
         │
         ▼ (scrapes /metrics every 15s)
[ Prometheus ] (:9090)
  (Evaluates alert rules)
         │
         ├──────────────────────────────┐
         ▼                              ▼
[ Grafana ] (:3000)             [ Alertmanager ] (:9093)
(Auto-provisioned dashboards)   (Routes critical/warning alerts)
```

## Setup Instructions

1. **Environment Variables**
   Copy the example config to create your local `.env` file:
   ```bash
   cp app/.env.example app/.env
   ```
   *(You can leave the defaults as they are for local testing).*

2. **Start the Stack**
   From the `app/` directory, spin everything up using Docker Compose:
   ```bash
   docker compose up --build -d
   ```

3. **Verify it's working**
   - **Prometheus Targets:** Go to `http://localhost:9090/targets`. You should see all three services listed as "UP".
   - **Grafana:** Go to `http://localhost:3000` (Login: `admin` / `reyla2024!`). You'll see the pre-built dashboard load automatically without any manual imports.

## Dashboard Walkthrough

The Grafana dashboard (found at `/dashboards`) is split into a few key sections to give the team instant visibility into what's happening:

- **Service Health:** Three main stat panels that simply say "UP" (green) or "DOWN" (red). It queries the `up` metric from Prometheus so we know immediately if a service is unreachable.
- **Traffic (HTTP Request Rate):** A line graph showing requests per second for each service (`rate(http_requests_total[1m])`). This helps identify traffic spikes.
- **5xx Error Rate:** Tracks server errors over a 5-minute window. If any service starts throwing 500s, it'll show up here (which ties into our alerting).
- **Total Requests:** Cumulative request counters for capacity planning.
- **P95 Response Latency:** Ready to track 95th percentile response times, though the current services only use basic counters so this panel will show "No data" until we add histogram instrumentation to the Express apps.

## Alert Testing Guide

I added three alerts in `app/prometheus/alerts.yml`. Here is how you can test them:

1. **ServiceDown (Critical)**
   - *Test:* Run `docker compose stop order-service`.
   - *Result:* Wait about 1 minute. Check `http://localhost:9090/alerts` and you will see the `ServiceDown` alert move from pending to FIRING. It will then be routed to Alertmanager. Restart the service (`docker compose start order-service`) to resolve it.

2. **HighErrorRate (Warning)**
   - *Test:* Since the apps don't throw 500s naturally, you would need to generate bad requests or modify the code to force a crash. But functionally, if 5xx errors exceed 5% of traffic over 5 minutes, this will fire.

3. **ServiceNotScraping (Warning)**
   - *Test:* This triggers if Prometheus gets no data at all for 2 minutes. Stopping a service (like in test 1) will also eventually trigger this alert after 2-4 minutes as the time-series goes completely stale.

## Log Commands

The `docker-compose.yml` configures Docker to use the `json-file` driver so our logs are structured and capped at 10MB to prevent disk issues. 

Here are the commands to view them:

**1. View live logs from all services at once:**
```bash
docker compose logs -f order-service tracking-service notification-service
```

**2. Filter logs to show only errors from a specific service:**
Since the apps output JSON logs, we can just grep for the error level. 
```bash
docker compose logs order-service 2>&1 | grep '"level":"error"'
```
*(Example output: `order-service | {"level":"error","service":"order-service","msg":"Unhandled exception"}`)*

---
*Note: I also added Alertmanager as a bonus feature to handle webhook routing for the alerts.*
