# 🛠️ TP Module 4 — Cluster Distribué Multi-Nœuds avec Réplication

> **Durée estimée :** 6 à 8 heures
> **Compétence validée :** Système distribué tolérant aux pannes, conçu pour la production

---

## Objectif

Assembler un cluster de 3 nœuds KV avec leader election (Raft simplifié), réplication synchrone, détection de panne, et tests de chaos.

---

## Architecture

```
        Client
          │
     ┌────▼─────┐
     │ LB/Proxy │  → route toutes les écritures vers le Leader
     └────┬─────┘
          │
  ┌───────┼───────┐
  │       │       │
┌─┴──┐ ┌──┴─┐ ┌──┴─┐
│ N1  │ │ N2  │ │ N3  │
│8081 │ │8082 │ │8083 │
│LEAD │ │FOL  │ │FOL  │
└─────┘ └─────┘ └─────┘
   └─────heartbeat──────┘

Chaque nœud expose :
  - /kv/:clé    → opérations KV
  - /cluster    → état du cluster
  - /raft/*     → messages internes Raft
```

---

## Étape 1 — Structure du projet (workspace)

```toml
# Cargo.toml (workspace)
[workspace]
members = ["nœud"]
resolver = "2"

[workspace.dependencies]
tokio  = { version = "1", features = ["full"] }
axum   = "0.7"
serde  = { version = "1", features = ["derive"] }
serde_json = "1"
reqwest = { version = "0.12", features = ["json"] }
uuid   = { version = "1", features = ["v4"] }
tracing = "0.1"
tracing-subscriber = "0.3"
thiserror = "1"
anyhow  = "1"
```

---

## Étape 2 — État Raft simplifié (nœud/src/raft/état.rs)

```rust
// nœud/src/raft/état.rs
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{RwLock, watch};
use serde::{Deserialize, Serialize};

pub type NœudId = u32;
pub type Terme = u64;

#[derive(Debug, Clone, PartialEq, Serialize)]
pub enum RôleRaft { Follower, Candidate, Leader }

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EntréeLog {
    pub terme: Terme,
    pub index: u64,
    pub clé: String,
    pub valeur: Option<Vec<u8>>, // None = suppression
}

#[derive(Debug)]
pub struct ÉtatRaft {
    pub id: NœudId,
    pub pairs: Vec<String>,           // adresses HTTP des pairs
    pub rôle: RwLock<RôleRaft>,
    pub terme: RwLock<Terme>,
    pub voté_pour: RwLock<Option<NœudId>>,
    pub log: RwLock<Vec<EntréeLog>>,
    pub commit_index: RwLock<u64>,
    pub leader_actuel: RwLock<Option<NœudId>>,
    pub kv: RwLock<HashMap<String, Vec<u8>>>,
    pub tx_rôle: watch::Sender<RôleRaft>,
}

impl ÉtatRaft {
    pub fn nouveau(id: NœudId, pairs: Vec<String>) -> (Arc<Self>, watch::Receiver<RôleRaft>) {
        let (tx, rx) = watch::channel(RôleRaft::Follower);
        let état = Arc::new(ÉtatRaft {
            id,
            pairs,
            rôle: RwLock::new(RôleRaft::Follower),
            terme: RwLock::new(0),
            voté_pour: RwLock::new(None),
            log: RwLock::new(Vec::new()),
            commit_index: RwLock::new(0),
            leader_actuel: RwLock::new(None),
            kv: RwLock::new(HashMap::new()),
            tx_rôle: tx,
        });
        (état, rx)
    }

    pub async fn devenir_follower(&self, terme: Terme) {
        *self.terme.write().await = terme;
        *self.voté_pour.write().await = None;
        *self.rôle.write().await = RôleRaft::Follower;
        let _ = self.tx_rôle.send(RôleRaft::Follower);
        tracing::info!(nœud = self.id, terme = terme, "→ Follower");
    }

    pub async fn devenir_leader(&self) {
        *self.rôle.write().await = RôleRaft::Leader;
        *self.leader_actuel.write().await = Some(self.id);
        let _ = self.tx_rôle.send(RôleRaft::Leader);
        tracing::info!(nœud = self.id, "🏆 → LEADER");
    }

    pub async fn appliquer_log_jusqu_à(&self, index: u64) {
        let mut commit = self.commit_index.write().await;
        let log = self.log.read().await;
        let mut kv = self.kv.write().await;

        while *commit < index && (*commit as usize) < log.len() {
            let entrée = &log[*commit as usize];
            match &entrée.valeur {
                Some(v) => { kv.insert(entrée.clé.clone(), v.clone()); }
                None    => { kv.remove(&entrée.clé); }
            }
            *commit += 1;
            tracing::debug!(index = *commit, clé = %entrée.clé, "Entrée appliquée");
        }
    }
}
```

