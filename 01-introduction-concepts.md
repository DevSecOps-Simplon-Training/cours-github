# 01 - Introduction à GitHub

[🏠 Retour à l'accueil](README.md) | [02 - Premiers pas →](02-premiers-pas-ssh.md)

---

## Objectifs de cette partie

- Comprendre ce qu'est GitHub et son rôle en DevOps Azure
- Distinguer Git et GitHub
- Découvrir les concepts clés de la plateforme
- Situer GitHub dans la chaîne DevOps Azure
- Comparer GitHub et Azure DevOps

---

## Qu'est-ce que GitHub ?

**GitHub** est une plateforme web d'hébergement et de gestion de code source basée sur Git. Lancée en 2008 et rachetée par Microsoft en 2018, c'est la plus grande plateforme de développement collaboratif au monde avec plus de 100 millions de développeurs.

Dans un contexte **DevOps Azure**, GitHub est bien plus qu'un simple hébergeur de code : c'est le **point de contrôle central** de toute modification d'infrastructure avant qu'elle n'atteigne Azure.

### GitHub vs Git

| Aspect | Git | GitHub |
|---|---|---|
| **Type** | Outil local de versionnage | Plateforme cloud collaborative |
| **Installation** | Sur votre machine | Hébergé par Microsoft |
| **Fonction principale** | Versionner du code / IaC | Héberger, collaborer, automatiser |
| **Interface** | Ligne de commande | Interface web + CLI + API |
| **Collaboration** | Possible mais limitée | Optimisée : PR, Reviews, Issues |

**Git** est le moteur local. **GitHub** est la plateforme qui orchestre la collaboration et les déploiements Azure autour de ce moteur.

---

## Pourquoi GitHub est central en DevOps Azure ?

### 🏗️ Source de vérité pour l'infrastructure

Tous vos fichiers Terraform, Bicep, manifests Kubernetes et scripts Azure CLI vivent dans GitHub. Personne ne modifie Azure directement via le portail — tout passe par un commit, une Pull Request et un pipeline automatisé. C'est le principe fondateur du **GitOps**.

### 🔄 CI/CD intégré nativement

**GitHub Actions** déclenche automatiquement `terraform plan` à chaque PR et `terraform apply` après merge sur `main`, sans outil externe. L'authentification avec Azure se fait via **OIDC** (sans stocker de credentials).

### 🔒 Contrôle d'accès et traçabilité

Les **Branch Rulesets** et **CODEOWNERS** garantissent qu'aucune modification d'infrastructure sensible n'est mergée sans la validation des bonnes personnes. Chaque action est auditée.

### 🛡️ Sécurité proactive

**Secret Scanning** bloque la publication accidentelle de credentials Azure. **Dependabot** maintient les versions des providers Terraform à jour. **CodeQL** détecte les failles dans les scripts.

### 📊 Gestion de projet intégrée

**Issues**, **Projects** et **Milestones** permettent de piloter les chantiers d'infrastructure (migration AKS, mise à jour providers, remédiation sécurité) sans quitter l'outil.

---

## Les concepts clés de GitHub

| Concept | Description en DevOps Azure |
|---|---|
| **Repository** | Dépôt contenant votre IaC, manifests K8s et pipelines CI/CD |
| **Fork** | Copie d'un module Terraform public pour le personnaliser |
| **Pull Request (PR)** | Validation obligatoire avant tout `terraform apply` en production |
| **Issue** | Ticket pour un incident infra, une évolution ou une faille de sécurité |
| **Actions** | Workflows CI/CD déclenchant les déploiements Azure |
| **Environments** | Représentation de dev/staging/prod avec leurs secrets et leurs gates |
| **CODEOWNERS** | Fichier définissant qui doit approuver chaque partie de l'infra |

---

## GitHub dans la chaîne DevOps Azure

```
Ingénieur DevOps
      │
      │  git push → Pull Request
      ▼
  ┌──────────────────────────────────────┐
  │              GitHub                  │
  │  Branch Rulesets ── CODEOWNERS       │
  │  Secret Scanning ── Dependabot       │
  └────────────────┬─────────────────────┘
                   │ GitHub Actions (OIDC)
                   ▼
  ┌──────────────────────────────────────┐
  │              Azure                   │
  │  terraform plan / apply → Resources  │
  │  az acr build           → ACR        │
  │  kubectl apply          → AKS        │
  └──────────────────────────────────────┘
```

---

## GitHub vs Azure DevOps

Les deux plateformes appartiennent à Microsoft et s'intègrent avec Azure. Le choix dépend du contexte :

| Critère | GitHub | Azure DevOps |
|---|---|---|
| **CI/CD** | GitHub Actions | Azure Pipelines |
| **Gestion de projet** | Projects (Kanban/Table/Roadmap) | Boards (plus complet, intégration Jira) |
| **Dépôts** | GitHub Repos | Azure Repos |
| **Intégration Azure** | Via OIDC + `azure/login` | Native (Service Connection) |
| **Communauté** | La plus grande, open source | Principalement entreprises |
| **Cas d'usage** | Équipes modernes, cloud-native | Organisations avec historique Microsoft |

> 💡 **En pratique** : les deux coexistent souvent en entreprise. Le code et les pipelines sur GitHub, la gestion de backlog sur Azure DevOps Boards.

---

[🏠 Retour à l'accueil](README.md) | [02 - Premiers pas →](02-premiers-pas-ssh.md)
