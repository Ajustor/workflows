# Workflows — Reusable GitHub Actions pour projets Rust

Ce dépôt contient des **workflows GitHub Actions réutilisables** conçus pour les projets Rust de l'organisation **Ajustor**. Ils automatisent la détection de changement de version, le build multi-plateforme et la création de releases GitHub.

---

## Workflows disponibles

| Workflow | Fichier | Rôle |
|---|---|---|
| **Version Detect** | `.github/workflows/cargo-version-detect.yml` | Détecte un changement de version dans `Cargo.toml` |
| **Build & Release** | `.github/workflows/cargo-release.yml` | Build multi-plateforme + publication GitHub Release |
| **Mono-Repo Release** | `.github/workflows/cargo-monorepo-release.yml` | Build multi-plateforme pour Cargo workspaces |

---

## Fonctionnement général

```
Push sur main
    │
    ▼
┌──────────────────────────┐
│  cargo-version-detect    │
│  Compare la version du   │
│  Cargo.toml entre HEAD   │
│  et HEAD~1               │
└──────────┬───────────────┘
           │
     version changée ?
       │          │
      oui        non
       │          │
       ▼          ▼
┌──────────────┐  (rien)
│ cargo-release│
│ ─ check      │
│ ─ test       │
│ ─ build      │
│ ─ publish    │
│ ─ release    │
└──────────────┘
```

### 1. Détection de version (`cargo-version-detect`)

À chaque push, ce workflow :

1. Clone le dépôt avec `fetch-depth: 2` (commit courant + parent).
2. Extrait la version depuis `Cargo.toml` du commit courant.
3. Extrait la version depuis `Cargo.toml` du commit parent (`HEAD~1`).
4. Compare les deux versions.
5. Expose les **outputs** suivants :

| Output | Type | Description |
|---|---|---|
| `version_changed` | `"true"` / `"false"` | Indique si la version a changé |
| `new_version` | `string` | Version actuelle (ex: `1.2.0`) |
| `previous_version` | `string` | Version précédente (vide si premier commit) |

**Cas particuliers gérés :**
- **Premier commit** (pas de parent) → considéré comme un changement de version.
- **Cargo.toml absent dans le commit parent** (nouveau projet) → considéré comme un changement.
- **Chemin personnalisé** → paramètre `cargo_toml_path` pour les workspaces.

### 2. Build & Release (`cargo-release`)

Si la version a changé, ce workflow enchaîne :

| Job | Description |
|---|---|
| **prepare** | Détermine le nom du package, la version et la matrice de build |
| **check** | `cargo fmt --check` |
| **test** | `cargo test --all-features` |
| **build** | Compilation en release pour chaque cible (Windows, Linux, macOS) |
| **publish** | Publication sur crates.io (si le secret `CARGO_REGISTRY_TOKEN` est fourni) |
| **release** | Création d'une GitHub Release avec tag `vX.Y.Z` et upload des binaires |
| **pages** | Déploiement de la documentation sur GitHub Pages (si `deploy_pages: true`) |

**Cibles supportées :**

| Clé | Target Rust | OS Runner |
|---|---|---|
| `windows` | `x86_64-pc-windows-msvc` | `windows-latest` |
| `linux` | `x86_64-unknown-linux-gnu` | `ubuntu-latest` |
| `macos` | `aarch64-apple-darwin` | `macos-latest` |

---

## Mise en place dans un projet

### Étape 1 — Créer le workflow appelant

Dans votre dépôt Rust, créez `.github/workflows/ci.yml` :

