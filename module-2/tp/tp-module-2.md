# 🛠️ TP Module 2 — API REST Sécurisée avec JWT et PostgreSQL

> **Durée estimée :** 5 à 7 heures
> **Compétence validée :** Backend sécurisé prêt pour la production

---

## Architecture cible

```
Client HTTP
    │
    ▼
┌───────────────────────────────────────┐
│  Axum Router                          │
│  ├── POST /auth/inscription           │
│  ├── POST /auth/connexion             │
│  └── /api/v1/* (protégé par JWT)     │
│       ├── GET    /utilisateurs        │
│       ├── POST   /utilisateurs        │
│       ├── GET    /utilisateurs/:id   │
│       ├── PUT    /utilisateurs/:id   │
│       └── DELETE /utilisateurs/:id  │
└─────────────────┬─────────────────────┘
                  │
            ┌─────▼──────┐
            │ PostgreSQL  │
            │  (sqlx)     │
            └────────────┘
```

---

## Étape 1 — Setup

```bash
# Docker pour PostgreSQL
docker run -d \
  --name postgres-dev \
  -e POSTGRES_DB=mon_api \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=motdepasse \
  -p 5432:5432 \
  postgres:16

# Variables d'environnement
export DATABASE_URL="postgresql://admin:motdepasse@localhost/mon_api"
export JWT_SECRET="un-secret-très-long-et-aléatoire-au-moins-256-bits"
export PORT=8080
```

```toml
# Cargo.toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
jsonwebtoken = "9"
bcrypt = "0.15"
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
validator = { version = "0.18", features = ["derive"] }
tower-http = { version = "0.5", features = ["cors", "trace"] }
tracing = "0.1"
tracing-subscriber = "0.3"
anyhow = "1"
thiserror = "1"
```

---

## Étape 2 — Migration SQL

```sql
-- migrations/001_init.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE utilisateurs (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    nom         VARCHAR(100) NOT NULL,
    email       VARCHAR(254) NOT NULL UNIQUE,
    mot_de_passe_hash VARCHAR(255) NOT NULL,
    rôle        VARCHAR(50) NOT NULL DEFAULT 'utilisateur',
    actif       BOOLEAN NOT NULL DEFAULT true,
    créé_le     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    modifié_le  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_utilisateurs_email ON utilisateurs(email);

-- Trigger de mise à jour automatique de modifié_le
CREATE OR REPLACE FUNCTION maj_modifié_le()
RETURNS TRIGGER AS $$
BEGIN
    NEW.modifié_le = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_maj_modifié_le
    BEFORE UPDATE ON utilisateurs
    FOR EACH ROW EXECUTE FUNCTION maj_modifié_le();
```

---

## Étape 3 — Service JWT (src/services/jwt.rs)

```rust
// src/services/jwt.rs
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use std::time::{Duration, SystemTime, UNIX_EPOCH};
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ErreurJWT {
    #[error("Token invalide")]
    Invalide,
    #[error("Token expiré")]
    Expiré,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,     // ID utilisateur
    pub email: String,
    pub rôle: String,
    pub exp: u64,        // Expiration (timestamp UNIX)
    pub iat: u64,        // Emission (timestamp UNIX)
}

pub struct ServiceJWT {
    clé_encodage: EncodingKey,
    clé_décodage: DecodingKey,
}

impl ServiceJWT {
    pub fn nouveau(secret: &str) -> Self {
        ServiceJWT {
            clé_encodage: EncodingKey::from_secret(secret.as_bytes()),
            clé_décodage: DecodingKey::from_secret(secret.as_bytes()),
        }
    }
    
    pub fn émettre(
        &self,
        id: &str,
        email: &str,
        rôle: &str,
        durée_validité: Duration,
    ) -> Result<String, ErreurJWT> {
        let maintenant = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();
        
        let claims = Claims {
            sub: id.to_string(),
            email: email.to_string(),
            rôle: rôle.to_string(),
            iat: maintenant,
            exp: maintenant + durée_validité.as_secs(),
        };
        
        encode(&Header::default(), &claims, &self.clé_encodage)
            .map_err(|_| ErreurJWT::Invalide)
    }
    
    pub fn vérifier(&self, token: &str) -> Result<Claims, ErreurJWT> {
        decode::<Claims>(token, &self.clé_décodage, &Validation::default())
            .map(|données| données.claims)
            .map_err(|e| {
                if e.kind() == &jsonwebtoken::errors::ErrorKind::ExpiredSignature {
                    ErreurJWT::Expiré
                } else {
                    ErreurJWT::Invalide
                }
            })
    }
}
```

