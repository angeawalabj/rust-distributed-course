# 🛠️ TP Module 1 — Serveur WebSocket Multi-Utilisateurs avec Actor System

> **Durée estimée :** 4 à 6 heures
> **Compétence validée :** Async, concurrence, actor model, debugging de deadlocks

---

## Objectif

Construire un serveur de chat temps réel complet : WebSocket, gestion de salons, broadcast de messages, système d'acteurs, et injection volontaire d'un deadlock à corriger.

---

## Spécifications

1. Connexions WebSocket multi-clients simultanées
2. Salons de discussion (rooms) avec rejoindre/quitter
3. Broadcast de messages à tous les membres d'un salon
4. Messages privés entre utilisateurs
5. Liste des utilisateurs connectés
6. Acteur de supervision qui redémarre les workers crashés
7. **Exercice Failure First :** provoquer et corriger un deadlock

---

## Étape 1 — Dépendances

```toml
# Cargo.toml
[package]
name = "chat-serveur"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-tungstenite = "0.21"
futures-util = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
thiserror = "1"
```

---

## Étape 2 — Types et messages (src/types.rs)

```rust
// src/types.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;

pub type ClientId = Uuid;
pub type SalonId = String;

/// Messages entrants du client WebSocket
#[derive(Debug, Deserialize)]
#[serde(tag = "type", content = "data")]
pub enum MessageEntrant {
    #[serde(rename = "rejoindre_salon")]
    RejoindreS { salon: SalonId },
    
    #[serde(rename = "quitter_salon")]
    QuitterSalon { salon: SalonId },
    
    #[serde(rename = "message_salon")]
    MessageSalon { salon: SalonId, contenu: String },
    
    #[serde(rename = "message_privé")]
    MessagePrivé { destinataire: ClientId, contenu: String },
    
    #[serde(rename = "lister_utilisateurs")]
    ListerUtilisateurs,
    
    #[serde(rename = "ping")]
    Ping,
}

/// Messages sortants vers le client
#[derive(Debug, Serialize)]
#[serde(tag = "type", content = "data")]
pub enum MessageSortant {
    #[serde(rename = "message_salon")]
    MessageSalon {
        salon: SalonId,
        auteur: ClientId,
        contenu: String,
        horodatage: u64,
    },
    
    #[serde(rename = "message_privé")]
    MessagePrivé {
        expéditeur: ClientId,
        contenu: String,
    },
    
    #[serde(rename = "notification")]
    Notification { contenu: String },
    
    #[serde(rename = "liste_utilisateurs")]
    ListeUtilisateurs { utilisateurs: Vec<InfoUtilisateur> },
    
    #[serde(rename = "erreur")]
    Erreur { code: String, message: String },
    
    #[serde(rename = "pong")]
    Pong,
}

#[derive(Debug, Serialize, Clone)]
pub struct InfoUtilisateur {
    pub id: ClientId,
    pub salons: Vec<SalonId>,
}
```

---

## Étape 3 — Acteur de registre central (src/registre.rs)

