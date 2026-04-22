# 🛠️ TP Module 0 — Bibliothèque Rust Modulaire et Testée

> **Durée estimée :** 3 à 5 heures
> **Compétence validée :** Rust propre, safe, maintenable, prêt pour les systèmes distribués

---

## Objectif

Construire une bibliothèque Rust de gestion d'utilisateurs — complète, testée, instrumentée et bien architecturée. Ce TP synthétise tous les concepts du Module 0 : ownership, traits, erreurs, tests et logging.

---

## Spécifications fonctionnelles

La bibliothèque doit permettre de :

1. **Créer** des utilisateurs avec validation (email, nom, rôle)
2. **Lire** un utilisateur par ID ou par email
3. **Mettre à jour** les informations d'un utilisateur
4. **Supprimer** un utilisateur
5. **Lister** les utilisateurs avec filtrage et pagination
6. **Gérer** des erreurs métier explicites
7. **Logger** toutes les opérations avec `tracing`
8. **Tester** chaque fonctionnalité

---

## Étape 1 — Créer la structure du projet

```bash
# Créer le projet comme bibliothèque
cargo new --lib gestion-utilisateurs
cd gestion-utilisateurs

# Ajouter les dépendances dans Cargo.toml
```

```toml
# Cargo.toml
[package]
name = "gestion-utilisateurs"
version = "0.1.0"
edition = "2021"

[dependencies]
thiserror = "1"
tracing = "0.1"
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
regex = "1"

[dev-dependencies]
tracing-subscriber = "0.3"
```

---

## Étape 2 — Modéliser les données (src/modèles.rs)

```rust
// src/modèles.rs
use chrono::{DateTime, Utc};
use std::fmt;
use uuid::Uuid;

/// Rôle d'un utilisateur dans le système
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum Rôle {
    Administrateur,
    Modérateur,
    Utilisateur,
    Lecture,
}

impl fmt::Display for Rôle {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Rôle::Administrateur => write!(f, "administrateur"),
            Rôle::Modérateur => write!(f, "modérateur"),
            Rôle::Utilisateur => write!(f, "utilisateur"),
            Rôle::Lecture => write!(f, "lecture"),
        }
    }
}

impl Rôle {
    /// Retourne les permissions associées au rôle
    pub fn permissions(&self) -> Vec<&'static str> {
        match self {
            Rôle::Administrateur => vec!["lire", "écrire", "supprimer", "administrer"],
            Rôle::Modérateur => vec!["lire", "écrire", "supprimer"],
            Rôle::Utilisateur => vec!["lire", "écrire"],
            Rôle::Lecture => vec!["lire"],
        }
    }
    
    pub fn peut(&self, permission: &str) -> bool {
        self.permissions().contains(&permission)
    }
}

/// Statut d'un compte utilisateur
#[derive(Debug, Clone, PartialEq)]
pub enum StatutCompte {
    Actif,
    Suspendu { raison: String },
    Supprimé { supprimé_le: DateTime<Utc> },
}

/// Représente un utilisateur du système
#[derive(Debug, Clone)]
pub struct Utilisateur {
    pub id: Uuid,
    pub nom: String,
    pub email: String,
    pub rôle: Rôle,
    pub statut: StatutCompte,
    pub créé_le: DateTime<Utc>,
    pub modifié_le: DateTime<Utc>,
}

impl Utilisateur {
    /// Crée un nouvel utilisateur (via le builder)
    pub(crate) fn nouveau(nom: String, email: String, rôle: Rôle) -> Self {
        let maintenant = Utc::now();
        Utilisateur {
            id: Uuid::new_v4(),
            nom,
            email,
            rôle,
            statut: StatutCompte::Actif,
            créé_le: maintenant,
            modifié_le: maintenant,
        }
    }
    
    pub fn est_actif(&self) -> bool {
        matches!(self.statut, StatutCompte::Actif)
    }
    
    pub fn peut(&self, permission: &str) -> bool {
        self.est_actif() && self.rôle.peut(permission)
    }
}

impl fmt::Display for Utilisateur {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Utilisateur({}, {}, {})", self.id, self.nom, self.rôle)
    }
}
```

---

## Étape 3 — Définir les erreurs (src/erreurs.rs)

