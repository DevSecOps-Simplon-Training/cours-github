# 03 - Pull Requests, CODEOWNERS et Templates

[← 02 - Premiers pas](02-premiers-pas-ssh.md) | [🏠 Accueil](README.md) | [04 - Issues et Projects →](04-issues-gestion-projet.md)

---

## Objectifs de cette partie

- Maîtriser le workflow Pull Request pour l'infrastructure
- Configurer CODEOWNERS pour protéger les modules critiques
- Créer des templates de PR et d'Issue adaptés au DevOps
- Effectuer des code reviews constructives sur du code IaC
- Choisir la bonne stratégie de merge

---

## Le rôle central de la Pull Request en DevOps Azure

Une **Pull Request** n'est pas simplement un mécanisme de code review : c'est la **porte d'entrée obligatoire** avant tout changement d'infrastructure Azure. Chaque PR déclenche automatiquement `terraform plan` et affiche le diff des ressources Azure qui seront créées, modifiées ou supprimées.

```
feature/add-aks-nodepool
         │
         │  git push
         ▼
    ┌──────────┐
    │    PR    │──── terraform plan (CI) ──── affiche le diff Azure
    └────┬─────┘
         │  Approbations CODEOWNERS + tests verts
         ▼
       main ──── terraform apply ──── Azure (prod)
```

---

## Workflow complet d'une PR d'infrastructure

### Étape 1 : Créer une branche

```bash
git switch main && git pull origin main
git switch -c feature/add-aks-nodepool
```

### Étape 2 : Développer et committer

```bash
# Modifier modules/aks/main.tf
git add modules/aks/main.tf
git commit -m "feat(aks): add spot node pool for batch workloads"
git push -u origin feature/add-aks-nodepool
```

### Étape 3 : Ouvrir la PR sur GitHub

GitHub affiche une bannière "Compare & pull request" dès que vous poussez une branche.

### Étape 4 : Rédiger une description complète

```markdown
## 🎯 Objectif

Ajout d'un node pool Spot pour les workloads batch (coût réduit de 70%).

## 📝 Changements Terraform

- `modules/aks/main.tf` : ajout de la ressource `azurerm_kubernetes_cluster_node_pool`
- `modules/aks/variables.tf` : nouvelles variables `spot_node_count`, `spot_vm_size`
- `environments/prod/terraform.tfvars` : activation du node pool Spot en prod

## 🔍 Terraform Plan

```
# azurerm_kubernetes_cluster_node_pool.spot will be created
+ resource "azurerm_kubernetes_cluster_node_pool" "spot" {
    + name            = "spot"
    + node_count      = 2
    + priority        = "Spot"
    + vm_size         = "Standard_D4s_v3"
  }

Plan: 1 to add, 0 to change, 0 to destroy.
```

## ✅ Checklist

- [x] `terraform validate` passe
- [x] `terraform plan` attaché ci-dessus
- [x] Pas de régression sur les ressources existantes
- [x] Variables documentées dans variables.tf

## ⚠️ Points d'attention

- Vérifier la disponibilité des VMs Spot dans la région West Europe
- Les nœuds Spot peuvent être évincés : les workloads doivent être tolerants

Closes #87
```

---

## CODEOWNERS : Protéger les modules critiques

Le fichier `.github/CODEOWNERS` définit qui doit **obligatoirement** approuver les modifications dans chaque répertoire. En DevOps Azure, c'est indispensable : personne ne doit pouvoir modifier un module réseau ou les variables de production sans que le bon expert valide.

### Créer `.github/CODEOWNERS`

```
# .github/CODEOWNERS
# Syntaxe : <pattern>  <@user ou @org/team>

# Tout le dépôt : revue obligatoire de l'équipe infra
*                           @mon-org/infra-team

# Modules réseau : validé par l'architecte réseau
/modules/network/           @alice-network-architect

# Module AKS : validé par l'expert Kubernetes
/modules/aks/               @bob-k8s-lead

# Environnement de production : double validation requise
/environments/prod/         @alice-network-architect @bob-k8s-lead

# Pipelines CI/CD : validé par le lead DevOps
/.github/workflows/         @charlie-devops-lead

# Fichiers de sécurité : validé par l'équipe sécurité
/modules/keyvault/          @mon-org/security-team
/.github/CODEOWNERS         @mon-org/security-team
```

