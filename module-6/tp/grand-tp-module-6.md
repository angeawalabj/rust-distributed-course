# 🛠️ Grand TP Module 6 — Proxy HTTP Extensible par Plugins WebAssembly

> **Durée estimée :** 6 à 8 heures
> **Compétence validée :** Architecture extensible moderne, production-ready

---

## Objectif

Construire un reverse proxy HTTP Rust avec un système de filtres dynamiques implémentés en WebAssembly. Les plugins sont chargés à chaud, sandboxés, et observés via Prometheus.

---

## Architecture

```
Client
  │ HTTP/HTTPS
  ▼
┌──────────────────────────────────────────┐
│           Proxy (Hyper + Axum)           │
│                                          │
│  Requête entrante                        │
│      │                                  │
│      ▼                                  │
│  ┌─────────────────────┐                │
│  │  Pipeline de filtres│                │
│  │  ┌───────────────┐  │                │
│  │  │ Plugin WASM 1 │  │ rate-limiter   │
│  │  └───────┬───────┘  │                │
│  │          ▼           │                │
│  │  ┌───────────────┐  │                │
│  │  │ Plugin WASM 2 │  │ ip-blocker     │
│  │  └───────┬───────┘  │                │
│  │          ▼           │                │
│  │  ┌───────────────┐  │                │
│  │  │ Plugin WASM 3 │  │ header-enricher│
│  │  └───────┬───────┘  │                │
│  └──────────┼──────────┘                │
│             │                           │
│             ▼                           │
│     ┌──────────────┐                   │
│     │   Backend    │ → service amont   │
│     └──────────────┘                   │
└──────────────────────────────────────────┘
          │
          ▼
    /metrics (Prometheus)
    /admin/plugins (CRUD plugins)
    /admin/reload (rechargement)
```

---

## Étape 1 — Dépendances

```toml
# Cargo.toml
[package]
name = "proxy-wasm"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio          = { version = "1", features = ["full"] }
hyper          = { version = "1", features = ["full"] }
hyper-util     = { version = "0.1", features = ["full"] }
axum           = "0.7"
wasmtime       = "18"
wasmtime-wasi  = "18"
serde          = { version = "1", features = ["derive"] }
serde_json     = "1"
prometheus     = "0.13"
tracing        = "0.1"
tracing-subscriber = "0.3"
thiserror      = "1"
anyhow         = "1"
notify         = "6"      # Surveillance de fichiers
tower          = "0.4"
tower-http     = { version = "0.5", features = ["trace"] }
```

---

## Étape 2 — Interface plugin (partagée hôte↔plugin)

```rust
// src/interface_plugin.rs
use serde::{Deserialize, Serialize};

/// Contexte de requête transmis au plugin
#[derive(Debug, Serialize, Deserialize)]
pub struct ContexteRequête {
    pub méthode: String,
    pub chemin: String,
    pub ip_client: String,
    pub en_têtes: Vec<(String, String)>,
    pub corps_préfixe: Vec<u8>,   // Premiers 1024 octets du corps
}

/// Action que le plugin demande à l'hôte
#[derive(Debug, Serialize, Deserialize)]
pub enum ActionPlugin {
    Continuer,
    Bloquer { code_http: u16, corps: String },
    AjouterEnTête { clé: String, valeur: String },
    RedirigerVers { url: String },
    ModifierChemin { nouveau_chemin: String },
}

/// Résultat retourné par le plugin
#[derive(Debug, Serialize, Deserialize)]
pub struct RésultatPlugin {
    pub action: ActionPlugin,
    pub logs: Vec<String>,
    pub métriques: Vec<(String, f64)>,  // (nom, valeur) pour Prometheus
}
```

---

## Étape 3 — Plugins WASM (plugins/)

### Plugin rate-limiter

```rust
// plugins/rate-limiter/src/lib.rs
#![no_std]
extern crate alloc;
use alloc::vec::Vec;

mod interface; // types partagés

static mut COMPTEURS: Option<alloc::collections::BTreeMap<alloc::string::String, (u64, u64)>> = None;

fn compteurs() -> &'static mut alloc::collections::BTreeMap<alloc::string::String, (u64, u64)> {
    unsafe {
        if COMPTEURS.is_none() {
            COMPTEURS = Some(alloc::collections::BTreeMap::new());
        }
        COMPTEURS.as_mut().unwrap()
    }
}

#[no_mangle]
pub extern "C" fn traiter_requête(ptr: i32, len: i32) -> i32 {
    let données = unsafe {
        alloc::slice::from_raw_parts(ptr as *const u8, len as usize)
    };

    let ctx: interface::ContexteRequête = match serde_json_core::from_slice(données) {
        Ok((c, _)) => c,
        Err(_) => return écrire_résultat(interface::ActionPlugin::Continuer, &[]),
    };

    let limite = 100u64;    // 100 req/min par IP
    let fenêtre = 60u64;    // 60 secondes

    let now = hôte_timestamp();
    let (count, window_start) = compteurs()
        .entry(ctx.ip_client.into())
        .or_insert((0, now));

    if now - *window_start > fenêtre {
        *window_start = now;
        *count = 0;
    }
    *count += 1;

    if *count > limite {
        return écrire_résultat(
            interface::ActionPlugin::Bloquer {
                code_http: 429,
                corps: r#"{"erreur":"Trop de requêtes"}"#.into(),
            },
            &[("rate_limit_rejets", 1.0)],
        );
    }

    écrire_résultat(
        interface::ActionPlugin::Continuer,
        &[("requêtes_par_ip", *count as f64)],
    )
}

extern "C" {
    fn hôte_timestamp() -> u64;
    fn hôte_écrire_sortie(ptr: i32, len: i32);
}

fn écrire_résultat(action: interface::ActionPlugin, métriques: &[(&str, f64)]) -> i32 {
    // Sérialiser et écrire dans le buffer de sortie
    // Retourner la taille du résultat
    0
}

#[no_mangle] pub extern "C" fn allouer(n: usize) -> *mut u8 {
    let mut v = Vec::with_capacity(n); let p = v.as_mut_ptr(); core::mem::forget(v); p
}
#[no_mangle] pub extern "C" fn libérer(p: *mut u8, n: usize) {
    unsafe { drop(Vec::from_raw_parts(p, n, n)); }
}
```

