# 10 - GitOps avec GitHub

[← 09 - GitHub Actions Avancées](09-github-actions-avance.md) | [🏠 Accueil](README.md) | [11 - Solutions →](11-solutions.md)

---

## Objectifs de cette partie

- Comprendre le paradigme GitOps et ses principes
- Appliquer GitOps à l'infrastructure Azure avec GitHub
- Structurer un dépôt GitOps multi-environnements
- Détecter et corriger le drift entre Git et Azure
- Distinguer GitOps "push" et "pull"

---

## Qu'est-ce que le GitOps ?

Le **GitOps** est un modèle opérationnel qui utilise Git comme **unique source de vérité** pour l'état désiré de votre infrastructure et de vos applications. Toute modification passe par Git — jamais par le portail Azure, la CLI ou un script manuel.

### Les 4 principes du GitOps

**1. Déclaratif** : l'état désiré du système est décrit en code (Terraform, Bicep, YAML Kubernetes), pas en procédures impératives.

**2. Versionné et immuable** : toute l'histoire des changements est dans Git. Revenir à un état précédent = `git revert` + pipeline.

**3. Récupéré automatiquement** : un agent ou un pipeline tire l'état de Git et l'applique automatiquement.

**4. Réconcilié en continu** : si l'état réel diverge de l'état Git (drift), le système le détecte et le corrige.

---

## GitOps en pratique sur Azure

```
┌─────────────────────────────────────────────────────────┐
│                        GitHub                           │
│                                                         │
│  main branch = état DÉSIRÉ de l'infra Azure             │
│                                                         │
│  PR → Review → Merge → Pipeline → Azure                 │
└──────────────────────────────┬──────────────────────────┘
                               │ GitHub Actions (OIDC)
                               │ terraform apply / kubectl apply
                               ▼
┌─────────────────────────────────────────────────────────┐
│                    Azure (état RÉEL)                    │
│  AKS cluster  ·  VNet  ·  Key Vault  ·  ACR            │
└─────────────────────────────────────────────────────────┘
```

### Règle fondamentale

> **On ne modifie jamais Azure directement.** Ni par le portail, ni par `az` en dehors d'un pipeline. Tout changement = une PR sur GitHub.

Si un ingénieur fait un changement manuel en urgence, il doit immédiatement ouvrir une PR pour "coder" ce changement, sinon il sera écrasé au prochain `terraform apply`.

---

## Structurer un dépôt GitOps multi-environnements

```
infra-azure-gitops/
├── modules/                    # Modules Terraform réutilisables
│   ├── aks/
│   ├── network/
│   └── keyvault/
│
├── environments/               # Un dossier = un environnement Azure
│   ├── dev/
│   │   ├── main.tf             # Appelle les modules
│   │   ├── terraform.tfvars    # Valeurs dev (non sensibles)
│   │   └── backend.tf          # Backend Terraform (Azure Storage)
│   ├── staging/
│   └── prod/
│
├── kubernetes/                 # Manifests K8s déployés sur AKS
│   ├── dev/
│   └── prod/
│
├── .github/
│   ├── workflows/
│   │   ├── terraform.yml       # Pipeline IaC
│   │   ├── k8s-deploy.yml      # Pipeline K8s
│   │   └── drift-detection.yml # Vérification quotidienne du drift
│   ├── CODEOWNERS
│   └── pull_request_template.md
│
└── .gitignore
```

### Backend Terraform dans Azure Storage

Pour stocker l'état Terraform dans Azure (et non localement) :

```hcl
# environments/prod/backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstateprod"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

---

## GitOps "Push" vs "Pull"

Il existe deux modèles d'application du GitOps :

### Push GitOps (GitHub Actions)

GitHub Actions **pousse** les changements vers Azure après chaque merge.

```
Git merge → GitHub Actions → terraform apply → Azure
```

✅ Simple à mettre en place  
✅ Adapté à l'IaC (Terraform, Bicep)  
⚠️ Le pipeline doit avoir des credentials Azure (OIDC)

### Pull GitOps (ArgoCD / Flux)

Un agent tourne **dans** le cluster AKS et **tire** en continu l'état depuis Git.

```
Git merge → ArgoCD/Flux (dans AKS) détecte le changement → applique
```

✅ Pas de credentials Azure dans GitHub  
✅ Réconciliation continue (corrige le drift automatiquement)  
✅ Idéal pour les manifests Kubernetes  
⚠️ Plus complexe à installer

**En pratique** : on utilise souvent les deux — GitHub Actions pour l'IaC Terraform (infrastructure), ArgoCD/Flux pour les déploiements Kubernetes (applications).

---

## Détection de drift

Le **drift** survient quand l'état réel d'Azure diverge de ce qui est dans Git (changement manuel, ressource supprimée accidentellement, quota Azure modifié).

### Workflow de détection quotidienne

```yaml
# .github/workflows/drift-detection.yml
name: Drift Detection

on:
  schedule:
    - cron: '0 6 * * 1-5'   # Tous les jours ouvrés à 6h
  workflow_dispatch:          # Déclenchement manuel possible

permissions:
  id-token: write
  contents: read
  issues: write               # Pour créer une issue si drift détecté

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      - name: Login Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        run: terraform init
        working-directory: environments/${{ matrix.environment }}

      - name: Terraform Plan (drift check)
        id: plan
        run: |
          terraform plan -detailed-exitcode -no-color 2>&1 | tee plan.txt
          echo "exitcode=${PIPESTATUS[0]}" >> $GITHUB_OUTPUT
        working-directory: environments/${{ matrix.environment }}
        continue-on-error: true

      # exitcode 2 = des changements existent = drift détecté
      - name: Create Issue if drift detected
        if: steps.plan.outputs.exitcode == '2'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('environments/${{ matrix.environment }}/plan.txt', 'utf8');
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `🚨 Drift détecté sur ${{ matrix.environment }}`,
              body: `Un drift a été détecté entre Git et Azure.\n\n\`\`\`\n${plan.substring(0, 3000)}\n\`\`\``,
              labels: ['incident', 'infrastructure', 'priority:high']
            });
```

---

## Bonnes pratiques GitOps

**Jamais de changement manuel sur Azure** : toute modification par le portail ou la CLI doit être immédiatement codée dans Git.

**Terraform State dans Azure Storage** : le state file ne doit jamais être versionné dans Git, uniquement dans un backend distant.

**Un environnement = un dossier = un state** : facilite les déploiements progressifs et limite le rayon d'impact.

**Les PR comme audit trail** : chaque PR représente une décision d'architecture. Utilisez des descriptions détaillées — elles servent de journal de bord à l'équipe.

**Drift detection régulière** : planifiez un `terraform plan` quotidien pour détecter rapidement les écarts.

---

[← 09 - GitHub Actions Avancées](09-github-actions-avance.md) | [🏠 Accueil](README.md) | [11 - Solutions →](11-solutions.md)