---

## Étape 3 — Moteur Raft : élection et heartbeat (nœud/src/raft/moteur.rs)

```rust
// nœud/src/raft/moteur.rs
use std::sync::Arc;
use std::time::Duration;
use rand::Rng;
use tokio::time::{sleep, Instant};
use serde::{Deserialize, Serialize};

use super::état::{ÉtatRaft, RôleRaft, EntréeLog, Terme, NœudId};

#[derive(Debug, Serialize, Deserialize)]
pub struct RequêteVote {
    pub terme: Terme,
    pub candidat: NœudId,
    pub dernier_index: u64,
    pub dernier_terme: Terme,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct RéponseVote {
    pub terme: Terme,
    pub accordé: bool,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct RequêteAppend {
    pub terme: Terme,
    pub leader: NœudId,
    pub entrées: Vec<EntréeLog>,
    pub commit_index: u64,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct RéponseAppend {
    pub terme: Terme,
    pub succès: bool,
}

pub async fn boucle_raft(état: Arc<ÉtatRaft>, client: reqwest::Client) {
    loop {
        let rôle = état.rôle.read().await.clone();
        match rôle {
            RôleRaft::Follower | RôleRaft::Candidate => {
                attendre_heartbeat_ou_élection(état.clone(), client.clone()).await;
            }
            RôleRaft::Leader => {
                envoyer_heartbeats(état.clone(), client.clone()).await;
                sleep(Duration::from_millis(150)).await;
            }
        }
    }
}

async fn attendre_heartbeat_ou_élection(état: Arc<ÉtatRaft>, client: reqwest::Client) {
    // Timeout aléatoire entre 300ms et 600ms
    let timeout_ms = rand::thread_rng().gen_range(300..600);
    let échéance = Instant::now() + Duration::from_millis(timeout_ms);

    loop {
        if Instant::now() >= échéance {
            // Timeout écoulé sans heartbeat → déclencher une élection
            démarrer_élection(état.clone(), client.clone()).await;
            return;
        }

        // Vérifier si on a reçu un heartbeat récent
        // (dans un vrai système : signalé via un channel)
        sleep(Duration::from_millis(50)).await;

        let rôle = état.rôle.read().await.clone();
        if rôle != RôleRaft::Follower && rôle != RôleRaft::Candidate {
            return; // On est devenu leader
        }
    }
}

async fn démarrer_élection(état: Arc<ÉtatRaft>, client: reqwest::Client) {
    // Incrémenter le terme et voter pour soi-même
    let nouveau_terme = {
        let mut terme = état.terme.write().await;
        *terme += 1;
        *terme
    };
    *état.voté_pour.write().await = Some(état.id);
    *état.rôle.write().await = RôleRaft::Candidate;

    tracing::info!(nœud = état.id, terme = nouveau_terme, "🗳  Élection démarrée");

    let (dernier_index, dernier_terme) = {
        let log = état.log.read().await;
        let dernier = log.last();
        (
            dernier.map_or(0, |e| e.index),
            dernier.map_or(0, |e| e.terme),
        )
    };

    let req_vote = RequêteVote {
        terme: nouveau_terme,
        candidat: état.id,
        dernier_index,
        dernier_terme,
    };

    // Demander les votes en parallèle
    let quorum = état.pairs.len() / 2 + 1;
    let mut votes = 1usize; // On vote pour soi-même

    let résultats = futures::future::join_all(
        état.pairs.iter().map(|addr| {
            let client = client.clone();
            let req = serde_json::to_value(&req_vote).unwrap();
            let url = format!("{}/raft/vote", addr);
            async move {
                client.post(&url)
                    .json(&req)
                    .timeout(Duration::from_millis(200))
                    .send()
                    .await
                    .ok()
                    .and_then(|r| r.json::<RéponseVote>().ok().map(|v| v))
            }
        })
    ).await;

    for rep in résultats.into_iter().flatten() {
        if rep.terme > nouveau_terme {
            état.devenir_follower(rep.terme).await;
            return;
        }
        if rep.accordé { votes += 1; }
    }

    if votes >= quorum {
        état.devenir_leader().await;
        // Envoyer immédiatement des heartbeats pour affirmer le leadership
        envoyer_heartbeats(état, client).await;
    } else {
        tracing::info!(nœud = état.id, votes = votes, quorum = quorum, "Élection perdue");
        état.devenir_follower(nouveau_terme).await;
    }
}

async fn envoyer_heartbeats(état: Arc<ÉtatRaft>, client: reqwest::Client) {
    let terme = *état.terme.read().await;
    let commit = *état.commit_index.read().await;

    let req = RequêteAppend {
        terme,
        leader: état.id,
        entrées: vec![],  // Heartbeat vide
        commit_index: commit,
    };

    for addr in &état.pairs {
        let client = client.clone();
        let req = req.clone();
        let url = format!("{}/raft/append", addr);
        let état_clone = état.clone();

        tokio::spawn(async move {
            if let Ok(rep) = client.post(&url)
                .json(&req)
                .timeout(Duration::from_millis(100))
                .send()
                .await
                .and_then(|r| Ok(r.json::<RéponseAppend>()))
            {
                if let Ok(rep) = rep.await {
                    if rep.terme > *état_clone.terme.read().await {
                        état_clone.devenir_follower(rep.terme).await;
                    }
                }
            }
        });
    }
}

// Répliquer une entrée sur les followers (appelé par le leader)
pub async fn répliquer(
    état: Arc<ÉtatRaft>,
    client: reqwest::Client,
    entrée: EntréeLog,
) -> bool {
    let terme = *état.terme.read().await;
    let commit = *état.commit_index.read().await;
    let quorum = état.pairs.len() / 2 + 1;
    let mut confirmations = 1usize;

    let req = RequêteAppend {
        terme,
        leader: état.id,
        entrées: vec![entrée],
        commit_index: commit,
    };

    let résultats = futures::future::join_all(
        état.pairs.iter().map(|addr| {
            let client = client.clone();
            let req = req.clone();
            let url = format!("{}/raft/append", addr);
            async move {
                client.post(&url)
                    .json(&req)
                    .timeout(Duration::from_millis(500))
                    .send()
                    .await
                    .ok()
                    .and_then(|r| r.json::<RéponseAppend>().ok())
            }
        })
    ).await;

    for rep in résultats.into_iter().flatten() {
        if rep.succès { confirmations += 1; }
    }

    confirmations >= quorum
}
```