```rust
// src/registre.rs
use crate::types::*;
use std::collections::{HashMap, HashSet};
use tokio::sync::{mpsc, oneshot};
use tracing::{info, warn};

/// Messages vers l'acteur registre
pub enum MessageRegistre {
    ConnexionClient {
        id: ClientId,
        tx: mpsc::Sender<MessageSortant>,
    },
    DéconnexionClient { id: ClientId },
    RejoindreS { client_id: ClientId, salon: SalonId },
    QuitterSalon { client_id: ClientId, salon: SalonId },
    EnvoyerSalon {
        salon: SalonId,
        auteur: ClientId,
        contenu: String,
    },
    EnvoyerPrivé {
        expéditeur: ClientId,
        destinataire: ClientId,
        contenu: String,
    },
    ListerUtilisateurs {
        réponse: oneshot::Sender<Vec<InfoUtilisateur>>,
    },
}

struct ÉtatRegistre {
    clients: HashMap<ClientId, mpsc::Sender<MessageSortant>>,
    salons: HashMap<SalonId, HashSet<ClientId>>,
    salons_par_client: HashMap<ClientId, HashSet<SalonId>>,
}

impl ÉtatRegistre {
    fn nouveau() -> Self {
        ÉtatRegistre {
            clients: HashMap::new(),
            salons: HashMap::new(),
            salons_par_client: HashMap::new(),
        }
    }
    
    fn connecter(&mut self, id: ClientId, tx: mpsc::Sender<MessageSortant>) {
        self.clients.insert(id, tx);
        self.salons_par_client.insert(id, HashSet::new());
        info!(client_id = %id, "Client connecté ({} total)", self.clients.len());
    }
    
    fn déconnecter(&mut self, id: ClientId) {
        // Retirer de tous les salons
        if let Some(salons) = self.salons_par_client.remove(&id) {
            for salon in salons {
                if let Some(membres) = self.salons.get_mut(&salon) {
                    membres.remove(&id);
                    // Notification aux autres membres
                    let notif = MessageSortant::Notification {
                        contenu: format!("Utilisateur {} a quitté", id),
                    };
                    self.diffuser_salon(&salon, &id, notif);
                }
            }
        }
        self.clients.remove(&id);
        info!(client_id = %id, "Client déconnecté ({} restants)", self.clients.len());
    }
    
    fn rejoindre_salon(&mut self, client_id: ClientId, salon: SalonId) {
        self.salons
            .entry(salon.clone())
            .or_default()
            .insert(client_id);
        
        self.salons_par_client
            .entry(client_id)
            .or_default()
            .insert(salon.clone());
        
        let notif = MessageSortant::Notification {
            contenu: format!("Bienvenue dans #{}", salon),
        };
        if let Some(tx) = self.clients.get(&client_id) {
            let _ = tx.try_send(notif);
        }
        
        info!(client_id = %client_id, salon = %salon, "Client a rejoint le salon");
    }
    
    fn diffuser_salon(
        &self,
        salon: &SalonId,
        exclu: &ClientId,
        message: MessageSortant,
    ) {
        let membres = match self.salons.get(salon) {
            Some(m) => m,
            None => return,
        };
        
        for membre_id in membres {
            if membre_id == exclu { continue; }
            if let Some(tx) = self.clients.get(membre_id) {
                // clone le message pour chaque destinataire
                // Dans un vrai système, utiliser Arc<MessageSortant>
                let msg_clone = serde_json::to_string(&message)
                    .map(|s| MessageSortant::Notification { contenu: s });
                if let Ok(msg) = msg_clone {
                    let _ = tx.try_send(msg);
                }
            }
        }
    }
    
    fn lister_utilisateurs(&self) -> Vec<InfoUtilisateur> {
        self.clients.keys().map(|&id| {
            let salons = self.salons_par_client
                .get(&id)
                .map(|s| s.iter().cloned().collect())
                .unwrap_or_default();
            InfoUtilisateur { id, salons }
        }).collect()
    }
}

pub struct ActeurRegistre;

impl ActeurRegistre {
    pub fn démarrer() -> mpsc::Sender<MessageRegistre> {
        let (tx, mut rx) = mpsc::channel::<MessageRegistre>(512);
        
        tokio::spawn(async move {
            let mut état = ÉtatRegistre::nouveau();
            
            while let Some(msg) = rx.recv().await {
                match msg {
                    MessageRegistre::ConnexionClient { id, tx } => {
                        état.connecter(id, tx);
                    }
                    MessageRegistre::DéconnexionClient { id } => {
                        état.déconnecter(id);
                    }
                    MessageRegistre::RejoindreS { client_id, salon } => {
                        état.rejoindre_salon(client_id, salon);
                    }
                    MessageRegistre::QuitterSalon { client_id, salon } => {
                        if let Some(salons) = état.salons_par_client.get_mut(&client_id) {
                            salons.remove(&salon);
                        }
                        if let Some(membres) = état.salons.get_mut(&salon) {
                            membres.remove(&client_id);
                        }
                    }
                    MessageRegistre::EnvoyerSalon { salon, auteur, contenu } => {
                        use std::time::{SystemTime, UNIX_EPOCH};
                        let horodatage = SystemTime::now()
                            .duration_since(UNIX_EPOCH).unwrap()
                            .as_secs();
                        
                        let message = MessageSortant::MessageSalon {
                            salon: salon.clone(),
                            auteur,
                            contenu,
                            horodatage,
                        };
                        état.diffuser_salon(&salon, &auteur, message);
                    }
                    MessageRegistre::EnvoyerPrivé { expéditeur, destinataire, contenu } => {
                        let message = MessageSortant::MessagePrivé { expéditeur, contenu };
                        if let Some(tx) = état.clients.get(&destinataire) {
                            let _ = tx.try_send(message);
                        } else {
                            warn!(destinataire = %destinataire, "Destinataire introuvable");
                        }
                    }
                    MessageRegistre::ListerUtilisateurs { réponse } => {
                        let _ = réponse.send(état.lister_utilisateurs());
                    }
                }
            }
        });
        
        tx
    }
}
```

