# 09 - GitHub Actions Avancées

[← 08 - Exercices](08-exercices.md) | [🏠 Accueil](README.md) | [10 - GitOps →](10-gitops.md)

---

## Objectifs de cette partie

- Configurer des Environments avec Deployment Gates (approbation manuelle prod)
- Créer des Reusable Workflows pour centraliser la logique CI/CD
- Mettre en place des Self-hosted runners Azure
- Contrôler la concurrence pour éviter les `terraform apply` parallèles

---

## 1. GitHub Environments et Deployment Gates

Les **Environments** représentent vos environnements Azure (dev, staging, prod). Chaque environment peut avoir ses propres secrets et des règles de protection — notamment une **approbation manuelle obligatoire** avant tout déploiement en production.

### Créer les Environments

**Settings → Environments → New environment**

| Environment | Protection rules | Secrets |
|---|---|---|
| `dev` | Aucune (déploiement automatique) | `AZURE_CLIENT_ID_DEV` |
| `staging` | Aucune | `AZURE_CLIENT_ID_STAGING` |
| `prod` | ✅ Required reviewers: `@alice @bob` | `AZURE_CLIENT_ID_PROD` |

### Utiliser les Environments dans un workflow

```yaml
# .github/workflows/deploy-multi-env.yml
name: Deploy Infrastructure — Multi-env

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy-dev:
    name: 🟢 Deploy Dev
    runs-on: ubuntu-latest
    environment: dev    # Déploiement automatique, pas de gate

    steps:
      - uses: actions/checkout@v4

      - name: Login Azure (OIDC — Dev)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: hashicorp/setup-terraform@v3
      - run: terraform init && terraform apply -auto-approve
        working-directory: environments/dev

  deploy-staging:
    name: 🟡 Deploy Staging
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: staging  # Démarre seulement si dev réussit

    steps:
      - uses: actions/checkout@v4
      - name: Login Azure (OIDC — Staging)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init && terraform apply -auto-approve
        working-directory: environments/staging

  deploy-prod:
    name: 🔴 Deploy Prod
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: prod   # ← PAUSE ICI : approbation manuelle requise !

    steps:
      - uses: actions/checkout@v4
      - name: Login Azure (OIDC — Prod)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init && terraform apply -auto-approve
        working-directory: environments/prod
```

Quand le job `deploy-prod` démarre, GitHub envoie une notification aux reviewers désignés. Le pipeline se met en pause jusqu'à ce que l'un d'eux approuve (ou rejette) le déploiement via l'interface GitHub.

---

## 2. Reusable Workflows

Les **Reusable Workflows** permettent de définir la logique CI/CD une seule fois dans un workflow "appelé" et de l'invoquer depuis n'importe quel autre workflow. Indispensable pour éviter de dupliquer la logique Terraform dans chaque repo.

### Créer un workflow réutilisable

```yaml
# .github/workflows/_terraform-deploy.yml  (préfixe _ = interne)
name: Terraform Deploy (Reusable)

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
        description: "Nom de l'environnement Azure (dev/staging/prod)"
      working_directory:
        required: true
        type: string
        description: "Chemin vers le dossier Terraform"
      terraform_version:
        required: false
        type: string
        default: '1.7.0'
    secrets:
      AZURE_CLIENT_ID:
        required: true
      AZURE_TENANT_ID:
        required: true
      AZURE_SUBSCRIPTION_ID:
        required: true

permissions:
  id-token: write
  contents: read

jobs:
  terraform:
    name: Deploy ${{ inputs.environment }}
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ inputs.terraform_version }}

      - name: Login Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        run: terraform init
        working-directory: ${{ inputs.working_directory }}

      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: ${{ inputs.working_directory }}
```

### Appeler le workflow réutilisable

```yaml
# .github/workflows/deploy.yml
name: Full Deployment Pipeline

on:
  push:
    branches: [main]

jobs:
  deploy-dev:
    uses: ./.github/workflows/_terraform-deploy.yml
    with:
      environment: dev
      working_directory: environments/dev
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID_DEV }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  deploy-prod:
    needs: deploy-dev
    uses: ./.github/workflows/_terraform-deploy.yml
    with:
      environment: prod
      working_directory: environments/prod
    secrets:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID_PROD }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

## 3. Self-hosted Runners Azure

Les runners GitHub-hosted n'ont pas accès aux ressources Azure privées (AKS en réseau privé, Key Vault sans accès public). Les **self-hosted runners** s'exécutent sur vos propres machines Azure.

### Déployer un runner sur ACI (Azure Container Instances)

```bash
# 1. Récupérer le token d'enregistrement depuis GitHub
REPO="mon-org/infra-azure-devops"
TOKEN=$(gh api repos/${REPO}/actions/runners/registration-token --jq .token)

# 2. Lancer le runner dans ACI
az container create \
  --resource-group rg-github-runners \
  --name github-runner-01 \
  --image ghcr.io/actions/runner:latest \
  --cpu 2 \
  --memory 4 \
  --environment-variables \
    REPO_URL="https://github.com/${REPO}" \
    RUNNER_TOKEN="${TOKEN}" \
    RUNNER_NAME="azure-runner-01" \
    RUNNER_LABELS="azure,terraform"
```

### Utiliser le self-hosted runner dans un workflow

```yaml
jobs:
  terraform-private:
    runs-on: [self-hosted, azure, terraform]  # Labels du runner
    steps:
      - uses: actions/checkout@v4
      - run: terraform apply -auto-approve
        working-directory: environments/prod
        # Ce job a accès aux ressources Azure privées !
```

> 💡 **Sécurité** : les self-hosted runners ont accès au réseau Azure. Utilisez des **managed identities** ACI plutôt que des secrets pour s'authentifier à Azure.

---

## 4. Contrôle de la concurrence

Deux `terraform apply` simultanés sur le même état Terraform provoquent des corruptions. La directive `concurrency` garantit qu'un seul job s'exécute à la fois.

```yaml
# Empêcher deux apply simultanés sur le même environnement
concurrency:
  group: terraform-prod
  cancel-in-progress: false  # false = attendre, true = annuler le précédent

jobs:
  apply:
    runs-on: ubuntu-latest
    # ...
```

### Concurrence par environment

```yaml
concurrency:
  group: terraform-${{ github.event.inputs.environment || 'prod' }}
  cancel-in-progress: false
```

---

[← 08 - Exercices](08-exercices.md) | [🏠 Accueil](README.md) | [10 - GitOps →](10-gitops.md)