> ⚠️ **Important** : pour que CODEOWNERS soit appliqué, la branche `main` doit avoir une **branch protection rule** avec "Require review from Code Owners" activé (voir module 06).

---

## Templates de PR et d'Issue

Les templates standardisent les informations fournies par les équipes et réduisent les allers-retours.

### Template de Pull Request : `.github/pull_request_template.md`

```markdown
## 🎯 Objectif
<!-- Décrivez le problème résolu ou la fonctionnalité ajoutée -->

## 📝 Changements IaC
<!-- Listez les fichiers Terraform/Bicep/YAML modifiés et pourquoi -->

## 🔍 Terraform Plan / az diff
<!-- Collez la sortie de terraform plan ou le diff de ressources Azure -->

## ✅ Checklist
- [ ] `terraform validate` et `terraform fmt` passent
- [ ] Pas de secrets dans les fichiers committés
- [ ] Variables documentées
- [ ] `.gitignore` à jour si nécessaire
- [ ] README mis à jour si l'interface du module change

## ⚠️ Risques et points d'attention pour les reviewers
<!-- Y a-t-il des ressources qui seront recréées (destroy + create) ? -->

## 🔗 Issue liée
Closes #
```

### Template d'Issue : `.github/ISSUE_TEMPLATE/incident-infra.md`

```markdown
---
name: Incident Infrastructure
about: Signaler un incident ou une dégradation sur l'infrastructure Azure
labels: incident, priority:high
---

## 🚨 Description de l'incident
<!-- Que se passe-t-il ? -->

## Impact
- Environnement concerné : [ ] dev  [ ] staging  [ ] prod
- Services impactés :
- Depuis quand :

## Symptômes observés
<!-- Logs Azure Monitor, alertes, messages d'erreur -->

## Hypothèse de cause
<!-- NSG mal configuré ? Node pool AKS saturé ? Quota dépassé ? -->

## Actions de remédiation envisagées
<!-- Quel hotfix ou rollback est envisagé ? -->
```

---

## Code Review IaC : bonnes pratiques

### Pour l'auteur de la PR

**À faire :** PR petite et focalisée sur une seule ressource ou décision. Toujours joindre le `terraform plan`. Répondre rapidement aux commentaires.

**À éviter :** Mélanger un nouveau module AKS et une refactorisation réseau dans la même PR. Description vide. Push force sur une branche partagée.

### Pour le reviewer

**Checklist de review IaC :**

- La ressource va-t-elle être **recréée** (`-/+`) alors qu'on voulait juste la modifier (`~`) ?
- Y a-t-il des `sensitive = true` manquants sur les outputs ?
- Les variables ont-elles des descriptions et des types explicites ?
- Le `.gitignore` couvre-t-il les nouveaux fichiers générés ?
- Les règles de sécurité (NSG, RBAC) respectent-elles le principe du moindre privilège ?

---

## Stratégies de merge

| Type | Comportement | Recommandé pour |
|---|---|---|
| **Merge commit** | Crée un commit de fusion, historique complet | Projets où l'audit trail est prioritaire |
| **Squash and merge** | Combine tous les commits en un seul propre | Historique lisible sur `main` ✅ |
| **Rebase and merge** | Historique linéaire strict | Équipes avec discipline de commit élevée |

Pour la plupart des dépôts IaC, **Squash and merge** est le meilleur choix : un commit sur `main` = une fonctionnalité ou un fix = un déploiement Azure potentiel.

---

[← 02 - Premiers pas](02-premiers-pas-ssh.md) | [🏠 Accueil](README.md) | [04 - Issues et Projects →](04-issues-gestion-projet.md)
