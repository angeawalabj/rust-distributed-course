# 🛠️ TP Module 3 — Moteur Clé-Valeur Persistant avec API Réseau

> **Durée estimée :** 5 à 7 heures
> **Compétence validée :** Concevoir un moteur de stockage distribué robuste et performant

---

## Objectif

Construire un moteur clé-valeur production-ready : persistance via WAL, API HTTP, sharding, indexation inversée, et tests de performance sous charge.

---

## Architecture cible

```
          Clients HTTP
               │
    ┌──────────┴──────────┐
    │   API Axum (:8080)  │
    │   GET/PUT/DELETE    │
    └──────────┬──────────┘
               │
    ┌──────────┴──────────────────────┐
    │        Router de shards         │
    │   hash(clé) % N → shard[i]     │
    └──┬──────┬──────┬──────┬────────┘
       │      │      │      │
    ┌──┴─┐ ┌──┴─┐ ┌──┴─┐ ┌──┴─┐
    │ S0 │ │ S1 │ │ S2 │ │ S3 │   ← 4 shards
    │WAL │ │WAL │ │WAL │ │WAL │
    └────┘ └────┘ └────┘ └────┘
```

---

## Étape 1 — Structure du projet

```
kv-store/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── store/
│   │   ├── mod.rs
│   │   ├── wal.rs          ← Write-Ahead Log
│   │   ├── memtable.rs     ← Mémtable en RAM
│   │   ├── shard.rs        ← Shard individuel
│   │   └── sharded.rs      ← Routeur de shards
│   ├── api/
│   │   ├── mod.rs
│   │   ├── handlers.rs     ← Handlers HTTP
│   │   └── erreurs.rs      ← Erreurs API
│   └── métriques.rs        ← Compteurs Prometheus
├── tests/
│   ├── test_wal.rs
│   ├── test_sharding.rs
│   └── test_charge.rs
└── benches/
    └── performance.rs
```

```toml
# Cargo.toml
[package]
name = "kv-store"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
axum = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
bincode = "1"
thiserror = "1"
tracing = "0.1"
tracing-subscriber = "0.3"
anyhow = "1"

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
reqwest = { version = "0.11", features = ["json"] }
tempfile = "3"

[[bench]]
name = "performance"
harness = false
```

---

## Étape 2 — WAL (src/store/wal.rs)