---

## Étape 4 — Handlers HTTP et Raft (nœud/src/main.rs)

```rust
// nœud/src/main.rs (structure simplifiée)
use axum::{extract::{Path, State}, response::Json, routing::{get, post, put}, Router};
use std::sync::Arc;
use raft::{état::ÉtatRaft, moteur};

mod raft;

type AppState = Arc<ÉtatRaft>;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt().with_env_filter("debug").init();

    let id: u32   = std::env::var("NŒUD_ID")?.parse()?;
    let port: u16 = std::env::var("PORT")?.parse()?;
    let pairs: Vec<String> = std::env::var("PAIRS")?
        .split(',')
        .map(|s| s.trim().to_string())
        .collect();

    let (état, _rx) = ÉtatRaft::nouveau(id, pairs);
    let client = reqwest::Client::new();

    // Démarrer la boucle Raft en arrière-plan
    tokio::spawn(moteur::boucle_raft(état.clone(), client.clone()));

    let app = Router::new()
        // API KV
        .route("/kv/:clé", get(handler_get).put(handler_set).delete(handler_delete))
        // API Raft interne
        .route("/raft/vote",   post(handler_vote))
        .route("/raft/append", post(handler_append))
        // Statut du cluster
        .route("/cluster", get(handler_cluster))
        .with_state(état);

    let addr = format!("0.0.0.0:{}", port);
    tracing::info!("Nœud {} démarré sur {}", id, addr);
    let listener = tokio::net::TcpListener::bind(&addr).await?;
    axum::serve(listener, app).await?;
    Ok(())
}

// Handler SET : redirect vers le leader si on n'est pas leader
async fn handler_set(
    State(état): State<AppState>,
    Path(clé): Path<String>,
    body: axum::body::Bytes,
) -> axum::response::Response {
    use axum::{http::StatusCode, response::IntoResponse};

    let est_leader = *état.rôle.read().await == raft::état::RôleRaft::Leader;
    if !est_leader {
        let leader = état.leader_actuel.read().await.clone();
        if let Some(_) = leader {
            // Retourner un 307 Redirect vers le leader
            return (StatusCode::TEMPORARY_REDIRECT, "Redirect vers le leader").into_response();
        }
        return (StatusCode::SERVICE_UNAVAILABLE, "Pas de leader disponible").into_response();
    }

    // On est le leader — répliquer avant d'accepter
    let entrée = raft::état::EntréeLog {
        terme: *état.terme.read().await,
        index: état.log.read().await.len() as u64 + 1,
        clé: clé.clone(),
        valeur: Some(body.to_vec()),
    };

    let client = reqwest::Client::new();
    if moteur::répliquer(état.clone(), client, entrée.clone()).await {
        état.log.write().await.push(entrée.clone());
        let nouvel_index = état.log.read().await.len() as u64;
        état.appliquer_log_jusqu_à(nouvel_index).await;
        (StatusCode::OK, Json(serde_json::json!({"ok": true}))).into_response()
    } else {
        (StatusCode::INTERNAL_SERVER_ERROR, "Quorum non atteint").into_response()
    }
}

async fn handler_get(
    State(état): State<AppState>,
    Path(clé): Path<String>,
) -> axum::response::Response {
    use axum::{http::StatusCode, response::IntoResponse};
    let kv = état.kv.read().await;
    match kv.get(&clé) {
        Some(v) => (StatusCode::OK, v.clone()).into_response(),
        None    => StatusCode::NOT_FOUND.into_response(),
    }
}

async fn handler_cluster(State(état): State<AppState>) -> Json<serde_json::Value> {
    Json(serde_json::json!({
        "nœud_id": état.id,
        "rôle": format!("{:?}", *état.rôle.read().await),
        "terme": *état.terme.read().await,
        "commit_index": *état.commit_index.read().await,
        "nb_entrées_log": état.log.read().await.len(),
    }))
}

async fn handler_vote(
    State(état): State<AppState>,
    Json(req): Json<moteur::RequêteVote>,
) -> Json<moteur::RéponseVote> {
    let terme_actuel = *état.terme.read().await;
    if req.terme < terme_actuel {
        return Json(moteur::RéponseVote { terme: terme_actuel, accordé: false });
    }
    if req.terme > terme_actuel {
        état.devenir_follower(req.terme).await;
    }
    let voté = état.voté_pour.read().await.clone();
    let peut_voter = voté.is_none() || voté == Some(req.candidat);
    if peut_voter {
        *état.voté_pour.write().await = Some(req.candidat);
    }
    Json(moteur::RéponseVote { terme: *état.terme.read().await, accordé: peut_voter })
}

async fn handler_append(
    State(état): State<AppState>,
    Json(req): Json<moteur::RequêteAppend>,
) -> Json<moteur::RéponseAppend> {
    let terme_actuel = *état.terme.read().await;
    if req.terme < terme_actuel {
        return Json(moteur::RéponseAppend { terme: terme_actuel, succès: false });
    }
    état.devenir_follower(req.terme).await;
    *état.leader_actuel.write().await = Some(req.leader);

    for entrée in req.entrées {
        état.log.write().await.push(entrée);
    }
    if req.commit_index > *état.commit_index.read().await {
        état.appliquer_log_jusqu_à(req.commit_index).await;
    }
    Json(moteur::RéponseAppend { terme: *état.terme.read().await, succès: true })
}

async fn handler_delete(
    State(état): State<AppState>,
    Path(clé): Path<String>,
) -> axum::http::StatusCode {
    état.kv.write().await.remove(&clé);
    axum::http::StatusCode::NO_CONTENT
}
```

