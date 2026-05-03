# 🛠️ TP Module 5 — Microservice Entièrement Instrumenté

> **Durée estimée :** 4 à 6 heures
> **Compétence validée :** Observabilité complète + résilience en production

---

## Objectif

Prendre l'API REST du Module 2, l'équiper d'une observabilité complète (logs JSON, métriques Prometheus, tracing OTLP), ajouter un circuit breaker sur les appels externes, injecter des erreurs et vérifier que les dashboards reflètent le comportement.

---

## Architecture finale

```
Requête
  │
  ▼
Axum Router
  │── middleware_tracing (span par requête)
  │── middleware_métriques (durée, statut, count)
  │── middleware_chaos (en dev seulement)
  │
  ├── /kv/*         → KV Store (Module 3)
  ├── /api/v1/*     → API REST (Module 2)
  │     └── appels externes → CircuitBreaker → Retry
  ├── /metrics      → Prometheus scrape
  ├── /santé        → Health check
  └── /cluster      → État du cluster

         │
         ▼
    ┌─────────────────┐   ┌──────────────┐   ┌──────────┐
    │   Prometheus    │   │    Jaeger    │   │  Loki    │
    │   (métriques)   │   │   (traces)   │   │  (logs)  │
    └────────┬────────┘   └──────┬───────┘   └────┬─────┘
             └──────────────────┼────────────────┘
                                ▼
                           Grafana (dashboards)
```

---

## Étape 1 — Stack d'observabilité (docker-compose.yml)

```yaml
# docker-compose.yml
services:
  mon-service:
    build: .
    ports: ["8080:8080"]
    environment:
      RUST_LOG: "mon_service=debug,info"
      RUST_LOG_JSON: "1"
      DATABASE_URL: "postgresql://admin:mdp@postgres/app"
      OTLP_ENDPOINT: "http://jaeger:4317"
    depends_on: [postgres, prometheus, jaeger]

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: mdp

  prometheus:
    image: prom/prometheus
    volumes:
      - ./infra/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./infra/alertes.yml:/etc/prometheus/alertes.yml
    ports: ["9090:9090"]

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "4317:4317"
    environment:
      COLLECTOR_OTLP_ENABLED: "true"

  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - ./infra/promtail.yml:/etc/promtail/config.yml

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./infra/grafana-datasources.yml:/etc/grafana/provisioning/datasources/ds.yml
      - ./infra/grafana-dashboards.yml:/etc/grafana/provisioning/dashboards/db.yml
      - ./infra/dashboards:/var/lib/grafana/dashboards
```

---

## Étape 2 — Application complète (src/main.rs)

```rust
// src/main.rs
mod api;
mod circuit_breaker;
mod métriques;
mod télémétrie;

use axum::{middleware, Router};
use prometheus::Registry;
use std::sync::Arc;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let env = std::env::var("APP_ENV").unwrap_or("development".into());
    let otlp = std::env::var("OTLP_ENDPOINT").ok();

    // Initialiser la télémétrie
    if let Some(ref endpoint) = otlp {
        télémétrie::initialiser_avec_otlp("mon-service", endpoint)?;
    } else {
        télémétrie::initialiser_tracing("mon-service", &env);
    }

    // Métriques Prometheus
    let registre = Arc::new(Registry::new());
    let métriques = Arc::new(métriques::Métriques::nouvelles(&registre));

    // Circuit breaker pour le service de notifications externe
    let cb_notifs = Arc::new(circuit_breaker::CircuitBreaker::nouveau(
        "service-notifications",
        circuit_breaker::ConfigCB {
            seuil_échecs: 5,
            délai_récupération: std::time::Duration::from_secs(30),
            seuil_succès_semi: 2,
        },
    ));

    let état_app = Arc::new(api::ÉtatApp {
        métriques: métriques.clone(),
        cb_notifs,
        // ... autres dépendances
    });

    // Config chaos (désactivé en production)
    let config_chaos = Arc::new(circuit_breaker::ConfigChaos {
        activé: env == "development",
        taux_latence: 0.05,
        latence_min_ms: 50,
        latence_max_ms: 500,
        taux_erreur: 0.02,
        taux_timeout: 0.01,
    });

    let app = Router::new()
        // Routes applicatives
        .nest("/api/v1", api::router(état_app.clone()))
        // Observabilité
        .route("/metrics", axum::routing::get(métriques::handler_métriques))
        .route("/santé", axum::routing::get(handler_santé))
        // Middlewares dans l'ordre (dernier = exécuté en premier)
        .layer(middleware::from_fn_with_state(
            config_chaos,
            circuit_breaker::middleware_chaos,
        ))
        .layer(middleware::from_fn_with_state(
            métriques.clone(),
            métriques::middleware_métriques,
        ))
        .layer(tower_http::trace::TraceLayer::new_for_http())
        .with_state((état_app, Arc::clone(&registre)));

    let addr = "0.0.0.0:8080";
    tracing::info!(addr = addr, env = %env, "Service démarré");

    axum::serve(
        tokio::net::TcpListener::bind(addr).await?,
        app,
    ).await?;

    // Flush des traces OTLP à l'arrêt
    opentelemetry::global::shutdown_tracer_provider();
    Ok(())
}

async fn handler_santé() -> axum::response::Json<serde_json::Value> {
    axum::response::Json(serde_json::json!({
        "statut": "ok",
        "version": env!("CARGO_PKG_VERSION"),
        "horodatage": chrono::Utc::now().to_rfc3339(),
    }))
}
```

