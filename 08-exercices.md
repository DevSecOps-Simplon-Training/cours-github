# 08 - Exercices : GitHub pour le DevOps Azure

[← 07 - Fonctionnalités avancées](07-fonctionnalites-avancees.md) | [🏠 Accueil](README.md) | [09 - GitHub Actions Avancées →](09-github-actions-avance.md)

---

## 📝 Niveau 1 : Premiers pas

1. Créez un compte GitHub et configurez votre profil avec une bio DevOps Azure.
2. Générez une clé SSH ED25519 et ajoutez-la à votre compte GitHub. Vérifiez la connexion avec `ssh -T git@github.com`.
3. Créez un dépôt privé `infra-azure-practice` avec le `.gitignore` Terraform généré par GitHub.
4. Clonez-le en SSH, créez un fichier `versions.tf` avec les contraintes de version Terraform et Azure, committez et poussez.
5. Installez GitHub CLI (`gh`) et connectez-vous avec `gh auth login`.

---

## 🌿 Niveau 2 : Pull Requests et CODEOWNERS

1. Dans votre dépôt `infra-azure-practice`, créez la structure suivante :
   ```
   .github/
   ├── CODEOWNERS
   └── pull_request_template.md
   modules/
   ├── aks/main.tf
   └── network/main.tf
   ```
2. Rédigez un fichier `CODEOWNERS` qui :
   - Assigne votre propre compte (`@votre-username`) à tout le dépôt
   - Assigne un second reviewer (un collègue ou un second compte) spécifiquement au dossier `modules/aks/`
3. Créez un template de PR avec les sections : Objectif, Changements, Terraform Plan, Checklist, Issue liée.
4. Créez une branche `feature/add-aks-module`, ajoutez du contenu à `modules/aks/main.tf`, ouvrez une PR avec une description complète en utilisant le template, puis mergez en **Squash and merge**.
5. Activez la protection de branche sur `main` (Rulesets) : PR obligatoire, 1 approbation, CI doit passer.

---

## ⚙️ Niveau 3 : GitHub Actions CI/CD Azure

1. Configurez l'authentification OIDC entre votre dépôt GitHub et votre subscription Azure :
   - Créez une App Registration Azure
   - Ajoutez une Federated Credential pour votre repo (branche `main` et `pull_request`)
   - Ajoutez les 3 secrets GitHub : `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`
2. Créez un workflow `.github/workflows/terraform.yml` qui :
   - Sur **Pull Request** : exécute `terraform init`, `fmt -check`, `validate` et `plan`, puis poste le plan en commentaire
   - Sur **merge dans main** : exécute `terraform apply -auto-approve`
3. Vérifiez que le workflow se déclenche en ouvrant une nouvelle PR.
4. Ajoutez un second workflow `.github/workflows/security.yml` qui lance `tfsec` sur vos fichiers `.tf` à chaque PR.

---

## 🔒 Niveau 4 : Sécurité et GitOps

1. Activez **Secret Scanning avec Push Protection** sur votre dépôt. Tentez de committer une fausse clé Azure (`AZURE_CLIENT_SECRET=abc123`) et vérifiez que le push est bloqué.
2. Configurez **Dependabot** pour mettre à jour les providers Terraform et les GitHub Actions chaque semaine.
3. Créez le fichier `.github/dependabot.yml` correspondant et vérifiez qu'il apparaît dans **Insights → Dependency graph**.
4. **Bonus GitOps** : simulez un principe GitOps en créant une règle dans votre workflow qui vérifie que tout `terraform apply` ne s'exécute **que** depuis une PR mergée dans `main` (jamais depuis une branche directe).

[📖 Voir les solutions](11-solutions.md)

---

[← 07 - Fonctionnalités avancées](07-fonctionnalites-avancees.md) | [🏠 Accueil](README.md) | [09 - GitHub Actions Avancées →](09-github-actions-avance.md)