---

## Étape 4 — Handler de connexion WebSocket (src/connexion.rs)

```rust
// src/connexion.rs
use crate::types::*;
use crate::registre::MessageRegistre;
use futures_util::{SinkExt, StreamExt};
use tokio::sync::mpsc;
use tokio_tungstenite::{accept_async, tungstenite::Message};
use tracing::{error, info};

pub async fn gérer_connexion(
    stream: tokio::net::TcpStream,
    registre_tx: mpsc::Sender<MessageRegistre>,
) {
    let client_id = uuid::Uuid::new_v4();
    
    // Handshake WebSocket
    let ws_stream = match accept_async(stream).await {
        Ok(ws) => ws,
        Err(e) => {
            error!("Échec du handshake WebSocket : {}", e);
            return;
        }
    };
    
    info!(client_id = %client_id, "Nouvelle connexion WebSocket");
    
    let (mut ws_sender, mut ws_receiver) = ws_stream.split();
    
    // Channel pour envoyer des messages à ce client
    let (client_tx, mut client_rx) = mpsc::channel::<MessageSortant>(64);
    
    // Enregistrer le client
    registre_tx.send(MessageRegistre::ConnexionClient {
        id: client_id,
        tx: client_tx,
    }).await.unwrap();
    
    // Tâche d'envoi sortant (registre → WebSocket)
    let envoi = tokio::spawn(async move {
        while let Some(msg) = client_rx.recv().await {
            let texte = serde_json::to_string(&msg).unwrap();
            if ws_sender.send(Message::Text(texte)).await.is_err() {
                break; // Connexion fermée
            }
        }
    });
    
    // Boucle de réception (WebSocket → registre)
    while let Some(résultat) = ws_receiver.next().await {
        match résultat {
            Ok(Message::Text(texte)) => {
                match serde_json::from_str::<MessageEntrant>(&texte) {
                    Ok(msg) => {
                        traiter_message(client_id, msg, &registre_tx).await;
                    }
                    Err(e) => {
                        error!(client_id = %client_id, "Message malformé : {}", e);
                    }
                }
            }
            Ok(Message::Close(_)) | Err(_) => break,
            _ => {} // Ping/Pong gérés automatiquement
        }
    }
    
    // Déconnexion propre
    envoi.abort();
    registre_tx.send(MessageRegistre::DéconnexionClient {
        id: client_id,
    }).await.unwrap();
    
    info!(client_id = %client_id, "Client déconnecté");
}

async fn traiter_message(
    client_id: ClientId,
    msg: MessageEntrant,
    registre_tx: &mpsc::Sender<MessageRegistre>,
) {
    match msg {
        MessageEntrant::RejoindreS { salon } => {
            let _ = registre_tx.send(MessageRegistre::RejoindreS {
                client_id, salon,
            }).await;
        }
        MessageEntrant::MessageSalon { salon, contenu } => {
            let _ = registre_tx.send(MessageRegistre::EnvoyerSalon {
                salon, auteur: client_id, contenu,
            }).await;
        }
        MessageEntrant::ListerUtilisateurs => {
            let (tx, rx) = tokio::sync::oneshot::channel();
            let _ = registre_tx.send(MessageRegistre::ListerUtilisateurs {
                réponse: tx,
            }).await;
            // La réponse sera envoyée directement via le channel du client
        }
        MessageEntrant::Ping => {
            // Géré automatiquement par tokio-tungstenite
        }
        _ => {}
    }
}
```