```rust
// src/store/wal.rs
use std::fs::{File, OpenOptions};
use std::io::{BufWriter, Read, Write};
use std::path::{Path, PathBuf};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, Clone)]
pub enum OpérationWAL {
    Set { clé: String, valeur: Vec<u8>, version: u64 },
    Delete { clé: String, version: u64 },
}

#[derive(Debug, thiserror::Error)]
pub enum ErreurWAL {
    #[error("I/O : {0}")]
    IO(#[from] std::io::Error),
    #[error("Sérialisation : {0}")]
    Codec(#[from] bincode::Error),
    #[error("WAL corrompu à l'offset {offset}")]
    Corrompu { offset: u64 },
}

pub struct WAL {
    chemin: PathBuf,
    écrivain: BufWriter<File>,
    offset: u64,
}

impl WAL {
    pub fn ouvrir(répertoire: &Path) -> Result<Self, ErreurWAL> {
        std::fs::create_dir_all(répertoire)?;
        let chemin = répertoire.join("wal.bin");

        let fichier = OpenOptions::new()
            .create(true)
            .append(true)
            .open(&chemin)?;

        let offset = fichier.metadata()?.len();

        Ok(WAL {
            chemin,
            écrivain: BufWriter::with_capacity(64 * 1024, fichier),
            offset,
        })
    }

    /// Écrire une opération — durable après retour
    pub fn écrire(&mut self, op: &OpérationWAL) -> Result<(), ErreurWAL> {
        let octets = bincode::serialize(op)?;
        let longueur = octets.len() as u32;

        // Checksum CRC32 pour détecter la corruption
        let crc = calculer_crc32(&octets);

        self.écrivain.write_all(&longueur.to_le_bytes())?;
        self.écrivain.write_all(&crc.to_le_bytes())?;
        self.écrivain.write_all(&octets)?;
        self.écrivain.flush()?; // fsync implicite via flush

        self.offset += 4 + 4 + octets.len() as u64;
        Ok(())
    }

    /// Rejouer le WAL depuis le début
    pub fn rejouer(&self) -> Result<Vec<OpérationWAL>, ErreurWAL> {
        let mut fichier = File::open(&self.chemin)?;
        let mut opérations = Vec::new();
        let mut offset = 0u64;

        loop {
            let mut buf_longueur = [0u8; 4];
            match fichier.read_exact(&mut buf_longueur) {
                Ok(()) => {}
                Err(e) if e.kind() == std::io::ErrorKind::UnexpectedEof => break,
                Err(e) => return Err(ErreurWAL::IO(e)),
            }

            let longueur = u32::from_le_bytes(buf_longueur) as usize;

            let mut buf_crc = [0u8; 4];
            fichier.read_exact(&mut buf_crc)?;
            let crc_attendu = u32::from_le_bytes(buf_crc);

            let mut données = vec![0u8; longueur];
            fichier.read_exact(&mut données)?;

            // Vérifier l'intégrité
            let crc_calculé = calculer_crc32(&données);
            if crc_calculé != crc_attendu {
                return Err(ErreurWAL::Corrompu { offset });
            }

            let op: OpérationWAL = bincode::deserialize(&données)?;
            opérations.push(op);

            offset += 4 + 4 + longueur as u64;
        }

        tracing::info!("{} opérations chargées depuis le WAL", opérations.len());
        Ok(opérations)
    }

    /// Taille actuelle du WAL
    pub fn taille(&self) -> u64 {
        self.offset
    }

    /// Tronquer le WAL (après un snapshot)
    pub fn tronquer(&mut self) -> Result<(), ErreurWAL> {
        drop(std::mem::replace(
            &mut self.écrivain,
            BufWriter::new(
                OpenOptions::new()
                    .create(true)
                    .write(true)
                    .truncate(true)
                    .open(&self.chemin)?
            ),
        ));
        self.offset = 0;
        Ok(())
    }
}

fn calculer_crc32(données: &[u8]) -> u32 {
    // Implémentation simple — en production utiliser crc32fast
    données.iter().fold(0u32, |acc, &b| {
        acc.wrapping_add(b as u32).wrapping_mul(1664525).wrapping_add(1013904223)
    })
}
```

---

## Étape 3 — Mémtable et Shard (src/store/shard.rs)

