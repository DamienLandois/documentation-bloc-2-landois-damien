# Présentation du Pipeline CI/CD

**Compétence :** C2.1.1 – Mettre en œuvre des environnements de déploiement et de test  
**Compétence :** C2.1.2 – Configurer le système d'intégration continue dans le cycle de développement
**Compétence :** C2.2.4. Déployer le logiciel à chaque modification de code et de façon progressive en vérifiant la performance fonctionnelle et technique auprès des utilisateurs afin de présenter une solution stable et conforme à l’attendu.

## Résumé

Le projet API Massage Backend s'appuie sur un pipeline CI/CD automatisé via **GitHub Actions** permettant d'automatiser les étapes clés du cycle de vie logiciel : installation des dépendances, vérification de la qualité du code (linting), exécution des tests unitaires et E2E, ainsi que la création et publication d'images Docker. Ce pipeline, défini dans le fichier `.github/workflows/ci.yml`, garantit la fiabilité et la reproductibilité des déploiements.

## Fichier CI/CD `.github/workflows/ci.yml`

```yaml
name: Backend CI/CD

on:
  push:
  pull_request:
    branches: ["master"]  

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8
        ports:
          - 3306:3306
        env:
          MYSQL_ROOT_PASSWORD: ${{ secrets.MYSQL_ROOT_PASSWORD }}
          MYSQL_DATABASE: ${{ secrets.MYSQL_DATABASE }}
          MYSQL_USER: ${{ secrets.MYSQL_USER }}
          MYSQL_PASSWORD: ${{ secrets.MYSQL_PASSWORD }}
        options: >-
          --health-cmd="mysqladmin ping --silent"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    env:
      DB_HOST: 127.0.0.1
      DB_PORT: 3306
      DB_USER: ${{ secrets.MYSQL_USER }}
      DB_PASSWORD: ${{ secrets.MYSQL_PASSWORD }}
      DB_NAME: ${{ secrets.MYSQL_DATABASE }}
      NODE_ENV: test
      PORT: 3000
      TZ: UTC
      JWT_SECRET: ${{ secrets.JWT_SECRET }}
      DATABASE_URL: mysql://${{ secrets.MYSQL_USER }}:${{ secrets.MYSQL_PASSWORD }}@127.0.0.1:3306/${{ secrets.MYSQL_DATABASE }}
      
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 22
      - name: Install dependencies
        run: npm ci
      - name: Generate Prisma client
        run: npx prisma generate
      - name: Run database migrations
        run: npx prisma migrate deploy
      - name: Run linter
        run: npm run lint
      - name: Run unit tests
        run: npm run test
      - name: Run e2e tests
        run: npm run test:e2e

  build-and-push:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/master'
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Setup Docker for GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Generate version number
        id: version
        run: |
          LATEST_TAG=$(git tag -l "v*.*" | sort -V | tail -n1)
          
          if [ -z "$LATEST_TAG" ]; then
            NEW_VERSION="1.0"
          else
            CURRENT_VERSION=${LATEST_TAG#v}
            MAJOR=$(echo $CURRENT_VERSION | cut -d. -f1)
            MINOR=$(echo $CURRENT_VERSION | cut -d. -f2)
            
            NEW_MINOR=$((MINOR + 1))
            NEW_VERSION="${MAJOR}.${NEW_MINOR}"
          fi
          
          echo "new_version=v${NEW_VERSION}" >> $GITHUB_OUTPUT
          echo "version_number=${NEW_VERSION}" >> $GITHUB_OUTPUT
          echo "🏷️ Nouvelle version: v${NEW_VERSION}"
      - name: Create and push new tag
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git tag ${{ steps.version.outputs.new_version }}
          git push origin ${{ steps.version.outputs.new_version }}
      - name: Set image names
        run: |
          REPO_LOWER=$(echo "${GITHUB_REPOSITORY}" | tr '[:upper:]' '[:lower:]')
          BASE_IMAGE="ghcr.io/${REPO_LOWER}"
          
          echo "IMAGE_LATEST=${BASE_IMAGE}:latest" >> $GITHUB_ENV
          echo "IMAGE_VERSIONED=${BASE_IMAGE}:${{ steps.version.outputs.version_number }}" >> $GITHUB_ENV
          echo "BASE_IMAGE=${BASE_IMAGE}" >> $GITHUB_ENV
      - name: Build Docker image
        run: |
          echo "🐳 Construction de l'image Docker..."
          docker build -t $IMAGE_LATEST -t $IMAGE_VERSIONED .
          echo "✅ Images construites:"
          echo "  - $IMAGE_LATEST"
          echo "  - $IMAGE_VERSIONED"
      - name: Push Docker images
        run: |
          echo "📤 Publication des images..."

          docker push $IMAGE_LATEST
          echo "✅ Image 'latest' publiée"

          docker push $IMAGE_VERSIONED
          echo "✅ Image 'v${{ steps.version.outputs.version_number }}' publiée"
          
          echo "🎉 Toutes les images sont disponibles:"
          echo "  - ghcr.io/$(echo ${GITHUB_REPOSITORY} | tr '[:upper:]' '[:lower:]'):latest"
          echo "  - ghcr.io/$(echo ${GITHUB_REPOSITORY} | tr '[:upper:]' '[:lower:]'):${{ steps.version.outputs.version_number }}"
```

