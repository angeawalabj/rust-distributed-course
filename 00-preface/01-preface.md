# 📖 Préface

> *"Le compilateur Rust n'est pas votre ennemi. C'est le meilleur collègue que vous n'avez jamais eu — celui qui refuse de laisser passer une erreur en production."*

---

## Pourquoi cette formation existe

Quand on parle de systèmes distribués aujourd'hui, on parle de services qui doivent fonctionner sans interruption, encaisser des pics de charge imprévus, se remettre d'une panne réseau en quelques millisecondes, et traiter des millions de requêtes simultanément. C'est le quotidien des équipes d'ingénierie chez les grandes plateformes — et c'est de plus en plus la réalité des startups, des fintechs et des infrastructures critiques dans tous les secteurs.

Pourtant, la majorité des formations disponibles abordent ces sujets de manière fragmentée : un cours sur Rust ici, un tutoriel sur les microservices là, un article sur Raft ailleurs. Vous vous retrouvez avec des connaissances en îlots, incapables de les assembler en une architecture cohérente et déployable.

Cette formation a été conçue pour combler ce vide. Elle ne se contente pas de vous enseigner Rust ou les systèmes distribués séparément — elle vous apprend à les penser ensemble, à concevoir des solutions où le langage et l'architecture se renforcent mutuellement.

---

## La philosophie derrière chaque chapitre

### "Failure First" — Échouer intelligemment

La plupart des formations vous montrent le code qui fonctionne. Ici, nous commençons souvent par le code qui casse.

Pourquoi ? Parce qu'un développeur qui n'a jamais provoqué un deadlock de sa propre main ne comprend pas vraiment ce qu'est un deadlock. Il connaît la définition. Mais face à un symptôme bizarre en production à 3h du matin, cette définition ne lui servira pas. Ce qui lui servira, c'est la mémoire musculaire d'avoir déjà diagnostiqué ce problème, d'avoir vu les outils pointer vers la cause, d'avoir su quoi chercher.

Chaque module introduit donc volontairement des erreurs : fuites mémoire, panics, partitions réseau simulées, corruptions de données. Vous apprendrez à les détecter avec les bons outils, à les comprendre dans leur profondeur, et à les corriger durablement.

### L'observabilité comme discipline, pas comme option

Un service sans logs structurés, sans métriques et sans tracing est un service que vous ne comprenez pas. Vous croyez le comprendre tant qu'il fonctionne — mais dès qu'il flanche, vous naviguez à l'aveugle.

Dans cette formation, l'observabilité n'est pas un chapitre annexe. C'est un fil rouge qui traverse tous les modules. Chaque TP vous demandera d'instrumenter votre code, d'exposer des métriques Prometheus, de produire des logs structurés exploitables. Déployer sans observer, c'est conduire sans tableau de bord.

### WebAssembly comme moteur d'extensibilité

WebAssembly n'est plus réservé aux navigateurs. Côté serveur, il représente une révolution architecturale : des plugins isolés, sécurisés, chargés à chaud, sans redémarrer le processus hôte. Cette formation intègre WASM comme un pilier réel de l'extensibilité backend, pas comme une curiosité technologique.

---

## Comment utiliser cette formation

Chaque chapitre est autonome et peut être lu indépendamment, mais ils ont été conçus pour être suivis dans l'ordre. Les concepts s'accumulent : ce que vous apprenez en Module 0 sera utilisé et approfondi en Module 3.

Les travaux pratiques (TP) en fin de chaque module sont la partie la plus importante. Ne les sautez pas. Un concept lu mais non pratiqué s'évapore en quelques jours. Un concept implémenté, débogué et instrumenté reste.

### Conventions de lecture

Tout au long de cette formation, vous rencontrerez ces marqueurs :

> 💡 **Concept clé** — Une notion fondamentale à retenir absolument.

> ⚠️ **Piège courant** — Une erreur que font la plupart des développeurs à ce stade.

> 🔥 **Failure First** — Un exercice où vous allez volontairement casser quelque chose.

> 🔬 **Sous le capot** — Une explication du mécanisme interne, pour les curieux.

> 🛠️ **En production** — Ce que font les équipes d'ingénierie dans le monde réel.

---

## Un mot sur Rust

Rust est un langage exigeant. Son compilateur refuse des programmes que d'autres langages accepteraient sans broncher — et cette rigueur est précisément ce qui le rend extraordinaire pour les systèmes critiques.

Mais cette rigueur a un coût d'apprentissage réel. Il y aura des moments où le borrow checker vous semblera arbitraire, où vous vous demanderez pourquoi Python ou Go ne posent pas autant de problèmes. Ces moments font partie du processus.

Ce que vous êtes en train d'apprendre, c'est à "danser avec le compilateur" : à écrire du code que le compilateur peut vérifier statiquement, à organiser vos données et vos références de manière à ce que les règles de Rust deviennent naturelles plutôt que contraignantes. Une fois cette danse maîtrisée, vous aurez un niveau de confiance dans votre code — en particulier dans du code concurrent — que vous n'aviez jamais eu avec un autre langage.

Patience, persévérance, et bonne formation.

---

*— L'auteur*

---

**→ Suivant : [Introduction](./02-introduction.md)**
