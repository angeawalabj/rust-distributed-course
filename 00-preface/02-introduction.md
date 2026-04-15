# 📝 Introduction

---

## Le monde réel des systèmes distribués

Imaginez la scène : il est 14h37 un mardi. Votre service de paiement commence à répondre avec des erreurs 503. Les premières alertes tombent. En deux minutes, votre équipe est en guerre room. Vous avez des logs — mais ils sont illisibles, mélange de `println!` et de messages ad hoc sans contexte. Vous avez des métriques — mais elles ne couvrent pas la bonne couche. Vous avez un système distribué — mais vous ne savez pas lequel des cinq nœuds est à l'origine du problème.

Ce scénario n'est pas fictif. Il se produit chaque jour dans des centaines d'équipes d'ingénierie, y compris dans des entreprises qui ont investi massivement dans leur infrastructure.

La différence entre une équipe qui résout ce genre d'incident en vingt minutes et une équipe qui y passe trois heures, c'est rarement une question de talent brut. C'est une question de **fondations** : est-ce que le système a été conçu pour être compris ? Est-ce que le code est écrit pour être débogué ? Est-ce que l'architecture tolère les pannes, ou s'effondre-t-elle à la première partition réseau ?

Cette formation vous donne ces fondations.

---

## Structure du cursus

Le cursus est organisé en sept modules progressifs. Chacun construit sur le précédent, et chacun se termine par un travail pratique qui synthétise les concepts abordés.

### 🟢 Module 0 — Fondations Rust

Le socle de tout le reste. Ownership, borrowing, lifetimes, gestion des erreurs, organisation de projet. Si vous connaissez déjà Rust, ce module vous permettra de solidifier votre compréhension du modèle mémoire et de vous aligner sur les conventions utilisées dans toute la formation.

**Ce que vous construirez :** une bibliothèque Rust modulaire avec gestion d'erreurs personnalisée, logging structuré et tests complets.

### 🟡 Module 1 — Concurrence et asynchronisme

Comment gérer des millions de connexions simultanées sans threads. Le modèle async/await de Rust, le runtime Tokio, les channels de communication, les patterns d'actor model. Et aussi : comment provoquer un deadlock, le détecter, et l'éliminer.

**Ce que vous construirez :** un serveur WebSocket multi-utilisateurs avec système d'acteurs.

### 🟠 Module 2 — Réseau et protocoles

TCP/IP, HTTP, REST, gRPC, WebSockets. Comment construire une API robuste qui gère les timeouts, les retries, et les erreurs réseau sans tomber en production. Authentification JWT, TLS avec rustls.

**Ce que vous construirez :** une API REST sécurisée avec authentification et injection d'erreurs réseau.

### 🔵 Module 3 — Stockage et performance

PostgreSQL avec SQLx, moteurs clé-valeur, indexation inversée, sharding basique. Comment concevoir un moteur de stockage qui reste performant sous charge et cohérent face aux pannes.

**Ce que vous construirez :** un moteur clé-valeur persistant avec API réseau et benchmarks.

### 🟣 Module 4 — Systèmes distribués et consensus

Le théorème CAP, la réplication, le consensus avec l'algorithme Raft, les transactions distribuées avec le pattern Saga. Comment concevoir un cluster qui survit à la perte d'un nœud sans perdre de données.

**Ce que vous construirez :** un cluster distribué multi-nœuds avec réplication et simulation de pannes.

### 🔴 Module 5 — Observabilité et résilience

Logs structurés avec `tracing`, métriques Prometheus, circuit breakers, bulkheads, chaos engineering. Comment transformer un système fragile en un système qui se dégrade gracieusement.

**Ce que vous construirez :** un microservice entièrement instrumenté avec circuit breaker et injection d'erreurs.

### ⚫ Module 6 — Extensions modernes et projet final