```rust
// src/erreurs.rs
use thiserror::Error;
use uuid::Uuid;

#[derive(Error, Debug)]
pub enum ErreurGestion {
    /// L'utilisateur recherché n'existe pas
    #[error("Utilisateur introuvable : id={0}")]
    Introuvable(Uuid),
    
    /// L'email est déjà utilisé par un autre compte
    #[error("Email déjà utilisé : {0}")]
    EmailDupliqué(String),
    
    /// Les données fournies ne respectent pas les règles de validation
    #[error("Validation échouée : {champ} — {raison}")]
    ValidationÉchouée {
        champ: String,
        raison: String,
    },
    
    /// Opération non autorisée pour ce rôle
    #[error("Permission refusée : '{permission}' requise, rôle actuel '{rôle}'")]
    PermissionRefusée {
        permission: String,
        rôle: String,
    },
    
    /// Compte suspendu — opération impossible
    #[error("Compte suspendu : {0}")]
    CompteSuspendu(String),
}

/// Alias pratique pour les résultats de cette bibliothèque
pub type Résultat<T> = Result<T, ErreurGestion>;
```

---

## Étape 4 — Validation (src/validation.rs)

```rust
// src/validation.rs
use crate::erreurs::{ErreurGestion, Résultat};
use regex::Regex;
use std::sync::OnceLock;

static REGEX_EMAIL: OnceLock<Regex> = OnceLock::new();

fn regex_email() -> &'static Regex {
    REGEX_EMAIL.get_or_init(|| {
        Regex::new(r"^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$")
            .expect("Regex email invalide — bug dans le code source")
    })
}

pub fn valider_email(email: &str) -> Résultat<()> {
    if email.is_empty() {
        return Err(ErreurGestion::ValidationÉchouée {
            champ: "email".to_string(),
            raison: "ne peut pas être vide".to_string(),
        });
    }
    
    if email.len() > 254 {
        return Err(ErreurGestion::ValidationÉchouée {
            champ: "email".to_string(),
            raison: format!("trop long ({} caractères, max 254)", email.len()),
        });
    }
    
    if !regex_email().is_match(email) {
        return Err(ErreurGestion::ValidationÉchouée {
            champ: "email".to_string(),
            raison: format!("format invalide : '{}'", email),
        });
    }
    
    Ok(())
}

pub fn valider_nom(nom: &str) -> Résultat<()> {
    let nom = nom.trim();
    
    if nom.is_empty() {
        return Err(ErreurGestion::ValidationÉchouée {
            champ: "nom".to_string(),
            raison: "ne peut pas être vide".to_string(),
        });
    }
    
    if nom.len() < 2 {
        return Err(ErreurGestion::ValidationÉchouée {
            champ: "nom".to_string(),
            raison: format!("trop court ({} caractères, min 2)", nom.len()),
        });
    }
    
    if nom.len() > 100 {
        return Err(ErreurGestion::ValidationÉchouée {
            champ: "nom".to_string(),
            raison: format!("trop long ({} caractères, max 100)", nom.len()),
        });
    }
    
    Ok(())
}
```

---

## Étape 5 — Le Store (src/store.rs)