```yaml
name: CI & Release

on:
  push:
    branches: [main]
    tags:
      - 'v*'  # Déclenche aussi sur les tags (ex: v1.0.0)
  pull_request:
    branches: [main]

jobs:
  # ── Détection de changement de version ──────────────────────
  version-check:
    uses: Ajustor/workflows/.github/workflows/cargo-version-detect.yml@master

  # ── Release (uniquement si la version a changé OU si c'est un tag) ─────────────
  release:
    needs: version-check
    if: needs.version-check.outputs.version_changed == 'true'
    uses: Ajustor/workflows/.github/workflows/cargo-release.yml@master
    with:
      targets: '["windows", "linux", "macos"]'
      version: ${{ needs.version-check.outputs.new_version }}
    secrets:
      CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

### Étape 2 — Configurer les secrets (optionnel)

Si vous souhaitez publier sur **crates.io**, ajoutez le secret `CARGO_REGISTRY_TOKEN` dans les paramètres du dépôt :

> **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

### Étape 3 — Bumper la version pour déclencher une release

Pour déclencher une release, il suffit de modifier la version dans `Cargo.toml` et de push sur `main` :

```toml
[package]
name = "mon-projet"
version = "1.1.0"   # ← changer cette ligne
```

```bash
git add Cargo.toml
git commit -m "chore: bump version to 1.1.0"
git push origin main
```

Le workflow détectera automatiquement le changement et lancera le pipeline de release.

### Alternative — Déclencher une release via un tag

Vous pouvez aussi déclencher une release en créant un tag, sans modifier la version dans `Cargo.toml` :

```bash
git tag v1.1.0
git push origin v1.1.0
```

Dans ce cas, le workflow `cargo-version-detect` retournera automatiquement `version_changed=true` et la release sera créée.

---

## Paramètres des workflows

### `cargo-version-detect`

| Input | Requis | Défaut | Description |
|---|---|---|---|
| `cargo_toml_path` | Non | `Cargo.toml` | Chemin vers le fichier Cargo.toml |

| Output | Type | Description |
|---|---|---|
| `version_changed` | `"true"` / `"false"` | Indique si la version a changé (toujours `true` pour un tag) |
| `new_version` | `string` | Version actuelle (ex: `1.2.0`) |
| `previous_version` | `string` | Version précédente (vide si premier commit ou tag) |
| `is_tag` | `"true"` / `"false"` | Indique si le workflow a été déclenché par un tag |

### `cargo-release`

| Input | Requis | Défaut | Description |
|---|---|---|---|
| `targets` | **Oui** | — | JSON array des cibles : `"windows"`, `"linux"`, `"macos"` |
| `package_name` | Non | Extrait de `Cargo.toml` | Nom du binaire |
| `publish_to_crate` | Non | `false` | Forcer la publication sur crates.io |
| `version` | Non | Extrait de `Cargo.toml` | Version pour le tag de release (ex: `1.2.3`) |
| `deploy_pages` | Non | `false` | Déployer la documentation sur GitHub Pages |

| Secret | Requis | Description |
|---|---|---|
| `CARGO_REGISTRY_TOKEN` | Non | Token crates.io pour la publication |

### `cargo-monorepo-release`

Ce workflow est conçu pour les **Cargo workspaces** (mono-repos) où plusieurs crates coexistent.

| Input | Requis | Défaut | Description |
|---|---|---|---|
| `targets` | **Oui** | — | JSON array des cibles : `"windows"`, `"linux"`, `"macos"`, `"macos-x86"`, `"linux-arm"` |
| `package_name` | **Oui** | — | Nom du crate à builder (doit correspondre au nom dans `Cargo.toml`) |
| `package_path` | **Oui** | — | Chemin vers le package (ex: `crates/my-cli`) |
| `binary_name` | Non | `package_name` | Nom du binaire si différent du nom du package |
| `workspace_root` | Non | `.` | Racine du workspace Cargo |
| `publish_to_crate` | Non | `false` | Forcer la publication sur crates.io |
| `version` | Non | Extrait de `Cargo.toml` | Version pour le tag de release |
| `run_tests` | Non | `true` | Exécuter les tests avant le build |
| `run_clippy` | Non | `false` | Exécuter clippy |
| `deploy_pages` | Non | `false` | Déployer la documentation sur GitHub Pages |

| Secret | Requis | Description |
|---|---|---|
| `CARGO_REGISTRY_TOKEN` | Non | Token crates.io pour la publication |

**Note :** Les tags de release utilisent le format `package_name-vX.Y.Z` pour distinguer les différents packages du mono-repo.

---

## GitHub Pages

Les workflows `cargo-release` et `cargo-monorepo-release` peuvent déployer automatiquement la documentation Rust sur GitHub Pages.

### Prérequis

1. **Activer GitHub Pages** dans les paramètres du dépôt :
   > **Settings** → **Pages** → **Source** → **GitHub Actions**

2. **Ajouter les permissions** dans le workflow appelant :

```yaml
name: Release

on:
  push:
    branches: [main]

# ⚠️ Permissions requises pour GitHub Pages
permissions:
  contents: write
  pages: write
  id-token: write

jobs:
  release:
    uses: Ajustor/workflows/.github/workflows/cargo-release.yml@master
    with:
      targets: '["linux", "windows", "macos"]'
      deploy_pages: true
    secrets:
      CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

> **Important :** Les permissions `pages: write` et `id-token: write` doivent être déclarées dans le workflow **appelant**, pas dans le workflow réutilisable. C'est une limitation des reusable workflows de GitHub Actions.

---

## Exemple concret

Supposons un projet avec `Cargo.toml` version `1.0.0` :

1. Un développeur modifie `version = "1.1.0"` et push sur `main`.
2. **version-check** compare `1.1.0` (HEAD) vs `1.0.0` (HEAD~1) → `version_changed = true`.
3. **release** se déclenche :
   - Vérifie le formatage et lance les tests.
   - Compile pour Windows, Linux et macOS.
   - Crée une GitHub Release `v1.1.0` avec les binaires en pièces jointes.
   - (Optionnel) Publie le crate sur crates.io.

Si la version **n'a pas changé** (ex: simple correction de typo), le job `release` est ignoré.
