# peakpool monitoring

Shared monitoring for **all** peakpool chains: one VictoriaMetrics + one Grafana,
one dashboard with a **Chain** filter. VictoriaMetrics scrapes every chain's pool
over a shared Docker network; Grafana shows it on a no-login dashboard.

## Setup

```bash
# 1. the shared network (once)
docker network create peakpool

# 2. a chain stack — it joins the peakpool network automatically
cd ../parano1d && cp .env.example .env && docker compose up -d

# 3. this monitoring stack
cd ../monitoring && docker compose up -d
```

Open the dashboard (no login):

```
http://127.0.0.1:3000/d/peakpool/peakpool?kiosk
```

Grafana binds to loopback, so from another machine reach it over an SSH tunnel
(same as the pool's telemetry): `ssh -L 3000:127.0.0.1:3000 <host>`.

## What the dashboard shows

Per selected chain (or **All**): pool **and estimated network** hashrate, miners,
height, wallet balance, blocks, effort/luck, share rate, template age, and
per-worker hashrate. Network hashrate is estimated as `net_difficulty ×
rate(height)` from the pool's own metrics.

## Notes

- **Retention** is 7 days (`-retentionPeriod=7d` in `docker-compose.yaml`).
  peakpool exposes only a few dozen series per chain, so this is a few MB.
- Data lives in the `vmdata` / `grafana-data` volumes and survives restarts.
- The dashboard is provisioned from [`grafana/dashboards/peakpool.json`](grafana/dashboards/peakpool.json);
  edits in the UI do not persist (that is why anonymous view is safe here).