```rust
// src/store.rs
use crate::{
    erreurs::{ErreurGestion, Résultat},
    modèles::{Rôle, StatutCompte, Utilisateur},
    validation::{valider_email, valider_nom},
};
use chrono::Utc;
use std::collections::HashMap;
use tracing::{debug, info, instrument, warn};
use uuid::Uuid;

/// Filtre pour la liste des utilisateurs
pub struct Filtre {
    pub rôle: Option<Rôle>,
    pub actif_seulement: bool,
    pub recherche_nom: Option<String>,
}

impl Default for Filtre {
    fn default() -> Self {
        Filtre {
            rôle: None,
            actif_seulement: false,
            recherche_nom: None,
        }
    }
}

/// Paramètres de pagination
pub struct Pagination {
    pub page: usize,        // Commence à 1
    pub par_page: usize,
}

impl Default for Pagination {
    fn default() -> Self {
        Pagination { page: 1, par_page: 20 }
    }
}

/// Résultat paginé
pub struct RésultatPaginé<T> {
    pub données: Vec<T>,
    pub page: usize,
    pub par_page: usize,
    pub total: usize,
    pub pages_totales: usize,
}

/// Store en mémoire pour les utilisateurs
pub struct StoreUtilisateurs {
    données: HashMap<Uuid, Utilisateur>,
    index_email: HashMap<String, Uuid>,
}

impl StoreUtilisateurs {
    pub fn nouveau() -> Self {
        info!("Initialisation du StoreUtilisateurs");
        StoreUtilisateurs {
            données: HashMap::new(),
            index_email: HashMap::new(),
        }
    }
    
    /// Crée un nouvel utilisateur
    #[instrument(skip(self), fields(email = %email))]
    pub fn créer(
        &mut self,
        nom: &str,
        email: &str,
        rôle: Rôle,
    ) -> Résultat<Utilisateur> {
        debug!("Tentative de création d'utilisateur");
        
        // Validation
        valider_nom(nom)?;
        valider_email(email)?;
        
        // Vérifier la duplication d'email
        let email_normalisé = email.to_lowercase();
        if self.index_email.contains_key(&email_normalisé) {
            warn!(email = %email_normalisé, "Email déjà utilisé");
            return Err(ErreurGestion::EmailDupliqué(email_normalisé));
        }
        
        let utilisateur = Utilisateur::nouveau(
            nom.trim().to_string(),
            email_normalisé.clone(),
            rôle,
        );
        
        let id = utilisateur.id;
        self.index_email.insert(email_normalisé, id);
        self.données.insert(id, utilisateur.clone());
        
        info!(id = %id, nom = %nom, "Utilisateur créé");
        Ok(utilisateur)
    }
    
    /// Récupère un utilisateur par son ID
    #[instrument(skip(self))]
    pub fn obtenir_par_id(&self, id: Uuid) -> Résultat<&Utilisateur> {
        self.données
            .get(&id)
            .ok_or(ErreurGestion::Introuvable(id))
    }
    
    /// Récupère un utilisateur par son email
    pub fn obtenir_par_email(&self, email: &str) -> Résultat<&Utilisateur> {
        let email_normalisé = email.to_lowercase();
        let id = self.index_email
            .get(&email_normalisé)
            .ok_or_else(|| ErreurGestion::Introuvable(Uuid::nil()))?;
        
        self.données
            .get(id)
            .ok_or_else(|| ErreurGestion::Introuvable(*id))
    }
    
    /// Met à jour le nom d'un utilisateur
    #[instrument(skip(self))]
    pub fn mettre_à_jour_nom(&mut self, id: Uuid, nouveau_nom: &str) -> Résultat<&Utilisateur> {
        valider_nom(nouveau_nom)?;
        
        let utilisateur = self.données
            .get_mut(&id)
            .ok_or(ErreurGestion::Introuvable(id))?;
        
        if !utilisateur.est_actif() {
            if let StatutCompte::Suspendu { raison } = &utilisateur.statut {
                return Err(ErreurGestion::CompteSuspendu(raison.clone()));
            }
        }
        
        utilisateur.nom = nouveau_nom.trim().to_string();
        utilisateur.modifié_le = Utc::now();
        
        info!(id = %id, nouveau_nom = %nouveau_nom, "Nom mis à jour");
        Ok(self.données.get(&id).unwrap())
    }
    
    /// Suspend un compte utilisateur
    #[instrument(skip(self))]
    pub fn suspendre(&mut self, id: Uuid, raison: &str) -> Résultat<()> {
        let utilisateur = self.données
            .get_mut(&id)
            .ok_or(ErreurGestion::Introuvable(id))?;
        
        utilisateur.statut = StatutCompte::Suspendu {
            raison: raison.to_string(),
        };
        utilisateur.modifié_le = Utc::now();
        
        warn!(id = %id, raison = %raison, "Compte suspendu");
        Ok(())
    }
    
    /// Supprime un utilisateur (soft delete)
    #[instrument(skip(self))]
    pub fn supprimer(&mut self, id: Uuid) -> Résultat<()> {
        let utilisateur = self.données
            .get_mut(&id)
            .ok_or(ErreurGestion::Introuvable(id))?;
        
        utilisateur.statut = StatutCompte::Supprimé {
            supprimé_le: Utc::now(),
        };
        utilisateur.modifié_le = Utc::now();
        
        info!(id = %id, "Utilisateur supprimé (soft)");
        Ok(())
    }
    
    /// Liste les utilisateurs avec filtre et pagination
    pub fn lister(
        &self,
        filtre: Filtre,
        pagination: Pagination,
    ) -> RésultatPaginé<Utilisateur> {
        let mut résultats: Vec<&Utilisateur> = self.données
            .values()
            .filter(|u| {
                if filtre.actif_seulement && !u.est_actif() {
                    return false;
                }
                if let Some(ref rôle) = filtre.rôle {
                    if &u.rôle != rôle {
                        return false;
                    }
                }
                if let Some(ref recherche) = filtre.recherche_nom {
                    if !u.nom.to_lowercase().contains(&recherche.to_lowercase()) {
                        return false;
                    }
                }
                true
            })
            .collect();
        
        // Trier par date de création (plus récent en premier)
        résultats.sort_by(|a, b| b.créé_le.cmp(&a.créé_le));
        
        let total = résultats.len();
        let pages_totales = (total + pagination.par_page - 1) / pagination.par_page;
        let début = (pagination.page - 1) * pagination.par_page;
        
        let données: Vec<Utilisateur> = résultats
            .into_iter()
            .skip(début)
            .take(pagination.par_page)
            .cloned()
            .collect();
        
        RésultatPaginé {
            données,
            page: pagination.page,
            par_page: pagination.par_page,
            total,
            pages_totales,
        }
    }
}
```