```rust
// src/store/shard.rs
use std::collections::HashMap;
use std::path::PathBuf;
use tokio::sync::RwLock;
use super::wal::{WAL, OpérationWAL, ErreurWAL};

#[derive(Debug, thiserror::Error)]
pub enum ErreurShard {
    #[error("WAL : {0}")]
    WAL(#[from] ErreurWAL),
    #[error("Clé introuvable")]
    Introuvable,
}

pub struct Shard {
    mémtable: RwLock<HashMap<String, Vec<u8>>>,
    wal: tokio::sync::Mutex<WAL>,
    version: std::sync::atomic::AtomicU64,
    taille_max_wal: u64,
}

impl Shard {
    pub fn nouveau(répertoire: PathBuf, taille_max_wal: u64) -> Result<Self, ErreurShard> {
        let mut wal = WAL::ouvrir(&répertoire)?;
        let mut mémtable = HashMap::new();

        // Rejouer le WAL pour reconstruire l'état
        let mut version_max = 0u64;
        for op in wal.rejouer()? {
            match op {
                OpérationWAL::Set { clé, valeur, version } => {
                    mémtable.insert(clé, valeur);
                    version_max = version_max.max(version);
                }
                OpérationWAL::Delete { clé, version } => {
                    mémtable.remove(&clé);
                    version_max = version_max.max(version);
                }
            }
        }

        Ok(Shard {
            mémtable: RwLock::new(mémtable),
            wal: tokio::sync::Mutex::new(wal),
            version: std::sync::atomic::AtomicU64::new(version_max),
            taille_max_wal,
        })
    }

    pub async fn set(&self, clé: &str, valeur: Vec<u8>) -> Result<u64, ErreurShard> {
        let version = self.version
            .fetch_add(1, std::sync::atomic::Ordering::SeqCst) + 1;

        // Écrire dans le WAL d'abord (durabilité)
        {
            let mut wal = self.wal.lock().await;
            wal.écrire(&OpérationWAL::Set {
                clé: clé.to_string(),
                valeur: valeur.clone(),
                version,
            })?;

            // Compaction si nécessaire
            if wal.taille() > self.taille_max_wal {
                drop(wal);
                self.compacter().await?;
            }
        }

        // Mise à jour de la mémtable
        self.mémtable.write().await.insert(clé.to_string(), valeur);
        Ok(version)
    }

    pub async fn get(&self, clé: &str) -> Result<Vec<u8>, ErreurShard> {
        self.mémtable
            .read()
            .await
            .get(clé)
            .cloned()
            .ok_or(ErreurShard::Introuvable)
    }

    pub async fn delete(&self, clé: &str) -> Result<(), ErreurShard> {
        let version = self.version
            .fetch_add(1, std::sync::atomic::Ordering::SeqCst) + 1;

        {
            let mut wal = self.wal.lock().await;
            wal.écrire(&OpérationWAL::Delete {
                clé: clé.to_string(),
                version,
            })?;
        }

        self.mémtable.write().await.remove(clé);
        Ok(())
    }

    pub async fn clés_avec_préfixe(&self, préfixe: &str) -> Vec<String> {
        self.mémtable
            .read()
            .await
            .keys()
            .filter(|k| k.starts_with(préfixe))
            .cloned()
            .collect()
    }

    async fn compacter(&self) -> Result<(), ErreurShard> {
        tracing::info!("Compaction du shard en cours...");
        let snapshot: Vec<(String, Vec<u8>)> = {
            self.mémtable.read().await
                .iter()
                .map(|(k, v)| (k.clone(), v.clone()))
                .collect()
        };

        let mut wal = self.wal.lock().await;
        wal.tronquer()?;

        let version = self.version.load(std::sync::atomic::Ordering::SeqCst);
        for (clé, valeur) in snapshot {
            wal.écrire(&OpérationWAL::Set {
                clé,
                valeur,
                version,
            })?;
        }

        tracing::info!("Compaction terminée");
        Ok(())
    }

    pub async fn stats(&self) -> StatsShard {
        let table = self.mémtable.read().await;
        StatsShard {
            nb_clés: table.len(),
            taille_octets: table.values().map(|v| v.len()).sum(),
        }
    }
}

#[derive(Debug, serde::Serialize)]
pub struct StatsShard {
    pub nb_clés: usize,
    pub taille_octets: usize,
}
```

---

## Étape 4 — Routeur de shards (src/store/sharded.rs)

```rust
// src/store/sharded.rs
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};
use std::path::Path;
use std::sync::Arc;

use super::shard::{Shard, ErreurShard, StatsShard};

pub struct KVShardé {
    shards: Vec<Arc<Shard>>,
    nb_shards: usize,
}

impl KVShardé {
    pub fn nouveau(répertoire: &Path, nb_shards: usize) -> Result<Self, ErreurShard> {
        let mut shards = Vec::with_capacity(nb_shards);
        for i in 0..nb_shards {
            let chemin = répertoire.join(format!("shard-{:03}", i));
            shards.push(Arc::new(Shard::nouveau(chemin, 32 * 1024 * 1024)?));
        }
        Ok(KVShardé { shards, nb_shards })
    }

    fn index_shard(&self, clé: &str) -> usize {
        let mut h = DefaultHasher::new();
        clé.hash(&mut h);
        (h.finish() as usize) % self.nb_shards
    }

    pub async fn set(&self, clé: &str, valeur: Vec<u8>) -> Result<u64, ErreurShard> {
        self.shards[self.index_shard(clé)].set(clé, valeur).await
    }

    pub async fn get(&self, clé: &str) -> Result<Vec<u8>, ErreurShard> {
        self.shards[self.index_shard(clé)].get(clé).await
    }

    pub async fn delete(&self, clé: &str) -> Result<(), ErreurShard> {
        self.shards[self.index_shard(clé)].delete(clé).await
    }

    /// Recherche par préfixe — scanne tous les shards
    pub async fn clés_avec_préfixe(&self, préfixe: &str) -> Vec<String> {
        let mut toutes = Vec::new();
        for shard in &self.shards {
            toutes.extend(shard.clés_avec_préfixe(préfixe).await);
        }
        toutes.sort();
        toutes
    }

    pub async fn stats_globales(&self) -> Vec<StatsShard> {
        let mut stats = Vec::new();
        for shard in &self.shards {
            stats.push(shard.stats().await);
        }
        stats
    }
}
```

