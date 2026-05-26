# 02 - Premiers pas sur GitHub

[← 01 - Introduction](01-introduction-concepts.md) | [🏠 Accueil](README.md) | [03 - Pull Requests →](03-pull-requests-code-review.md)

---

## Objectifs de cette partie

- Créer un compte GitHub professionnel
- Configurer votre profil DevOps
- Mettre en place l'authentification SSH
- Créer votre premier dépôt IaC
- Pousser un projet existant vers GitHub

---

## Créer un compte GitHub

1. Rendez-vous sur [github.com](https://github.com)
2. Cliquez sur **Sign up**
3. Choisissez un nom d'utilisateur professionnel (visible sur toutes vos contributions)
4. Vérifiez votre email
5. Le plan **Free** est suffisant — GitHub Teams/Enterprise est géré par votre organisation

### Configurer votre profil DevOps

Un profil complet renforce votre crédibilité auprès des recruteurs et collègues :

- **Photo de profil** : photo professionnelle
- **Nom complet** : votre vrai nom
- **Bio** : `Ingénieur DevOps Azure | Terraform · AKS · GitHub Actions · CI/CD`
- **Localisation** : ville, pays
- **Site web** : LinkedIn ou portfolio
- **Entreprise** : votre employeur actuel

---

## Authentification SSH

SSH est la méthode recommandée pour interagir avec GitHub depuis le terminal : plus sécurisée, pas de mot de passe à chaque push.

### Étape 1 : Générer une clé SSH

```bash
# Algorithme ED25519 — moderne et recommandé
ssh-keygen -t ed25519 -C "votre.email@example.com"

# Acceptez le chemin par défaut (~/.ssh/id_ed25519)
# Définissez une passphrase (recommandé)
```

### Étape 2 : Démarrer l'agent SSH

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

### Étape 3 : Copier la clé publique

```bash
cat ~/.ssh/id_ed25519.pub
# Sur macOS : pbcopy < ~/.ssh/id_ed25519.pub
```

### Étape 4 : Ajouter la clé sur GitHub

1. **Settings** → **SSH and GPG keys** → **New SSH key**
2. Titre : `Laptop DevOps` (identifiez la machine)
3. Collez la clé publique
4. **Add SSH key**

### Étape 5 : Tester la connexion

```bash
ssh -T git@github.com
# Réponse attendue :
# Hi username! You've successfully authenticated...
```

---

## Créer votre premier dépôt IaC

### Via l'interface web

1. Cliquez sur **+** → **New repository**
2. Configurez :
   - **Repository name** : `infra-azure-devops`
   - **Description** : `Infrastructure Azure - Terraform + AKS`
   - **Private** (par défaut pour l'infra)
   - ✅ Add a README file
   - **.gitignore** : choisissez **Terraform**
3. **Create repository**

### Structure de dépôt recommandée

```
infra-azure-devops/
├── modules/
│   ├── aks/
│   └── network/
├── environments/
│   ├── dev/
│   └── prod/
├── kubernetes/
├── .github/
│   ├── workflows/
│   ├── CODEOWNERS
│   └── pull_request_template.md
├── .gitignore         ← généré par GitHub (Terraform)
└── README.md
```

---

## Pousser un projet local existant vers GitHub

```bash
# 1. Créer un repo VIDE sur GitHub (sans README ni .gitignore)

# 2. Dans votre projet local
cd infra-azure-devops

# 3. Lier le remote
git remote add origin git@github.com:votre-username/infra-azure-devops.git

# 4. Vérifier
git remote -v

# 5. Pousser
git branch -M main
git push -u origin main
```

---

## ⚠️ Avant de pousser : vérifiez votre .gitignore

Un credential Azure commité sur GitHub est un credential compromis — même supprimé, il reste dans l'historique Git.

```gitignore
# Terraform — généré automatiquement par GitHub
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl
*.tfplan

# Secrets Azure
.env
*.pem
service-principal.json
terraform.tfvars      # contient souvent des valeurs sensibles
```

> 💡 **Conseil** : activez dès maintenant **Secret Scanning** sur votre organisation GitHub (Settings → Code security). Il bloquera tout push contenant un pattern de credential Azure reconnu.

---

[← 01 - Introduction](01-introduction-concepts.md) | [🏠 Accueil](README.md) | [03 - Pull Requests →](03-pull-requests-code-review.md)