---

## Étape 3 — Dashboard Grafana (infra/dashboards/service.json)

Créer un dashboard avec ces panneaux :

**Row 1 — Vue d'ensemble**
- Taux de requêtes (req/s) : `rate(mon_service_http_requêtes_total[1m])`
- Taux d'erreurs : `rate(mon_service_erreurs_total[1m]) / rate(mon_service_http_requêtes_total[1m])`
- P50/P95/P99 latence

**Row 2 — Ressources**
- Connexions DB actives
- Requêtes HTTP actives
- Taille du cache

**Row 3 — Résilience**
- État du circuit breaker (0 = fermé, 1 = ouvert)
- Requêtes rejetées par bulkhead
- Retries par seconde

---

## Étape 4 — Tests de résilience

```bash
# Démarrer la stack
docker compose up -d
sleep 5

# Test 1 : charge normale
ab -n 10000 -c 100 http://localhost:8080/santé
# Vérifier dans Grafana : latence P99 < 50ms

# Test 2 : saturation
ab -n 100000 -c 500 http://localhost:8080/api/v1/utilisateurs
# Observer : requêtes rejetées, latence qui monte, puis se stabilise

# Test 3 : activer le chaos
curl -X POST http://localhost:8080/admin/chaos \
  -d '{"activé": true, "taux_erreur": 0.10}'
# Observer : circuit breaker qui s'ouvre dans Grafana

# Test 4 : coupure base de données
docker compose stop postgres
# Observer : erreurs DB → circuit breaker ouvert → réponses rapides (pas de timeout)
# Puis :
docker compose start postgres
# Observer : circuit breaker passe en semi-ouvert puis se ferme

# Vérifier les traces dans Jaeger : http://localhost:16686
# Chercher les traces avec erreurs et voir le chemin d'exécution complet
```

---

## Étape 5 — Alertes opérationnelles

```yaml
# infra/alertes.yml
groups:
  - name: slo_service
    rules:
      # SLO : 99.5% de disponibilité
      - alert: DisponibilitéInsuffisante
        expr: |
          1 - (
            rate(mon_service_http_requêtes_total{statut=~"5.."}[1h]) /
            rate(mon_service_http_requêtes_total[1h])
          ) < 0.995
        for: 5m
        annotations:
          summary: "Disponibilité < 99.5% sur la dernière heure"

      # Circuit breaker ouvert depuis > 5 minutes
      - alert: CircuitBreakerOuvert
        expr: mon_service_circuit_breaker_état == 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Circuit breaker {{ $labels.service }} ouvert depuis 5min"
```

---

## 🏆 Critères de réussite

| Critère | Validé ? |
|---|---|
| Logs JSON avec correlation ID par requête | ✅ |
| Métriques Prometheus exposées sur /metrics | ✅ |
| Traces visibles dans Jaeger (span parent→enfant) | ✅ |
| Circuit breaker observable dans Grafana | ✅ |
| Les alertes se déclenchent lors des tests de charge | ✅ |
| Le service reste fonctionnel avec 10% d'erreurs injectées | ✅ |
| Graceful shutdown : les requêtes en cours se terminent | ✅ |

---

**→ Suivant : [Module 6 — WebAssembly, Blockchain et Projet Final](../../module-6/README.md)**