---

## Étape 5 — API HTTP (src/api/handlers.rs)

```rust
// src/api/handlers.rs
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::Json,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;

use crate::store::sharded::KVShardé;
use super::erreurs::ErreurAPI;

pub type AppState = Arc<KVShardé>;

// PUT /kv/:clé — écrire une valeur
pub async fn set(
    State(store): State<AppState>,
    Path(clé): Path<String>,
    body: axum::body::Bytes,
) -> Result<Json<RéponseSet>, ErreurAPI> {
    let version = store.set(&clé, body.to_vec()).await?;
    tracing::debug!(clé = %clé, version = version, "SET");
    Ok(Json(RéponseSet { clé, version }))
}

// GET /kv/:clé — lire une valeur
pub async fn get(
    State(store): State<AppState>,
    Path(clé): Path<String>,
) -> Result<axum::response::Response, ErreurAPI> {
    use axum::response::IntoResponse;
    let valeur = store.get(&clé).await?;
    Ok((StatusCode::OK, valeur).into_response())
}

// DELETE /kv/:clé — supprimer
pub async fn delete(
    State(store): State<AppState>,
    Path(clé): Path<String>,
) -> Result<StatusCode, ErreurAPI> {
    store.delete(&clé).await?;
    tracing::debug!(clé = %clé, "DELETE");
    Ok(StatusCode::NO_CONTENT)
}

// GET /kv?préfixe=xxx — recherche par préfixe
#[derive(Deserialize)]
pub struct ParamsPréfixe {
    préfixe: Option<String>,
}

pub async fn lister(
    State(store): State<AppState>,
    Query(params): Query<ParamsPréfixe>,
) -> Json<Vec<String>> {
    let préfixe = params.préfixe.as_deref().unwrap_or("");
    let clés = store.clés_avec_préfixe(préfixe).await;
    Json(clés)
}

// GET /stats — métriques du store
pub async fn stats(State(store): State<AppState>) -> Json<serde_json::Value> {
    let stats = store.stats_globales().await;
    let total_clés: usize = stats.iter().map(|s| s.nb_clés).sum();
    let total_octets: usize = stats.iter().map(|s| s.taille_octets).sum();

    Json(serde_json::json!({
        "total_clés": total_clés,
        "total_octets": total_octets,
        "nb_shards": stats.len(),
        "shards": stats,
    }))
}

#[derive(Serialize)]
pub struct RéponseSet {
    pub clé: String,
    pub version: u64,
}
```

```rust
// src/api/erreurs.rs
use axum::{http::StatusCode, response::{IntoResponse, Json, Response}};
use crate::store::shard::ErreurShard;

pub enum ErreurAPI {
    Introuvable,
    Stockage(String),
}

impl From<ErreurShard> for ErreurAPI {
    fn from(e: ErreurShard) -> Self {
        match e {
            ErreurShard::Introuvable => ErreurAPI::Introuvable,
            e => ErreurAPI::Stockage(e.to_string()),
        }
    }
}

impl IntoResponse for ErreurAPI {
    fn into_response(self) -> Response {
        let (code, msg) = match self {
            ErreurAPI::Introuvable => (StatusCode::NOT_FOUND, "Clé introuvable"),
            ErreurAPI::Stockage(ref m) => {
                tracing::error!("Erreur de stockage : {}", m);
                (StatusCode::INTERNAL_SERVER_ERROR, "Erreur de stockage")
            }
        };
        (code, Json(serde_json::json!({"erreur": msg}))).into_response()
    }
}
```

---

## Étape 6 — Main (src/main.rs)

```rust
// src/main.rs
mod api;
mod store;

use api::handlers::{self, AppState};
use axum::{routing::{delete, get, put}, Router};
use std::sync::Arc;
use store::sharded::KVShardé;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter("kv_store=debug,info")
        .init();

    let données_dir = std::path::PathBuf::from("./données-kv");
    let nb_shards = 4;

    tracing::info!("Ouverture du store ({} shards)...", nb_shards);
    let store: AppState = Arc::new(KVShardé::nouveau(&données_dir, nb_shards)?);

    let app = Router::new()
        .route("/kv/:clé", get(handlers::get).put(handlers::set).delete(handlers::delete))
        .route("/kv",      get(handlers::lister))
        .route("/stats",   get(handlers::stats))
        .with_state(store);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    tracing::info!("KV Store démarré sur http://0.0.0.0:8080");
    axum::serve(listener, app).await?;
    Ok(())
}
```

