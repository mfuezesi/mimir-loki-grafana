# Monitoring Stack: Mimir · Loki · Grafana

Docker-Compose-Stack für zentrales Monitoring mit Grafana Mimir (Metriken), Grafana Loki (Logs) und Grafana (Visualisierung). Läuft hinter einem bestehenden Traefik Reverse Proxy. Multi-Tenancy wird über Traefik BasicAuth geregelt — Clients authentifizieren sich, Traefik setzt den Tenant-Header automatisch.

## Architektur

```
Internet
    │
 Traefik  (externes Docker-Netzwerk: proxy)
    │
    ├── grafana.example.com        →  Grafana  :3000
    ├── mimir.example.com/t/team-a →  BasicAuth → X-Scope-OrgID: team-a → Mimir :8080
    ├── mimir.example.com/t/team-b →  BasicAuth → X-Scope-OrgID: team-b → Mimir :8080
    └── loki.example.com           →  Loki     :3100

 Internes Netzwerk: monitoring
    Grafana → http://mimir:8080/prometheus  (direkt, kein Traefik)
    Grafana → http://loki:3100             (direkt, kein Traefik)

 Client-Maschinen (Alloy)
    → HTTPS + BasicAuth → mimir.example.com/t/{tenant}/api/v1/push
    → Traefik authentifiziert, setzt X-Scope-OrgID, leitet an Mimir weiter
```

**Sicherheit:** Traefik *überschreibt* jeden `X-Scope-OrgID`-Header, den ein Client sendet — Tenant-Spoofing ist nicht möglich.

## Verzeichnisstruktur

```
.
├── docker-compose.yml
├── .env.example
├── .env                          # gitignored, aus .env.example kopieren
├── mimir/
│   └── config/
│       └── mimir.yaml            # Mimir-Konfiguration
├── loki/
│   └── config/
│       └── loki.yaml             # Loki-Konfiguration
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       │   └── datasources.yaml  # Mimir- und Loki-Datasources
│       └── dashboards/
│           └── dashboards.yaml   # Dashboard-Provider
└── alloy-client/
    └── config.alloy              # Alloy-Config für Client-Maschinen
```

## Voraussetzungen

- Docker + Docker Compose v2
- Traefik läuft bereits und verwaltet ein Docker-Netzwerk namens `proxy`
- `apache2-utils` für `htpasswd` (Credential-Generierung)

## Inbetriebnahme

### 1. Konfiguration

```bash
cp .env.example .env
```

`.env` befüllen:

| Variable | Beschreibung |
|----------|-------------|
| `GRAFANA_DOMAIN` | Domain für Grafana, z.B. `grafana.example.com` |
| `MIMIR_DOMAIN` | Domain für Mimir, z.B. `mimir.example.com` |
| `LOKI_DOMAIN` | Domain für Loki, z.B. `loki.example.com` |
| `TRAEFIK_ENTRYPOINT` | Traefik-Entrypoint: `websecure` (HTTPS) oder `web` (HTTP) |
| `TRAEFIK_CERTRESOLVER` | TLS-Resolver aus der Traefik-Konfiguration, z.B. `letsencrypt` |
| `MIMIR_GRAFANA_TENANT` | Tenant-ID für Grafanas internen Mimir-Zugriff |
| `MIMIR_FEDERATION_TENANTS` | Tenant-IDs für Cross-Tenant-Datasource, `\|`-getrennt |
| `GF_SECURITY_ADMIN_USER` | Grafana Admin-Benutzername |
| `GF_SECURITY_ADMIN_PASSWORD` | Grafana Admin-Passwort |
| `GF_SECURITY_SECRET_KEY` | Grafana Secret Key (`openssl rand -hex 32`) |
| `MIMIR_BASICAUTH_TEAM_ALPHA` | htpasswd-Hash für Tenant `team-alpha` |
| `MIMIR_BASICAUTH_TEAM_BETA` | htpasswd-Hash für Tenant `team-beta` |

### 2. BasicAuth-Hashes generieren

Für jeden Tenant wird ein htpasswd-Hash benötigt. Das `$$`-Escaping ist für Docker Compose erforderlich:

```bash
echo $(htpasswd -nb BENUTZERNAME PASSWORT) | sed -e 's/\$/\$\$/g'
```

Beispiel-Ausgabe:
```
user:$$apr1$$xyz$$abc123...
```

Diesen Wert in `.env` eintragen:
```dotenv
MIMIR_BASICAUTH_TEAM_ALPHA=user:$$apr1$$xyz$$abc123...
```

### 3. Stack starten

```bash
docker compose up -d
```

### 4. Status prüfen

```bash
docker compose ps

# Mimir
docker compose exec mimir wget -qO- http://localhost:8080/ready

# Loki
docker compose exec loki wget -qO- http://localhost:3100/ready
```

Grafana ist erreichbar unter `https://GRAFANA_DOMAIN`.

## Tenants verwalten

### Neuen Tenant hinzufügen

**1. Hash generieren und in `.env` eintragen:**
```bash
echo $(htpasswd -nb myuser mypassword) | sed -e 's/\$/\$\$/g'
```
```dotenv
MIMIR_BASICAUTH_TEAM_GAMMA=myuser:$$apr1$$...
```

**2. Traefik-Labels in `docker-compose.yml` ergänzen** (Label-Block von `team-alpha` kopieren, Namen anpassen):