## Découpage du pipeline

Le pipeline est organisé en **2 jobs principaux** avec une approche de **versioning automatique** :

1. **test** : Installation, linting, migrations Prisma, tests unitaires et E2E avec MySQL
2. **build-and-push** : Génération automatique de version, construction et publication d'images Docker avec double tagging (master uniquement)

## Détail des étapes

### 1. Job `test`
**But :** Valider la qualité du code et s'assurer que toutes les fonctionnalités marchent  
**Environnement :** Ubuntu + MySQL 8 + Node.js 22

**Étapes :**
- **Checkout repository** : Récupération du code source
- **Setup Node.js** : Installation de Node.js 22
- **Install dependencies** : `npm ci` pour installation déterministe
- **Generate Prisma client** : Génération du client ORM
- **Run database migrations** : `npx prisma migrate deploy` pour synchroniser la base
- **Run linter** : Vérification qualité code avec ESLint
- **Run unit tests** : Tests unitaires avec Jest (93 tests)
- **Run e2e tests** : Tests d'intégration bout en bout (22 tests)

**Services :**
- Base MySQL 8 avec health check automatique
- Variables d'environnement via GitHub Secrets (JWT_SECRET inclus)
- Connexion DATABASE_URL automatique

**Avantage :** S'assure que toutes les fonctionnalités sont validées avec une base de données réelle

### 2. Job `build-and-push`
**But :** Générer automatiquement une version sémantique, construire et publier les images Docker  
**Condition :** Tests réussis + push sur branche `master`

**Étapes :**
- **Checkout repository** : Récupération du code validé avec historique complet (`fetch-depth: 0`)
- **Setup Docker login** : Authentification au GitHub Container Registry
- **Generate version number** : Calcul automatique de la version sémantique (v1.0, v1.1, v1.2...)
- **Create and push new tag** : Création du tag Git avec la nouvelle version
- **Set image names** : Configuration des noms d'images (latest + version)
- **Build Docker image** : Construction avec double tagging simultané
- **Push Docker images** : Publication des deux images (latest + versionnée)

**Versioning automatique :**
- **Premier déploiement** : v1.0
- **Déploiements suivants** : Incrémentation automatique (v1.1, v1.2, etc.)
- **Tags Git** : Création automatique pour traçabilité

**Sécurité :** Authentification automatique via `GITHUB_TOKEN`  
**Registres :** 
- `ghcr.io/damienlandois/block-2-landois-damien-backend:latest`
- `ghcr.io/damienlandois/block-2-landois-damien-backend:1.0` (exemple)

**Avantage :** Déploiement avec versioning sémantique automatique et traçabilité complète

## Variables et Secrets

### Secrets GitHub requis
```bash
MYSQL_ROOT_PASSWORD=password_admin_mysql
MYSQL_DATABASE=nom_base_donnees  
MYSQL_USER=utilisateur_mysql
MYSQL_PASSWORD=mot_de_passe_utilisateur
JWT_SECRET=secret_jwt_pour_authentification
GITHUB_TOKEN=token_automatique_github
```

### Variables d'environnement (job test)
```bash
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=utilisateur_mysql
DB_PASSWORD=mot_de_passe_utilisateur
DB_NAME=nom_base_donnees
NODE_ENV=test
PORT=3000
TZ=UTC
JWT_SECRET=secret_jwt_pour_authentification
DATABASE_URL=mysql://user:pass@127.0.0.1:3306/database
```

## Protocole de déploiement continu avec versioning automatique

Le déploiement continu de l'API Massage repose sur un système de versioning sémantique automatique :

### Automatisation complète (GitHub Actions)
- À chaque push sur `master` avec tests réussis, le pipeline :
  . **Génère automatiquement** une nouvelle version sémantique (v1.0 → v1.1 → v1.2...)
  . **Crée un tag Git** pour la traçabilité
  . **Construit deux images Docker** : une avec tag `latest` et une avec tag de version
  . **Publie automatiquement** les deux images sur le GitHub Container Registry

