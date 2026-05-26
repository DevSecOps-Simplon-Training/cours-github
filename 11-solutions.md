# 11 - Solutions des exercices

[← 10 - GitOps](10-gitops.md) | [🏠 Accueil](README.md)

---

## 📝 Solutions Niveau 1 : Premiers pas

```bash
# 2. Générer la clé SSH et l'ajouter à GitHub
ssh-keygen -t ed25519 -C "votre.email@cloudops.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub   # Copier et coller dans GitHub → Settings → SSH keys

# Tester
ssh -T git@github.com

# 4. Cloner en SSH, créer versions.tf, committer
git clone git@github.com:votre-user/infra-azure-practice.git
cd infra-azure-practice

cat > versions.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85"
    }
  }
}
EOF

git add versions.tf
git commit -m "chore: add Terraform version constraints"
git push

# 5. GitHub CLI
gh auth login   # Suivre les instructions
gh repo view    # Vérifier la connexion
```

---

## 🌿 Solutions Niveau 2 : Pull Requests et CODEOWNERS

```bash
# 1. Créer la structure
mkdir -p .github modules/aks modules/network
touch .github/CODEOWNERS .github/pull_request_template.md
touch modules/aks/main.tf modules/network/main.tf
```

**`.github/CODEOWNERS`** :
```
# Tout le dépôt
*                    @votre-username

# Module AKS : second reviewer requis
/modules/aks/        @votre-username @collègue-username
```

**`.github/pull_request_template.md`** :
```markdown
## 🎯 Objectif
<!-- Décrivez le changement -->

## 📝 Changements IaC
<!-- Fichiers modifiés -->

## 🔍 Terraform Plan
<!-- Coller la sortie de terraform plan -->

## ✅ Checklist
- [ ] terraform validate passe
- [ ] terraform fmt appliqué
- [ ] Pas de secrets dans les fichiers
- [ ] Variables documentées

## 🔗 Issue liée
Closes #
```

```bash
# 4. Créer la branche, ouvrir la PR et merger
git switch -c feature/add-aks-module

cat >> modules/aks/main.tf << 'EOF'
resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-practice"
  location            = "West Europe"
  resource_group_name = "rg-practice"
  dns_prefix          = "aks-practice"

  default_node_pool {
    name       = "default"
    node_count = 2
    vm_size    = "Standard_D2_v2"
  }

  identity {
    type = "SystemAssigned"
  }
}
EOF

git add .
git commit -m "feat(aks): add AKS cluster resource"
git push -u origin feature/add-aks-module

# Créer la PR via CLI
gh pr create \
  --title "feat(aks): add AKS cluster resource" \
  --body "Ajout du module AKS avec node pool par défaut." \
  --label "infrastructure"

# Merger en Squash
gh pr merge --squash --delete-branch
```

**Ruleset `main`** (via Settings → Rules → Rulesets) :
- Target : branch `main`
- ✅ Require pull request (1 approval, dismiss stale reviews, require code owners)
- ✅ Require status checks
- ✅ Block force pushes

---

## ⚙️ Solutions Niveau 3 : GitHub Actions CI/CD Azure

```bash
# 1. Créer l'App Registration et la Federated Credential
APP_ID=$(az ad app create --display-name "github-actions-practice" --query appId -o tsv)
az ad sp create --id $APP_ID
SP_OID=$(az ad sp show --id $APP_ID --query id -o tsv)
SUB_ID=$(az account show --query id -o tsv)

az role assignment create --assignee $APP_ID --role Contributor \
  --scope "/subscriptions/${SUB_ID}"

# Federated credential pour main
az ad app federated-credential create --id $APP_ID --parameters '{
  "name": "github-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:votre-user/infra-azure-practice:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Federated credential pour pull_request
az ad app federated-credential create --id $APP_ID --parameters '{
  "name": "github-pr",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:votre-user/infra-azure-practice:pull_request",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Récupérer les valeurs pour les secrets GitHub
echo "AZURE_CLIENT_ID: $APP_ID"
echo "AZURE_TENANT_ID: $(az account show --query tenantId -o tsv)"
echo "AZURE_SUBSCRIPTION_ID: $SUB_ID"
```

```bash
# Ajouter les secrets via gh CLI
gh secret set AZURE_CLIENT_ID --body "$APP_ID"
gh secret set AZURE_TENANT_ID --body "$(az account show --query tenantId -o tsv)"
gh secret set AZURE_SUBSCRIPTION_ID --body "$SUB_ID"
```

**`.github/workflows/terraform.yml`** :

```yaml
name: Terraform CI/CD

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - name: Login Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - run: terraform init
      - run: terraform fmt -check
      - run: terraform validate

  plan:
    needs: validate
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - run: terraform init
      - id: plan
        run: terraform plan -no-color 2>&1 | tee plan.txt
        continue-on-error: true
      - uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '## Terraform Plan\n```\n' + plan.substring(0, 10000) + '\n```'
            });

  apply:
    needs: validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - run: terraform init && terraform apply -auto-approve
```

**`.github/workflows/security.yml`** :

```yaml
name: IaC Security Scan
on:
  pull_request:
    paths: ['**.tf']
jobs:
  tfsec:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/tfsec-action@v1.0.3
        with:
          soft_fail: false
```

---

## 🔒 Solutions Niveau 4 : Sécurité et GitOps

```bash
# 1. Tester le Push Protection
# Activer dans Settings → Code security → Secret scanning → Push protection

# Tenter de committer un faux secret
echo "AZURE_CLIENT_SECRET=FakeSecret123!" > test-secret.env
git add test-secret.env
git commit -m "test: should be blocked"
git push
# → Push bloqué par GitHub Secret Scanning
git restore --staged test-secret.env
rm test-secret.env
```

**`.github/dependabot.yml`** :

```yaml
version: 2
updates:
  - package-ecosystem: "terraform"
    directory: "/environments/dev"
    schedule:
      interval: "weekly"
    labels: ["terraform", "dependencies"]

  - package-ecosystem: "terraform"
    directory: "/environments/prod"
    schedule:
      interval: "weekly"
    labels: ["terraform", "dependencies"]

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels: ["ci-cd", "dependencies"]
```

**Bonus GitOps — Garantir que apply ne s'exécute que depuis main :**

```yaml
# Dans le job apply du workflow terraform.yml
apply:
  needs: validate
  runs-on: ubuntu-latest
  # Double condition : branche main ET event push (pas workflow_dispatch depuis une branche)
  if: |
    github.ref == 'refs/heads/main' &&
    github.event_name == 'push' &&
    github.event.base_ref == null
  steps:
    # ...
```

---

[← 10 - GitOps](10-gitops.md) | [🏠 Accueil](README.md)