---

## Étape 4 — Repository PostgreSQL (src/db/repo_utilisateurs.rs)

```rust
// src/db/repo_utilisateurs.rs
use sqlx::{PgPool, Row};
use uuid::Uuid;

use crate::modèles::Utilisateur;
use crate::erreurs::ErreurAPI;

pub struct RepoUtilisateurs {
    pool: PgPool,
}

impl RepoUtilisateurs {
    pub fn nouveau(pool: PgPool) -> Self {
        RepoUtilisateurs { pool }
    }
    
    pub async fn créer(
        &self,
        nom: &str,
        email: &str,
        hash_mdp: &str,
    ) -> Result<Utilisateur, ErreurAPI> {
        sqlx::query_as!(
            Utilisateur,
            r#"
            INSERT INTO utilisateurs (nom, email, mot_de_passe_hash)
            VALUES ($1, $2, $3)
            RETURNING id, nom, email, rôle, actif, créé_le, modifié_le
            "#,
            nom, email, hash_mdp
        )
        .fetch_one(&self.pool)
        .await
        .map_err(|e| match e {
            sqlx::Error::Database(ref db_err) if db_err.is_unique_violation() => {
                ErreurAPI::Conflit(format!("Email '{}' déjà utilisé", email))
            }
            e => ErreurAPI::Interne(e.into()),
        })
    }
    
    pub async fn obtenir_par_email(
        &self,
        email: &str,
    ) -> Result<Option<UtilisateurAvecHash>, ErreurAPI> {
        sqlx::query_as!(
            UtilisateurAvecHash,
            "SELECT id, nom, email, mot_de_passe_hash, rôle, actif FROM utilisateurs WHERE email = $1",
            email
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| ErreurAPI::Interne(e.into()))
    }
    
    pub async fn lister(
        &self,
        page: i64,
        par_page: i64,
    ) -> Result<(Vec<Utilisateur>, i64), ErreurAPI> {
        let offset = (page - 1) * par_page;
        
        let utilisateurs = sqlx::query_as!(
            Utilisateur,
            r#"
            SELECT id, nom, email, rôle, actif, créé_le, modifié_le
            FROM utilisateurs
            WHERE actif = true
            ORDER BY créé_le DESC
            LIMIT $1 OFFSET $2
            "#,
            par_page, offset
        )
        .fetch_all(&self.pool)
        .await
        .map_err(|e| ErreurAPI::Interne(e.into()))?;
        
        let total: i64 = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM utilisateurs WHERE actif = true"
        )
        .fetch_one(&self.pool)
        .await
        .map_err(|e| ErreurAPI::Interne(e.into()))?
        .unwrap_or(0);
        
        Ok((utilisateurs, total))
    }
}

#[derive(sqlx::FromRow)]
pub struct UtilisateurAvecHash {
    pub id: Uuid,
    pub nom: String,
    pub email: String,
    pub mot_de_passe_hash: String,
    pub rôle: String,
    pub actif: bool,
}
```

---

## Étape 5 — Handler d'authentification