WebAssembly côté serveur avec Wasmtime, blockchain avec Substrate, synchronisation d'état temps réel. Et surtout : le projet final, qui assemble tout ce que vous avez appris en un système distribué complet déployé sur Kubernetes.

**Ce que vous construirez :** un backend distribué complet — stockage répliqué, API sécurisée, observabilité intégrée, plugins WASM, déploiement cloud natif.

---

## La méthodologie "Failure First"

La plupart des ingénieurs apprennent les systèmes distribués en lisant de la théorie, puis en essayant de l'appliquer. Le problème : les systèmes distribués tombent en panne de manières que la théorie ne prédit pas toujours clairement.

Cette formation inverse l'approche. Avant de vous montrer comment éviter un problème, nous vous montrons comment le provoquer délibérément dans un environnement contrôlé. Puis nous utilisons les outils de diagnostic pour comprendre exactement ce qui s'est passé. Puis nous corrigeons.

Cette approche a plusieurs avantages :

**Vous développez une intuition de débogage.** Quand vous avez déjà vu à quoi ressemble un deadlock dans les traces d'exécution, vous le reconnaissez immédiatement en production.

**Vous apprenez à faire confiance aux outils.** Les logs structurés, les métriques et le tracing ne sont pas des luxes — ce sont des instruments de navigation sans lesquels vous volez à l'aveugle.

**Vous cessez d'avoir peur des pannes.** Un système bien conçu ne craint pas les pannes — il les anticipe, les détecte, et s'en remet. Vous apprendrez à penser en termes de modes de défaillance dès la conception.

---

## Environnement de travail recommandé

Pour suivre cette formation dans de bonnes conditions :

**Rust toolchain :**
```bash
# Installer rustup (gestionnaire de versions Rust)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Installer la version stable
rustup update stable

# Outils utiles
cargo install cargo-watch cargo-edit cargo-expand
```

**Outils de développement :**
```bash
# Docker (pour PostgreSQL, Redis, etc.)
# VS Code avec l'extension rust-analyzer
# ou CLion / Zed avec support Rust natif
```

**Vérification de l'installation :**
```bash
rustc --version  # rustc 1.75.0 ou supérieur
cargo --version  # cargo 1.75.0 ou supérieur
```

---

## Pour les développeurs qui viennent d'autres langages

**Depuis Go :** Rust et Go partagent une philosophie de systèmes sans garbage collector traditionnel, mais leurs modèles sont très différents. Go délègue la gestion mémoire à un GC. Rust la résout à la compilation. Vous trouverez Rust plus verbeux au début, mais vous gagnerez en contrôle et en performance.

**Depuis C/C++ :** Vous reconnaîtrez de nombreux concepts — stack vs heap, pointeurs, gestion manuelle de la mémoire. La différence : Rust garantit statiquement qu'un programme correct ne peut pas avoir de dangling pointers, use-after-free ou data races. Ce que vous faisiez avec discipline et conventions, Rust le fait avec des règles du compilateur.

**Depuis Python/JavaScript :** Le saut est plus important, mais la formation est conçue pour vous y préparer. Les concepts de Module 0 sont expliqués en partant de zéro, avec des analogies claires. Accrochez-vous sur les lifetimes — une fois ce cap franchi, tout devient plus naturel.

---

## Une dernière chose

Cette formation est difficile. Elle demande du temps, de la concentration, et une certaine tolérance à la frustration.

Elle en vaut la peine.

Les ingénieurs qui maîtrisent les systèmes distribués en Rust sont parmi les plus recherchés de l'industrie. Mais au-delà du marché, il y a quelque chose de plus profond : la satisfaction de comprendre vraiment ce qui se passe dans votre système, de pouvoir diagnostiquer n'importe quel problème avec méthode, de déployer du code avec confiance.

C'est l'objectif de cette formation.

Bonne lecture — et bon débogage.

---

**→ Suivant : [Module 0 — Fondations Rust](../module-0/README.md)**