---

## Étape 7 — Tests et benchmarks

```rust
// tests/test_wal.rs
use kv_store::store::shard::Shard;
use tempfile::TempDir;

#[tokio::test]
async fn persistance_après_redémarrage() {
    let dir = TempDir::new().unwrap();

    // Premier démarrage — écrire des données
    {
        let shard = Shard::nouveau(dir.path().to_path_buf(), 1024 * 1024).unwrap();
        shard.set("clé1", b"valeur1".to_vec()).await.unwrap();
        shard.set("clé2", b"valeur2".to_vec()).await.unwrap();
        shard.delete("clé1").await.unwrap();
    }
    // Shard droppé — simule un redémarrage

    // Second démarrage — vérifier la restauration
    {
        let shard = Shard::nouveau(dir.path().to_path_buf(), 1024 * 1024).unwrap();
        assert!(shard.get("clé1").await.is_err(), "clé1 doit être supprimée");
        assert_eq!(shard.get("clé2").await.unwrap(), b"valeur2");
    }
}

#[tokio::test]
async fn concurrent_writes_sans_corruption() {
    let dir = TempDir::new().unwrap();
    let shard = std::sync::Arc::new(
        Shard::nouveau(dir.path().to_path_buf(), 64 * 1024 * 1024).unwrap()
    );

    let mut handles = vec![];
    for i in 0..50 {
        let s = shard.clone();
        handles.push(tokio::spawn(async move {
            let clé = format!("clé-{}", i);
            let val = format!("valeur-{}", i).into_bytes();
            s.set(&clé, val).await.unwrap();
        }));
    }

    for h in handles { h.await.unwrap(); }

    let stats = shard.stats().await;
    assert_eq!(stats.nb_clés, 50);
}
```

```rust
// benches/performance.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, Throughput};
use kv_store::store::sharded::KVShardé;
use tempfile::TempDir;

fn bench_écriture(c: &mut Criterion) {
    let rt = tokio::runtime::Runtime::new().unwrap();
    let dir = TempDir::new().unwrap();
    let store = rt.block_on(async {
        std::sync::Arc::new(KVShardé::nouveau(dir.path(), 4).unwrap())
    });

    let mut groupe = c.benchmark_group("kv_store");
    groupe.throughput(Throughput::Elements(1));

    groupe.bench_function("set_1kb", |b| {
        b.to_async(&rt).iter(|| async {
            let clé = format!("clé-{}", rand::random::<u32>());
            let valeur = vec![0u8; 1024];
            store.set(black_box(&clé), black_box(valeur)).await.unwrap();
        })
    });

    groupe.bench_function("get_existant", |b| {
        rt.block_on(store.set("bench-clé", b"valeur".to_vec())).unwrap();
        b.to_async(&rt).iter(|| async {
            store.get(black_box("bench-clé")).await.unwrap()
        })
    });

    groupe.finish();
}

criterion_group!(benches, bench_écriture);
criterion_main!(benches);
```

---

## Test de charge avec curl

```bash
cargo run &

# Écrire 10 000 clés
for i in $(seq 1 10000); do
  curl -s -X PUT http://localhost:8080/kv/clé-$i -d "valeur-$i" &
done
wait

# Lire en parallèle
for i in $(seq 1 1000); do
  curl -s http://localhost:8080/kv/clé-$i &
done
wait

# Statistiques
curl http://localhost:8080/stats | jq .
```

---

## 🏆 Critères de réussite

| Critère | Validé ? |
|---|---|
| Persistance : les données survivent au redémarrage | ✅ |
| Concurrence : 100 écritures simultanées sans corruption | ✅ |
| Performance : > 10 000 ops/sec en écriture | ✅ |
| Compaction : le WAL est tronqué quand il dépasse le seuil | ✅ |
| API REST : GET, PUT, DELETE, liste par préfixe, stats | ✅ |
| Intégrité : CRC32 détecte la corruption du WAL | ✅ |

---

**→ Suivant : [Module 4 — Systèmes Distribués et Consensus](../../module-4/README.md)**