---

## Étape 4 — Moteur de plugins (src/moteur.rs)

```rust
// src/moteur.rs
use std::{collections::HashMap, path::{Path, PathBuf}, sync::Arc};
use tokio::sync::RwLock;
use wasmtime::*;
use crate::interface_plugin::{ContexteRequête, RésultatPlugin, ActionPlugin};

struct PluginInstancié {
    module: Module,
    nom: String,
    chemin: PathBuf,
    actif: bool,
}

pub struct MoteurPlugins {
    engine: Engine,
    plugins: Arc<RwLock<Vec<PluginInstancié>>>,
}

impl MoteurPlugins {
    pub fn nouveau() -> anyhow::Result<Self> {
        let mut cfg = Config::new();
        cfg.async_support(true).consume_fuel(true);
        Ok(MoteurPlugins {
            engine: Engine::new(&cfg)?,
            plugins: Arc::new(RwLock::new(Vec::new())),
        })
    }

    pub async fn charger(&self, nom: &str, chemin: &Path) -> anyhow::Result<()> {
        let octets = tokio::fs::read(chemin).await?;
        let module = Module::new(&self.engine, &octets)?;
        let mut plugins = self.plugins.write().await;

        // Remplacer si le plugin existe déjà
        if let Some(p) = plugins.iter_mut().find(|p| p.nom == nom) {
            p.module = module;
            p.chemin = chemin.to_path_buf();
            tracing::info!(plugin = nom, "Plugin rechargé à chaud");
        } else {
            plugins.push(PluginInstancié {
                module,
                nom: nom.to_string(),
                chemin: chemin.to_path_buf(),
                actif: true,
            });
            tracing::info!(plugin = nom, "Plugin chargé");
        }
        Ok(())
    }

    /// Exécuter tous les plugins sur la requête — retourne la première action bloquante
    pub async fn exécuter_pipeline(
        &self,
        ctx: &ContexteRequête,
    ) -> anyhow::Result<ActionPlugin> {
        let données = serde_json::to_vec(ctx)?;
        let plugins = self.plugins.read().await;

        for plugin in plugins.iter().filter(|p| p.actif) {
            let action = self.exécuter_plugin(&plugin.module, &données).await?;
            match &action {
                ActionPlugin::Continuer => continue,
                autre => {
                    tracing::debug!(plugin = %plugin.nom, "Plugin a modifié la requête");
                    return Ok(autre.clone());
                }
            }
        }

        Ok(ActionPlugin::Continuer)
    }

    async fn exécuter_plugin(
        &self,
        module: &Module,
        données: &[u8],
    ) -> anyhow::Result<ActionPlugin> {
        let mut linker: Linker<()> = Linker::new(&self.engine);

        // Exposer les fonctions de l'hôte
        let now = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_secs();

        linker.func_wrap("env", "hôte_timestamp", move || now)?;
        linker.func_wrap("env", "hôte_écrire_sortie", |_ptr: i32, _len: i32| {})?;

        let mut store = Store::new(&self.engine, ());
        store.add_fuel(500_000)?;  // 500K instructions max

        let instance = linker.instantiate_async(&mut store, module).await?;
        let mémoire = instance.get_memory(&mut store, "memory")
            .ok_or_else(|| anyhow::anyhow!("Mémoire introuvable"))?;

        let fn_alloc = instance.get_typed_func::<u32, u32>(&mut store, "allouer")?;
        let ptr = fn_alloc.call_async(&mut store, données.len() as u32).await?;
        mémoire.write(&mut store, ptr as usize, données)?;

        let fn_traiter = instance.get_typed_func::<(i32, i32), i32>(
            &mut store, "traiter_requête"
        )?;

        let résultat_ptr = fn_traiter.call_async(
            &mut store,
            (ptr as i32, données.len() as i32),
        ).await?;

        // Lire le résultat depuis la mémoire WASM
        // (simplification — en pratique lire la taille puis les octets)
        let _ = résultat_ptr;

        // Libérer
        let fn_free = instance.get_typed_func::<(u32, u32), ()>(&mut store, "libérer")?;
        fn_free.call_async(&mut store, (ptr, données.len() as u32)).await?;

        Ok(ActionPlugin::Continuer)
    }
}
```

