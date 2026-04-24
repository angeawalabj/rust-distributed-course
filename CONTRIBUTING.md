# 🤝 Contribuer à cette formation

Merci de votre intérêt pour contribuer ! Cette formation est open source et vos contributions sont les bienvenues.

---

## Types de contributions

### 📝 Améliorer le contenu
- Corriger une erreur dans un exemple de code
- Améliorer une explication peu claire
- Ajouter un exemple supplémentaire
- Mettre à jour du code déprécié (nouvelles versions de crates)

### 🐛 Signaler un problème
Ouvrez une issue en précisant :
- Le chapitre concerné (ex: `module-1/chapitre-1.1`)
- La nature du problème (code qui ne compile pas, explication incorrecte, etc.)
- Votre version de Rust (`rustc --version`)

### ➕ Ajouter du contenu
- Nouveau chapitre ou sous-chapitre
- Exercice supplémentaire
- Traduction

---

## Process de contribution

```bash
# 1. Fork le dépôt sur GitHub

# 2. Cloner votre fork
git clone https://github.com/VOTRE_NOM/rust-distributed-course.git

# 3. Créer une branche
git checkout -b fix/module-1-deadlock-example

# 4. Faire vos modifications
# ...

# 5. Vérifier que les exemples de code compilent
# (copier le code dans un projet Rust et tester)

# 6. Commit avec un message clair
git commit -m "fix(module-1): corriger l'exemple de deadlock (ajout d'ordre fixe)"

# 7. Push et ouvrir une Pull Request
git push origin fix/module-1-deadlock-example
```

---

## Conventions

### Format des fichiers Markdown
- Titres hiérarchiques (H1 → H2 → H3)
- Blocs de code avec langage spécifié (` ```rust `)
- Marqueurs standardisés : `> 💡`, `> ⚠️`, `> 🔥`, `> 🔬`, `> 🛠️`
- Lien vers le chapitre suivant en fin de fichier

### Code Rust
- Doit compiler avec `rustc 1.75+`
- Respecter `cargo fmt` et `cargo clippy`
- Inclure les imports nécessaires
- Commenter les parties non-triviales

### Messages de commit
```
type(scope): description courte

Types : feat, fix, docs, refactor, test
Scope : module-0, module-1, ..., module-6, infra
```

---

## Code of Conduct

Soyez respectueux, constructif et bienveillant. Toute contribution est la bienvenue, quel que soit le niveau d'expérience.

---

*Merci de rendre cette formation meilleure pour tous les apprenants !*
