# 04 - Issues et Gestion de projet

[← 03 - Pull Requests](03-pull-requests-code-review.md) | [🏠 Accueil](README.md) | [05 - GitHub Actions →](05-github-actions-cicd.md)

---

## Objectifs de cette partie

- Créer et gérer des Issues pour les incidents et évolutions d'infrastructure
- Organiser avec Labels et Milestones DevOps
- Maîtriser GitHub Projects pour piloter les chantiers d'infra
- Lier automatiquement Issues et Pull Requests

---

## Les Issues en DevOps Azure

Les **Issues** sont le système de tickets de GitHub. En DevOps Azure, elles servent à :

- 🚨 Signaler un incident d'infrastructure (AKS en panne, quota dépassé)
- 💡 Proposer une évolution (nouveau module Terraform, migration vers Bicep)
- 🔒 Tracer une remédiation de sécurité (faille NSG, rotation de secrets)
- 📋 Planifier une mise à jour de provider Terraform
- 💬 Discuter d'une décision d'architecture

### Exemple d'issue bien rédigée — Incident AKS

```markdown
## 🚨 Incident : Node pool AKS saturé en production

**Description**
Le node pool `default` du cluster `aks-cloudops-prod` est à 100% de capacité.
Les nouveaux pods sont en état `Pending` depuis 14h35.

**Impact**
- Environnement : Production
- Services impactés : API backend, workers batch
- Depuis : 2024-03-15 14:35 UTC

**Logs / Alertes**
```
kubectl describe node aks-default-xxxxx
  Allocatable: cpu: 1930m, memory: 5GiB
  Allocated: cpu: 1920m (99%), memory: 5GiB (99%)
```

**Hypothèse de cause**
Augmentation soudaine de trafic suite au lancement marketing. Le HPA a atteint
maxReplicas mais les nodes sont insuffisants.

**Actions envisagées**
- Court terme : augmenter manuellement `node_count` dans Terraform (PR #92)
- Long terme : activer le Cluster Autoscaler (Issue #93)
```

---

## Labels : Organiser les Issues DevOps

Créez des labels adaptés à vos chantiers d'infrastructure :

| Label | Usage |
|---|---|
| `incident` | Problème actif en production |
| `infrastructure` | Modification de ressources Azure |
| `security` | Faille, rotation de secrets, RBAC |
| `terraform` | Mise à jour provider, refacto IaC |
| `kubernetes` | AKS, manifests, Helm |
| `ci-cd` | Workflows GitHub Actions |
| `priority:critical` | À traiter dans l'heure |
| `priority:high` | À traiter dans la journée |
| `priority:low` | Backlog |
| `good-first-issue` | Bon pour les nouveaux membres |

---

## Milestones : Planifier les releases d'infrastructure

Les **Milestones** regroupent les Issues autour d'un objectif ou d'une version.

**Exemple : Milestone "Infrastructure v2.0 — Migration AKS 1.29"**

- Date cible : 30 juin 2025
- Description : Migration du cluster AKS vers la version 1.29 avec activation du Cluster Autoscaler
- Issues liées : 12 issues (8 fermées, 4 ouvertes)
- Progression : 67%

---

## GitHub Projects : Tableau Kanban DevOps

**GitHub Projects** est le système de gestion de projet intégré. En DevOps Azure, il permet de visualiser l'avancement des chantiers d'infrastructure en temps réel.

### Colonnes typiques pour un projet d'infra

```
┌───────────┬───────────┬─────────────┬──────────┬──────────┐
│  Backlog  │   To Do   │ In Progress │  Review  │   Done   │
│           │           │             │          │          │
│ Issue #95 │ Issue #91 │  Issue #88  │  PR #90  │ Issue #85│
│ Issue #96 │ Issue #92 │  Issue #89  │          │ Issue #86│
└───────────┴───────────┴─────────────┴──────────┴──────────┘
```

- **Backlog** : évolutions et idées non encore planifiées
- **To Do** : issues assignées, prêtes à démarrer
- **In Progress** : en cours de développement IaC
- **Review** : PR ouverte, en attente d'approbation CODEOWNERS
- **Done** : mergé et déployé sur Azure

### Automatisation

GitHub Projects peut déplacer automatiquement les items :
- Issue assignée → **To Do**
- PR créée → **Review**
- PR mergée → **Done** + ferme l'issue liée

---

## Lier Issues et Pull Requests

Utilisez des mots-clés dans la description de vos PR pour fermer automatiquement les issues au merge :

```markdown
## Description

Correction de la saturation du node pool AKS en augmentant la capacité
et en activant le Cluster Autoscaler.

## Closes

Closes #88
Fixes #89
```

Mots-clés reconnus : `closes`, `fixes`, `resolves` (insensible à la casse).

Quand la PR est mergée dans `main`, les issues liées sont automatiquement fermées et passent en **Done** dans GitHub Projects.

---

[← 03 - Pull Requests](03-pull-requests-code-review.md) | [🏠 Accueil](README.md) | [05 - GitHub Actions →](05-github-actions-cicd.md)