```yaml
# ── Tenant: team-gamma ──
- "traefik.http.routers.mimir-team-gamma.rule=Host(`${MIMIR_DOMAIN}`) && PathPrefix(`/t/team-gamma`)"
- "traefik.http.routers.mimir-team-gamma.entrypoints=${TRAEFIK_ENTRYPOINT}"
- "traefik.http.routers.mimir-team-gamma.tls.certresolver=${TRAEFIK_CERTRESOLVER}"
- "traefik.http.routers.mimir-team-gamma.service=mimir"
- "traefik.http.routers.mimir-team-gamma.middlewares=mimir-auth-team-gamma,mimir-strip-team-gamma,mimir-tenant-team-gamma"
- "traefik.http.middlewares.mimir-auth-team-gamma.basicauth.users=${MIMIR_BASICAUTH_TEAM_GAMMA}"
- "traefik.http.middlewares.mimir-strip-team-gamma.stripprefix.prefixes=/t/team-gamma"
- "traefik.http.middlewares.mimir-tenant-team-gamma.headers.customrequestheaders.X-Scope-OrgID=team-gamma"
```

**3. Grafana-Datasource in `grafana/provisioning/datasources/datasources.yaml` hinzufügen:**

```yaml
- name: "Mimir - team-gamma"
  type: prometheus
  uid: mimir-team-gamma
  access: proxy
  url: http://mimir:8080/prometheus
  editable: false
  jsonData:
    httpMethod: POST
    prometheusType: Mimir
    httpHeaderName1: "X-Scope-OrgID"
  secureJsonData:
    httpHeaderValue1: "team-gamma"
```

**4. Cross-Tenant-Datasource aktualisieren** (`.env`):
```dotenv
MIMIR_FEDERATION_TENANTS=team-alpha|team-beta|team-gamma
```

**5. Anwenden:**
```bash
docker compose up -d mimir
docker compose restart grafana
```

Mimir erstellt den Tenant-Storage automatisch beim ersten Eingang von Metriken — keine manuelle Registrierung nötig.

### Per-Tenant-Limits konfigurieren

Mimir unterstützt Limits pro Tenant über eine Runtime-Konfiguration, die ohne Neustart alle 10 Sekunden neu geladen wird.

**1. `mimir/config/runtime.yaml` anlegen:**

```yaml
overrides:
  team-alpha:
    ingestion_rate: 50000
    max_global_series_per_user: 1000000
    compactor_blocks_retention_period: 365d
  team-beta:
    compactor_blocks_retention_period: 7d
```

**2. In `mimir/config/mimir.yaml` aktivieren** (auskommentierten Block am Ende einkommentieren):

```yaml
runtime_config:
  file: /etc/mimir/runtime.yaml
  period: 10s
```

**3. Mimir neu starten:**

```bash
docker compose restart mimir
```

Danach greift Mimir sofort auf `runtime.yaml` und lädt Änderungen darin ohne Neustart.

## Clients konfigurieren (Alloy)

Die Datei `alloy-client/config.alloy` ist eine fertige Konfiguration für Client-Maschinen.

### Docker

```bash
docker run --rm --net=host \
  -v ./alloy-client/config.alloy:/etc/alloy/config.alloy \
  -e MIMIR_URL=https://mimir.example.com/t/team-alpha/api/v1/push \
  -e MIMIR_USER=myuser \
  -e MIMIR_PASSWORD=mypassword \
  grafana/alloy:latest run /etc/alloy/config.alloy
```

### Nativ (systemd)

```bash
# /etc/alloy/config.alloy kopieren
# /etc/default/alloy oder systemd unit Environment= setzen:
MIMIR_URL=https://mimir.example.com/t/team-alpha/api/v1/push
MIMIR_USER=myuser
MIMIR_PASSWORD=mypassword
```

Der Client setzt **keinen** `X-Scope-OrgID`-Header — das übernimmt Traefik nach der Authentifizierung. Der Tenant ergibt sich aus dem URL-Pfad `/t/{tenant-id}/`.

### Optionale Erweiterungen in `config.alloy`

Die Datei enthält auskommentierte Blöcke für:
- **Alloy-Selbst-Monitoring** — Alloys eigene Metriken an Mimir senden
- **Loki-Log-Shipping** — systemd-Journal-Logs an Loki weiterleiten

## Cross-Tenant-Queries

In Grafana steht die Datasource **"Mimir - All Tenants"** zur Verfügung. Sie fragt alle in `MIMIR_FEDERATION_TENANTS` aufgelisteten Tenants gleichzeitig ab.

- Metriken werden zusammengeführt
- Jede Serie erhält ein zusätzliches Label `__tenant_id__`
- Nützlich für tenantübergreifende Dashboards (z.B. Kapazitätsplanung, Gesamtübersicht)

## Betrieb

### Logs ansehen

```bash
docker compose logs -f mimir
docker compose logs -f loki
docker compose logs -f grafana
```

### Images aktualisieren

```bash
docker compose pull
docker compose up -d
```

### Backup

```bash
docker compose stop
docker run --rm \
  -v mimir-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/mimir-data-$(date +%Y%m%d).tar.gz /data
docker run --rm \
  -v loki-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/loki-data-$(date +%Y%m%d).tar.gz /data
docker compose start
```

### Mimir-Ring-Status

```bash
curl http://localhost:8080/memberlist  # vom Host aus (Port nicht exponiert, daher exec)
docker compose exec mimir wget -qO- http://localhost:8080/memberlist
```

## Komponenten

| Komponente | Version | Beschreibung |
|------------|---------|-------------|
| [Grafana Mimir](https://grafana.com/docs/mimir/latest/) | latest | Prometheus-kompatibler Metrik-Backend, Multi-Tenant |
| [Grafana Loki](https://grafana.com/docs/loki/latest/) | latest | Log-Aggregation, Schema v13 |
| [Grafana](https://grafana.com/docs/grafana/latest/) | latest | Visualisierung, Dashboards |
| [Grafana Alloy](https://grafana.com/docs/alloy/latest/) | latest | Collector für Client-Maschinen (River-Format) |