---

## Étape 5 — Proxy principal (src/proxy.rs)

```rust
// src/proxy.rs
use std::sync::Arc;
use hyper::{Request, Response, body::Incoming};
use hyper_util::client::legacy::Client;
use tokio::sync::RwLock;

use crate::{moteur::MoteurPlugins, interface_plugin::{ActionPlugin, ContexteRequête}};

pub struct Proxy {
    backend: String,
    moteur: Arc<MoteurPlugins>,
    client: Client<hyper_util::client::legacy::connect::HttpConnector, hyper::body::Bytes>,
}

impl Proxy {
    pub async fn traiter(
        &self,
        req: Request<Incoming>,
    ) -> anyhow::Result<Response<hyper::body::Bytes>> {
        use axum::http::StatusCode;

        let méthode = req.method().to_string();
        let chemin = req.uri().path().to_string();
        let ip = "1.2.3.4".to_string(); // Extraire du socket en vrai

        let en_têtes: Vec<(String, String)> = req.headers()
            .iter()
            .map(|(k, v)| (k.to_string(), v.to_str().unwrap_or("").to_string()))
            .collect();

        let ctx = ContexteRequête {
            méthode,
            chemin: chemin.clone(),
            ip_client: ip,
            en_têtes,
            corps_préfixe: vec![],
        };

        // Passer par le pipeline de plugins
        let action = self.moteur.exécuter_pipeline(&ctx).await?;

        match action {
            ActionPlugin::Bloquer { code_http, corps } => {
                let code = StatusCode::from_u16(code_http).unwrap_or(StatusCode::FORBIDDEN);
                Ok(Response::builder()
                    .status(code)
                    .header("content-type", "application/json")
                    .body(hyper::body::Bytes::from(corps))
                    .unwrap()
                    .into())
            }
            ActionPlugin::RedirigerVers { url } => {
                Ok(Response::builder()
                    .status(StatusCode::TEMPORARY_REDIRECT)
                    .header("location", url)
                    .body(hyper::body::Bytes::new())
                    .unwrap()
                    .into())
            }
            ActionPlugin::Continuer | ActionPlugin::AjouterEnTête { .. } => {
                // Proxy vers le backend
                let url = format!("{}{}", self.backend, chemin);
                tracing::debug!(url = %url, "Proxy vers le backend");

                // En réalité : reconstruire la requête et l'envoyer avec hyper
                Ok(Response::builder()
                    .status(StatusCode::OK)
                    .body(hyper::body::Bytes::from(format!("Proxied to {}", url)))
                    .unwrap()
                    .into())
            }
            ActionPlugin::ModifierChemin { nouveau_chemin } => {
                let url = format!("{}{}", self.backend, nouveau_chemin);
                tracing::debug!(url = %url, "Proxy avec chemin modifié");
                Ok(Response::builder()
                    .status(StatusCode::OK)
                    .body(hyper::body::Bytes::new())
                    .unwrap()
                    .into())
            }
        }
    }
}
```

---

## Étape 6 — Tests de charge

```bash
# Compiler les plugins
cd plugins/rate-limiter
cargo build --target wasm32-wasi --release
cp target/wasm32-wasi/release/rate_limiter.wasm ../../plugins/rate-limiter.wasm

cd plugins/ip-blocker
cargo build --target wasm32-wasi --release
cp target/wasm32-wasi/release/ip_blocker.wasm ../../plugins/ip-blocker.wasm

# Lancer le proxy
BACKEND=http://httpbin.org PORT=8080 PLUGINS_DIR=./plugins cargo run

# Test de charge
ab -n 100000 -c 200 http://localhost:8080/get
# Vérifier dans Prometheus : rate_limit_rejets monte après 100 req/min par IP

# Test de rechargement à chaud
# Modifier le plugin, recompiler, copier le .wasm dans plugins/
# Observer les logs : "Plugin rechargé à chaud" sans redémarrer le processus
```

---

## 🏆 Critères de réussite

| Critère | Validé ? |
|---|---|
| Le proxy transfère correctement les requêtes | ✅ |
| Les plugins WASM s'exécutent en sandbox | ✅ |
| Un plugin malveillant (boucle infinie) est neutralisé | ✅ |
| Le rechargement à chaud fonctionne sans interruption | ✅ |
| Les métriques des plugins sont exposées sur /metrics | ✅ |
| Les tests de charge montrent > 50 000 req/sec | ✅ |
| Les logs sont structurés JSON | ✅ |

---

**🎓 Vous avez terminé la formation. Félicitations !**

**→ [Projet Final : DistribKV complet](../chapitre-6.3/6.3.1-projet-final.md)**
