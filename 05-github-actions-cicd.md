# 05 - GitHub Actions : CI/CD Azure

[← 04 - Issues et Projects](04-issues-gestion-projet.md) | [🏠 Accueil](README.md) | [06 - Sécurité →](06-securite-bonnes-pratiques.md)

---

## Objectifs de cette partie

- Comprendre GitHub Actions et ses concepts clés
- Configurer l'authentification OIDC avec Azure (sans credentials long-lived)
- Créer un pipeline Terraform CI/CD complet
- Déployer sur AKS via GitHub Actions
- Gérer les secrets et variables d'environnement

---

## Qu'est-ce que GitHub Actions ?

**GitHub Actions** est un système d'automatisation intégré à GitHub. Chaque workflow est un fichier YAML dans `.github/workflows/` déclenché par des événements Git (push, PR, tag, schedule...).

### Concepts clés

| Concept | Description |
|---|---|
| **Workflow** | Processus automatisé défini dans un fichier YAML |
| **Job** | Ensemble d'étapes s'exécutant sur le même runner |
| **Step** | Commande ou action individuelle |
| **Runner** | Machine virtuelle (Ubuntu, macOS, Windows) ou self-hosted |
| **Action** | Composant réutilisable du marketplace |
| **Event** | Déclencheur : push, pull_request, schedule, workflow_dispatch... |

---

## 1. Authentification Azure avec OIDC (recommandé)

L'**OIDC (OpenID Connect)** est la méthode moderne pour authentifier GitHub Actions vers Azure **sans stocker de secrets long-lived**. GitHub génère un token JWT à chaque exécution ; Azure le valide directement.

### Pourquoi OIDC plutôt qu'un Service Principal avec secret ?

| | Service Principal (ancien) | OIDC (recommandé) |
|---|---|---|
| **Secret stocké dans GitHub** | Oui (`AZURE_CLIENT_SECRET`) | Non |
| **Rotation des credentials** | Manuelle, risque d'oubli | Automatique (token éphémère) |
| **Surface d'attaque** | Secret permanent volable | Token valable quelques minutes |
| **Conformité** | ❌ Déconseillé | ✅ Recommandé par Microsoft |

### Configurer OIDC (une seule fois)

**Étape 1 : Créer une App Registration Azure**

```bash
# Créer l'App Registration
az ad app create --display-name "github-actions-infra-azure"

# Récupérer l'appId
APP_ID=$(az ad app list --display-name "github-actions-infra-azure" --query "[0].appId" -o tsv)

# Créer le Service Principal associé
az ad sp create --id $APP_ID

# Récupérer l'Object ID du SP
SP_OBJECT_ID=$(az ad sp show --id $APP_ID --query "id" -o tsv)
```

**Étape 2 : Créer la Federated Credential**

```bash
# Autoriser GitHub Actions à s'authentifier (branche main)
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:mon-org/infra-azure-devops:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Autoriser aussi les Pull Requests
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-pr",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:mon-org/infra-azure-devops:pull_request",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

**Étape 3 : Assigner les permissions Azure**

```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Rôle Contributor sur la subscription (ou scope plus restreint)
az role assignment create \
  --assignee $APP_ID \
  --role "Contributor" \
  --scope "/subscriptions/${SUBSCRIPTION_ID}"
```

**Étape 4 : Ajouter les secrets GitHub** (sans le `AZURE_CLIENT_SECRET` !)

Dans **Settings → Secrets and variables → Actions** :

| Secret | Valeur |
|---|---|
| `AZURE_CLIENT_ID` | `appId` de l'App Registration |
| `AZURE_TENANT_ID` | ID de votre tenant Azure |
| `AZURE_SUBSCRIPTION_ID` | ID de votre subscription |

---

## 2. Pipeline Terraform CI/CD complet

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD Azure

on:
  pull_request:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'
  push:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'

permissions:
  id-token: write   # Requis pour OIDC
  contents: read
  pull-requests: write  # Pour commenter le plan sur la PR

env:
  TF_VERSION: '1.7.0'
  WORKING_DIR: ./environments/prod

jobs:
  terraform-validate:
    name: 🔍 Validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Login Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        run: terraform init
        working-directory: ${{ env.WORKING_DIR }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate
        working-directory: ${{ env.WORKING_DIR }}

  terraform-plan:
    name: 📋 Plan
    needs: terraform-validate
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    outputs:
      plan_exitcode: ${{ steps.plan.outputs.exitcode }}

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Login Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        run: terraform init
        working-directory: ${{ env.WORKING_DIR }}

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan -no-color -detailed-exitcode -out=tfplan 2>&1 | tee plan_output.txt
          echo "exitcode=${PIPESTATUS[0]}" >> $GITHUB_OUTPUT
        working-directory: ${{ env.WORKING_DIR }}
        continue-on-error: true

      # Publier le plan en commentaire sur la PR
      - name: Comment PR with Plan
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('${{ env.WORKING_DIR }}/plan_output.txt', 'utf8');
            const maxLen = 65000;
            const truncated = plan.length > maxLen ? plan.substring(0, maxLen) + '\n...(tronqué)' : plan;
            const body = `## 📋 Terraform Plan\n\`\`\`hcl\n${truncated}\n\`\`\``;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body
            });

      - name: Fail if plan errored
        if: steps.plan.outputs.exitcode == '1'
        run: exit 1

  terraform-apply:
    name: 🚀 Apply
    needs: terraform-validate
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Login Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Init
        run: terraform init
        working-directory: ${{ env.WORKING_DIR }}

      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: ${{ env.WORKING_DIR }}
```

---

## 3. Déploiement sur AKS

```yaml
# .github/workflows/deploy-aks.yml
name: Build and Deploy to AKS

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'kubernetes/**'

permissions:
  id-token: write
  contents: read

jobs:
  build-push:
    name: 🐳 Build & Push ACR
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Login Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Build and push to ACR
        id: meta
        run: |
          TAG=$(git rev-parse --short HEAD)
          az acr build \
            --registry ${{ vars.ACR_NAME }} \
            --image mon-app:${TAG} \
            --image mon-app:latest .
          echo "version=${TAG}" >> $GITHUB_OUTPUT

  deploy-aks:
    name: ☸️ Deploy AKS
    needs: build-push
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Login Azure (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Set AKS context
        uses: azure/aks-set-context@v3
        with:
          resource-group: rg-cloudops-prod
          cluster-name: aks-cloudops-prod

      - name: Deploy to AKS
        uses: azure/k8s-deploy@v4
        with:
          manifests: kubernetes/deployment.yaml
          images: ${{ vars.ACR_NAME }}.azurecr.io/mon-app:${{ needs.build-push.outputs.image_tag }}
```

---

## 4. Secrets et Variables

### Hiérarchie

| Niveau | Portée | Cas d'usage |
|---|---|---|
| **Repository secret** | Un seul repo | Credentials propres au projet |
| **Repository variable** | Un seul repo | Config non-sensible (`ACR_NAME`) |
| **Environment secret** | Un env (prod/staging) | Credentials par environnement |
| **Organization secret** | Tous les repos de l'org | Credentials partagés |

### Bonnes pratiques

- Ne jamais mettre de secret dans le YAML du workflow
- Utiliser OIDC plutôt que des secrets pour Azure
- Les secrets sont masqués dans les logs automatiquement
- Faites une rotation régulière des secrets résiduels

---

[← 04 - Issues et Projects](04-issues-gestion-projet.md) | [🏠 Accueil](README.md) | [06 - Sécurité →](06-securite-bonnes-pratiques.md)