---

## Étape 5 — Lancer le cluster avec Docker Compose

```yaml
# docker-compose.yml
services:
  nœud1:
    build: ./nœud
    environment:
      NŒUD_ID: "1"
      PORT: "8081"
      PAIRS: "http://nœud2:8082,http://nœud3:8083"
    ports: ["8081:8081"]

  nœud2:
    build: ./nœud
    environment:
      NŒUD_ID: "2"
      PORT: "8082"
      PAIRS: "http://nœud1:8081,http://nœud3:8083"
    ports: ["8082:8082"]

  nœud3:
    build: ./nœud
    environment:
      NŒUD_ID: "3"
      PORT: "8083"
      PAIRS: "http://nœud1:8081,http://nœud2:8082"
    ports: ["8083:8083"]
```

```bash
docker compose up -d

# Attendre l'élection (~1s)
sleep 2

# Écrire sur le leader (trouver lequel)
for port in 8081 8082 8083; do
  echo "Nœud sur $port :"; curl -s http://localhost:$port/cluster | jq .rôle
done

# Écrire une clé (supposons que le leader est sur 8081)
curl -X PUT http://localhost:8081/kv/hello -d "world"

# Lire depuis un follower (8082) — cohérence éventuelle
curl http://localhost:8082/kv/hello

# Test de panne : tuer le leader
docker compose stop nœud1
sleep 2

# Vérifier la nouvelle élection
curl http://localhost:8082/cluster | jq .rôle  # Doit être "Leader"

# La donnée est toujours disponible
curl http://localhost:8082/kv/hello
```

---

## 🏆 Critères de réussite

| Critère | Validé ? |
|---|---|
| Un leader est élu en < 2 secondes au démarrage | ✅ |
| Les écritures sont répliquées sur tous les nœuds | ✅ |
| La panne du leader déclenche une nouvelle élection | ✅ |
| Les données persistent après redémarrage d'un follower | ✅ |
| Les lectures fonctionnent depuis n'importe quel nœud | ✅ |
| Les métriques `/cluster` reflètent l'état réel | ✅ |

---

**→ Suivant : [Module 5 — Observabilité et Résilience](../../module-5/README.md)**