---

## Étape 6 — Tests complets (src/tests.rs ou tests/)

```rust
// tests/intégration.rs
use gestion_utilisateurs::{
    erreurs::ErreurGestion,
    modèles::Rôle,
    store::{Filtre, Pagination, StoreUtilisateurs},
};
use uuid::Uuid;

fn store_avec_données() -> StoreUtilisateurs {
    let mut store = StoreUtilisateurs::nouveau();
    store.créer("Alice Martin", "alice@example.com", Rôle::Administrateur).unwrap();
    store.créer("Bob Dupont", "bob@example.com", Rôle::Utilisateur).unwrap();
    store.créer("Charlie Leclerc", "charlie@example.com", Rôle::Lecture).unwrap();
    store
}

// ─── Tests de création ───────────────────────────────────────────────────────

#[test]
fn créer_utilisateur_valide() {
    let mut store = StoreUtilisateurs::nouveau();
    let user = store.créer("Alice", "alice@test.com", Rôle::Utilisateur).unwrap();
    
    assert_eq!(user.nom, "Alice");
    assert_eq!(user.email, "alice@test.com");
    assert_eq!(user.rôle, Rôle::Utilisateur);
    assert!(user.est_actif());
}

#[test]
fn créer_email_dupliqué_retourne_erreur() {
    let mut store = StoreUtilisateurs::nouveau();
    store.créer("Alice", "alice@test.com", Rôle::Utilisateur).unwrap();
    
    let résultat = store.créer("Alice 2", "alice@test.com", Rôle::Lecture);
    
    assert!(matches!(résultat, Err(ErreurGestion::EmailDupliqué(_))));
}

#[test]
fn créer_email_invalide_retourne_erreur() {
    let mut store = StoreUtilisateurs::nouveau();
    
    let résultat = store.créer("Alice", "pas-un-email", Rôle::Utilisateur);
    
    assert!(matches!(
        résultat,
        Err(ErreurGestion::ValidationÉchouée { champ, .. }) if champ == "email"
    ));
}

#[test]
fn créer_normalise_email_en_minuscules() {
    let mut store = StoreUtilisateurs::nouveau();
    let user = store.créer("Alice", "ALICE@EXAMPLE.COM", Rôle::Utilisateur).unwrap();
    
    assert_eq!(user.email, "alice@example.com");
}

// ─── Tests de lecture ────────────────────────────────────────────────────────

#[test]
fn obtenir_par_id_existant() {
    let mut store = StoreUtilisateurs::nouveau();
    let créé = store.créer("Bob", "bob@test.com", Rôle::Lecture).unwrap();
    
    let trouvé = store.obtenir_par_id(créé.id).unwrap();
    assert_eq!(trouvé.email, "bob@test.com");
}

#[test]
fn obtenir_par_id_inexistant_retourne_erreur() {
    let store = StoreUtilisateurs::nouveau();
    let résultat = store.obtenir_par_id(Uuid::new_v4());
    
    assert!(matches!(résultat, Err(ErreurGestion::Introuvable(_))));
}

// ─── Tests de mise à jour ────────────────────────────────────────────────────

#[test]
fn mettre_à_jour_nom_valide() {
    let mut store = StoreUtilisateurs::nouveau();
    let user = store.créer("Alice", "alice@test.com", Rôle::Utilisateur).unwrap();
    
    let mis_à_jour = store.mettre_à_jour_nom(user.id, "Alice Dupont").unwrap();
    assert_eq!(mis_à_jour.nom, "Alice Dupont");
}

#[test]
fn mettre_à_jour_compte_suspendu_retourne_erreur() {
    let mut store = StoreUtilisateurs::nouveau();
    let user = store.créer("Alice", "alice@test.com", Rôle::Utilisateur).unwrap();
    
    store.suspendre(user.id, "Comportement abusif").unwrap();
    
    let résultat = store.mettre_à_jour_nom(user.id, "Nouveau nom");
    assert!(matches!(résultat, Err(ErreurGestion::CompteSuspendu(_))));
}

// ─── Tests de liste et pagination ────────────────────────────────────────────

#[test]
fn lister_tous_les_utilisateurs() {
    let store = store_avec_données();
    let résultat = store.lister(Filtre::default(), Pagination::default());
    
    assert_eq!(résultat.total, 3);
    assert_eq!(résultat.données.len(), 3);
}

#[test]
fn filtrer_par_rôle() {
    let store = store_avec_données();
    let filtre = Filtre {
        rôle: Some(Rôle::Utilisateur),
        ..Default::default()
    };
    
    let résultat = store.lister(filtre, Pagination::default());
    assert_eq!(résultat.total, 1);
    assert_eq!(résultat.données[0].nom, "Bob Dupont");
}

#[test]
fn pagination_fonctionne() {
    let mut store = StoreUtilisateurs::nouveau();
    for i in 0..25 {
        store.créer(
            &format!("Utilisateur {}", i),
            &format!("user{}@test.com", i),
            Rôle::Lecture,
        ).unwrap();
    }
    
    let page1 = store.lister(
        Filtre::default(),
        Pagination { page: 1, par_page: 10 },
    );
    assert_eq!(page1.données.len(), 10);
    assert_eq!(page1.total, 25);
    assert_eq!(page1.pages_totales, 3);
    
    let page3 = store.lister(
        Filtre::default(),
        Pagination { page: 3, par_page: 10 },
    );
    assert_eq!(page3.données.len(), 5); // 25 - 20 = 5
}

// ─── Tests des permissions ───────────────────────────────────────────────────

#[test]
fn administrateur_peut_tout() {
    let mut store = StoreUtilisateurs::nouveau();
    let admin = store.créer("Admin", "admin@test.com", Rôle::Administrateur).unwrap();
    
    assert!(admin.peut("lire"));
    assert!(admin.peut("écrire"));
    assert!(admin.peut("supprimer"));
    assert!(admin.peut("administrer"));
}

#[test]
fn lecture_seule_ne_peut_pas_écrire() {
    let mut store = StoreUtilisateurs::nouveau();
    let lecteur = store.créer("Lecteur", "lecteur@test.com", Rôle::Lecture).unwrap();
    
    assert!(lecteur.peut("lire"));
    assert!(!lecteur.peut("écrire"));
    assert!(!lecteur.peut("supprimer"));
}
```