---

## Étape 5 — Main (src/main.rs)

```rust
// src/main.rs
mod connexion;
mod registre;
mod types;

use connexion::gérer_connexion;
use registre::ActeurRegistre;
use tokio::net::TcpListener;
use tracing::info;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter("chat_serveur=debug,info")
        .init();
    
    let adresse = "127.0.0.1:8765";
    let listener = TcpListener::bind(adresse).await?;
    
    info!("Serveur démarré sur ws://{}", adresse);
    
    let registre_tx = ActeurRegistre::démarrer();
    
    loop {
        let (stream, addr) = listener.accept().await?;
        info!("Nouvelle connexion TCP depuis {}", addr);
        
        let reg = registre_tx.clone();
        tokio::spawn(async move {
            gérer_connexion(stream, reg).await;
        });
    }
}
```

---

## Étape 6 — 🔥 Exercice Failure First : Injecter un deadlock

Ajoutez ce code dans le registre et observez le comportement :

```rust
// ⚠️ Code à injecter volontairement pour l'exercice
use tokio::sync::Mutex;
use std::sync::Arc;

// Deux mutex partagés entre des handlers
let mutex_sessions = Arc::new(Mutex::new(HashMap::<ClientId, String>::new()));
let mutex_logs = Arc::new(Mutex::new(Vec::<String>::new()));

// Handler A : prend sessions puis logs
let m_s = Arc::clone(&mutex_sessions);
let m_l = Arc::clone(&mutex_logs);
tokio::spawn(async move {
    let _s = m_s.lock().await;
    tokio::time::sleep(Duration::from_millis(10)).await;
    let _l = m_l.lock().await; // DEADLOCK si B a déjà pris logs
});

// Handler B : prend logs puis sessions (ordre inversé !)
let m_s2 = Arc::clone(&mutex_sessions);
let m_l2 = Arc::clone(&mutex_logs);
tokio::spawn(async move {
    let _l = m_l2.lock().await;
    tokio::time::sleep(Duration::from_millis(10)).await;
    let _s = m_s2.lock().await; // DEADLOCK
});
```

**Observer :** le serveur se fige sans message d'erreur.

**Diagnostiquer :** avec `tokio-console`, observer les tâches bloquées sur `lock().await`.

**Corriger :** toujours acquérir les verrous dans le même ordre.

---

## Test avec un client WebSocket

```bash
# Installer wscat
npm install -g wscat

# Se connecter
wscat -c ws://localhost:8765

# Envoyer des messages
{"type":"rejoindre_salon","data":{"salon":"général"}}
{"type":"message_salon","data":{"salon":"général","contenu":"Bonjour tout le monde !"}}
{"type":"lister_utilisateurs"}
```

---

## 🏆 Critères de réussite

| Critère | Validé ? |
|---|---|
| Le serveur accepte plusieurs connexions simultanées | ✅ |
| Les messages sont diffusés à tous les membres du salon | ✅ |
| La déconnexion est propre (pas de ressources orphelines) | ✅ |
| Le deadlock est reproduit et corrigé | ✅ |
| `tracing` instrumente toutes les opérations | ✅ |
| Pas de panics en conditions normales | ✅ |

---

**→ Suivant : [Module 2 — Réseau et Protocoles](../../module-2/README.md)**
