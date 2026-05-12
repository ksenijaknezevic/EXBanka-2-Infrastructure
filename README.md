# EXBanka-2-Infrastructure

## Trgovina na berzi (bank-service) — API ključevi i worker

Za osvežavanje cena hartija `bank-service` očekuje u `.env` sledeće promenljive (vidi `.env.example`):

- **EODHD_API_KEY** — primarni izvor (akcije, forex, futures, istorija).
- **FINNHUB_API_KEY** — sekundarni izvor za akcije / forex.
- **ALPHAVANTAGE_API_KEY** — Company Overview za akcije i forex kurs (sekundarno).

Ponašanje kada API ne odgovori: uz **`LISTING_REQUIRE_LIVE_QUOTES=true`** (podrazumevano u `docker-compose`) worker **ne upisuje** sintetičke ili nasumične cene; u bazi ostaje poslednji poznati quote dok se sledeći uspešan odgovor ne dobije. Za lokalno testiranje bez ključeva možete postaviti `LISTING_REQUIRE_LIVE_QUOTES=false` (samo development).

**Napomena:** opcije koriste Yahoo Finance lanac u worker-u; ako Yahoo ne vrati podatke, uz `LISTING_REQUIRE_LIVE_QUOTES=true` takođe nema ažuriranja mock vrednostima.

**Provizija (bank-service):** Market/Stop ≈ min(14% notionala, fiksni iznos); Limit/Stop-Limit ≈ min(24%, fiksni iznos) — implementacija u `EXBanka-2-Backend/services/bank-service/internal/trading/calculations.go`.

**Porez / država kao firma:** opciono postaviti **`STATE_REVENUE_ACCOUNT_ID`** na `core_banking.racun.id` tekućeg **RSD** računa „države”; pri `POST /bank/tax/calculate` isti iznos u RSD knjiži se kao prijem na taj račun (pored skidanja sa klijenta).

## Monitoring / Logging / Alerting (MLA)

Stack: **Prometheus** (scrape svakih 15s) → **Grafana** (dashboard) + **AlertManager** (Discord webhook).

Backend servisi izlažu `/metrics` (Prometheus format):

| Servis                | Endpoint (intra-network)            |
|-----------------------|-------------------------------------|
| user-service          | `http://user-service:8080/metrics`  |
| bank-service          | `http://bank-service:8080/metrics`  |
| notification-service  | `http://notification-service:8083/metrics` |

### Pokretanje

```bash
cp .env.example .env
cp alertmanager/discord_webhook_url.example alertmanager/discord_webhook_url
# Otvori alertmanager/discord_webhook_url i upiši pravi Discord webhook URL
docker compose up -d
```

UI portovi (default):
- Grafana:      http://localhost:3000  (admin/admin — promeni u `.env`)
- Prometheus:   http://localhost:9090  (provera: `/targets` — sve mora biti UP)
- AlertManager: http://localhost:9093

### Discord webhook

U Discord serveru: **Channel Settings → Integrations → Webhooks → New Webhook**, kopiraj URL i upiši ga u `alertmanager/discord_webhook_url` (taj fajl je u `.gitignore`).

### Alert pravila

Definisana u `prometheus/rules/alerts.yml`:
- **ServiceDown** — `up == 0` ≥ 1min (critical)
- **HighGRPCErrorRate** — > 5% non-OK gRPC odgovora ≥ 5min
- **HighGRPCLatencyP99** — p99 > 1s ≥ 5min
- **HighHTTPErrorRate** — > 5% HTTP 5xx ≥ 5min

### Grafana dashboard

Auto-provisionovan kao **„EXBanka — RED Overview"** (Rate / Errors / Duration + Service UP panel).