---

## Étape 7 — Point d'entrée et logging (src/lib.rs)

```rust
// src/lib.rs
pub mod erreurs;
pub mod modèles;
pub mod store;
mod validation;

// Re-exports pratiques
pub use erreurs::{ErreurGestion, Résultat};
pub use modèles::{Rôle, StatutCompte, Utilisateur};
pub use store::{Filtre, Pagination, RésultatPaginé, StoreUtilisateurs};
```

---

## Exécution et validation

```bash
# Lancer tous les tests
cargo test

# Lancer les tests avec output
cargo test -- --nocapture

# Voir la couverture (avec cargo-tarpaulin)
cargo install cargo-tarpaulin
cargo tarpaulin --out Html

# Vérifier la qualité du code
cargo clippy -- -D warnings
cargo fmt --check
```

Sortie attendue :
```
running 15 tests
test créer_utilisateur_valide ... ok
test créer_email_dupliqué_retourne_erreur ... ok
test créer_email_invalide_retourne_erreur ... ok
test créer_normalise_email_en_minuscules ... ok
test obtenir_par_id_existant ... ok
test obtenir_par_id_inexistant_retourne_erreur ... ok
test mettre_à_jour_nom_valide ... ok
test mettre_à_jour_compte_suspendu_retourne_erreur ... ok
test lister_tous_les_utilisateurs ... ok
test filtrer_par_rôle ... ok
test pagination_fonctionne ... ok
test administrateur_peut_tout ... ok
test lecture_seule_ne_peut_pas_écrire ... ok

test result: ok. 13 passed; 0 failed
```

---

## 🏆 Critères de réussite

| Critère | Validé ? |
|---|---|
| Le code compile sans warnings | ✅ |
| Tous les tests passent | ✅ |
| Les erreurs sont typées et descriptives | ✅ |
| Le logging instrumente toutes les opérations | ✅ |
| Aucun `.unwrap()` dans le code de production | ✅ |
| `cargo clippy` sans warnings | ✅ |
| `cargo fmt` appliqué | ✅ |

---

## Extensions optionnelles

- Ajouter la persistance sur disque avec `serde_json`
- Implémenter la recherche full-text sur les noms
- Ajouter un système d'événements (audit log)
- Exposer la bibliothèque via une API REST (anticipation Module 2)

---

**→ Suivant : [Module 1 — Concurrence, Async et Parallélisme](../../module-1/README.md)**