```rust
// src/handlers/auth.rs
use axum::{extract::State, response::Json};
use bcrypt::{hash, verify, DEFAULT_COST};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use std::time::Duration;
use validator::Validate;

use crate::{erreurs::ErreurAPI, état::ÉtatApp};

#[derive(Deserialize, Validate)]
pub struct RequêteInscription {
    #[validate(length(min = 2, max = 100))]
    pub nom: String,
    #[validate(email)]
    pub email: String,
    #[validate(length(min = 8))]
    pub mot_de_passe: String,
}

#[derive(Deserialize)]
pub struct RequêteConnexion {
    pub email: String,
    pub mot_de_passe: String,
}

#[derive(Serialize)]
pub struct RéponseAuth {
    pub token: String,
    pub expire_dans: u64,
    pub utilisateur_id: String,
}

pub async fn inscription(
    State(état): State<Arc<ÉtatApp>>,
    Json(corps): Json<RequêteInscription>,
) -> Result<(axum::http::StatusCode, Json<RéponseAuth>), ErreurAPI> {
    corps.validate().map_err(ErreurAPI::ValidationÉchouée)?;
    
    // Hash du mot de passe (dans spawn_blocking — bcrypt est CPU-intensif)
    let mdp = corps.mot_de_passe.clone();
    let hash_mdp = tokio::task::spawn_blocking(move || {
        hash(&mdp, DEFAULT_COST)
    }).await
    .map_err(|e| ErreurAPI::Interne(e.into()))?
    .map_err(|e| ErreurAPI::Interne(e.into()))?;
    
    let utilisateur = état.repo.créer(&corps.nom, &corps.email, &hash_mdp).await?;
    
    let durée = Duration::from_secs(24 * 3600); // 24h
    let token = état.jwt.émettre(
        &utilisateur.id.to_string(),
        &utilisateur.email,
        &utilisateur.rôle,
        durée,
    ).map_err(|_| ErreurAPI::Interne(anyhow::anyhow!("Erreur JWT")))?;
    
    Ok((
        axum::http::StatusCode::CREATED,
        Json(RéponseAuth {
            token,
            expire_dans: durée.as_secs(),
            utilisateur_id: utilisateur.id.to_string(),
        }),
    ))
}

pub async fn connexion(
    State(état): State<Arc<ÉtatApp>>,
    Json(corps): Json<RequêteConnexion>,
) -> Result<Json<RéponseAuth>, ErreurAPI> {
    let utilisateur = état.repo
        .obtenir_par_email(&corps.email).await?
        .ok_or(ErreurAPI::NonAuthentifié)?;
    
    if !utilisateur.actif {
        return Err(ErreurAPI::Interdit);
    }
    
    // Vérification du mot de passe (CPU-intensif → spawn_blocking)
    let mdp_soumis = corps.mot_de_passe.clone();
    let hash_stocké = utilisateur.mot_de_passe_hash.clone();
    
    let valide = tokio::task::spawn_blocking(move || {
        verify(&mdp_soumis, &hash_stocké)
    }).await
    .map_err(|e| ErreurAPI::Interne(e.into()))?
    .map_err(|e| ErreurAPI::Interne(e.into()))?;
    
    if !valide {
        return Err(ErreurAPI::NonAuthentifié);
    }
    
    let durée = Duration::from_secs(24 * 3600);
    let token = état.jwt.émettre(
        &utilisateur.id.to_string(),
        &utilisateur.email,
        &utilisateur.rôle,
        durée,
    ).map_err(|_| ErreurAPI::Interne(anyhow::anyhow!("Erreur JWT")))?;
    
    Ok(Json(RéponseAuth {
        token,
        expire_dans: durée.as_secs(),
        utilisateur_id: utilisateur.id.to_string(),
    }))
}
```

---

## Lancer et tester

```bash
# Appliquer les migrations
sqlx migrate run

# Lancer le serveur
cargo run

# Tester avec curl
# Inscription
curl -X POST http://localhost:8080/auth/inscription \
  -H "Content-Type: application/json" \
  -d '{"nom":"Alice","email":"alice@test.com","mot_de_passe":"motdepasse123"}'

# Connexion
TOKEN=$(curl -s -X POST http://localhost:8080/auth/connexion \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@test.com","mot_de_passe":"motdepasse123"}' \
  | jq -r '.token')

# Lister les utilisateurs (protégé)
curl http://localhost:8080/api/v1/utilisateurs \
  -H "Authorization: Bearer $TOKEN"
```

---

**→ Suivant : [Module 3 — Stockage et Moteur Clé-Valeur](../../module-3/README.md)**