### Versioning sémantique automatique
- **Premier déploiement** : Version v1.0
- **Déploiements suivants** : Incrémentation automatique du numéro mineur (v1.1, v1.2, etc.)
- **Tags Git** : Chaque version est taggée automatiquement dans le repository

### Déploiement manuel contrôlé
- Aucune image n'est automatiquement déployée sur le serveur de production
- La mise à jour nécessite une action manuelle pour assurer un contrôle final

### Étapes détaillées

#### Build & push automatique avec versioning
```bash
# Le pipeline CI génère automatiquement une version (ex: v1.0, v1.1, v1.2...)
# Puis build deux images et les push sur le registre
ghcr.io/damienlandois/block-2-landois-damien-backend:latest
ghcr.io/damienlandois/block-2-landois-damien-backend:1.0  # (exemple pour première version)
```

#### Déploiement manuel sur serveur (choix de version)
```bash
# Option A: Pull de la dernière version (latest)
docker pull ghcr.io/damienlandois/block-2-landois-damien-backend:latest

# Option B: Pull d'une version spécifique pour rollback/stabilité
docker pull ghcr.io/damienlandois/block-2-landois-damien-backend:1.2

# Tag local pour docker-compose
docker tag ghcr.io/damienlandois/block-2-landois-damien-backend:latest massage-backend:latest

# Redémarrage avec la nouvelle version
docker-compose down
docker-compose up -d --build
```

#### Contrôle final / Healthcheck
```bash
# Vérification du bon fonctionnement
curl http://localhost:3001/
docker logs massage-backend
```

## Avantages de ce fonctionnement

### Sécurité
- Aucun passage en production sans contrôle humain préalable
- Tests automatiques obligatoires avant toute image

### Qualité  
- ESLint imposé sur tout le code
- Migrations Prisma automatiques en CI
- 22 tests E2E obligatoires
- 93 tests unitaires systématiques

### Traçabilité
- Chaque image liée à un commit ET une version sémantique
- Historique complet sur GitHub Container Registry
- Tags Git automatiques pour navigation dans les versions

### Versioning automatique
- Génération automatique de versions sémantiques (v1.0, v1.1, v1.2...)
- Tags Git créés automatiquement pour traçabilité complète
- Double tagging des images Docker (latest + version spécifique)
- Possibilité de rollback vers une version antérieure

### Simplicité
- 2 jobs seulement, faciles à comprendre
- Versioning complètement automatisé
- Déploiement Docker standard avec choix de version

## Système de versioning automatique

### Principe de fonctionnement
Le pipeline utilise un système de **versioning sémantique automatique** basé sur les tags Git :

. **Détection de la dernière version** : `git tag -l "v*.*" | sort -V | tail -n1`
. **Calcul de la nouvelle version** :
   - Si aucun tag existe : **v1.0**
   - Sinon : incrémentation du numéro mineur (v1.0 → v1.1 → v1.2...)
. **Création du tag Git** : `git tag v1.x && git push origin v1.x`
. **Double tagging Docker** : 
   - `ghcr.io/repo:latest` (toujours la dernière version)
   - `ghcr.io/repo:1.x` (version spécifique pour rollback)

### Avantages du système
- **Traçabilité complète** : Chaque déploiement a un numéro de version unique
- **Rollback facilité** : Possibilité de revenir à une version antérieure
- **Zéro intervention manuelle** : Versioning entièrement automatisé
- **Compatibilité Git** : Tags visibles dans l'interface GitHub

## Exemple de cycle de vie d'une modification

. **Développement** : Un développeur pousse du code sur une branche
. **Tests automatiques** : Le pipeline s'exécute (install → lint → migrations → tests → tests E2E)
. **Validation** : Fusion sur `master` après validation
. **Versioning automatique** : Le pipeline génère une nouvelle version (ex: v1.3)
. **Build automatique** : Construction de deux images Docker (latest + v1.3)
. **Tag Git** : Création automatique du tag v1.3 dans le repository
. **Publication** : Push des deux images sur GitHub Container Registry
. **Déploiement manuel** : Admin choisit latest ou une version spécifique et redémarre
. **Contrôle** : Vérification du bon fonctionnement

## Métriques actuelles

- **Fréquence de build** : À chaque push sur master
- **Durée du pipeline** : ~8-13 minutes (test + build + versioning + push)
- **Taux de succès des tests** : 100% (115 tests : 93 unitaires + 22 E2E)
- **Couverture** : Tests unitaires + E2E + migrations automatiques
- **Sécurité** : Linting + audit automatique des dépendances + JWT en CI
- **Versioning** : Automatique avec tags Git (v1.0, v1.1, v1.2...)
- **Images Docker** : Double tagging (latest + version) pour flexibilité maximale

