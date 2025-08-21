# Documentation

**Compétence** : C2.4.1 – Rédiger la documentation technique d'exploitation du logiciel

Cette documentation comprend :
- **Le manuel de déploiement** avec versioning automatique
- **Le manuel d'utilisation** des endpoints API
- **Le manuel de mise à jour** et rollback

---

## Manuel de Déploiement

### Prérequis d'installation

**Versions requises :**
- `Node.js 22.16.0+` 
- `Docker Engine 20.x+`
- `Docker Compose 2.x+`

**Installation des dépendances :**
```bash
npm install
```

**Configuration des variables d'environnement :**

Créer un fichier `.env` à la racine (prendre exemple sur le fichier `.env.production`) :


**Sécurité** : Utiliser des mots de passe complexes et des secrets JWT uniques en production.

### Déploiement avec image Docker pré-construite

**Option A : Déploiement avec dernière version (recommandé) :**
```bash
# Pull de la dernière image depuis GitHub Container Registry
docker pull ghcr.io/damienlandois/block-2-landois-damien-backend:latest

# Tag local pour docker-compose
docker tag ghcr.io/damienlandois/block-2-landois-damien-backend:latest \
  massage-backend:latest

# Démarrage de la stack complète
docker-compose up -d --build
```

**Option B : Déploiement avec version spécifique :**
```bash
# Pull d'une version spécifique pour stabilité
docker pull ghcr.io/damienlandois/block-2-landois-damien-backend:1.2

# Tag local avec la version choisie
docker tag ghcr.io/damienlandois/block-2-landois-damien-backend:1.2 \
  massage-backend:latest

# Démarrage de la stack
docker-compose up -d
```

**Option C : Build local depuis les sources :**
```bash
# Pour développement ou personnalisation
docker-compose up -d --build
```

**Services déployés automatiquement :**
- **MySQL 8** : Base de données relationnelle
- **API NestJS** : Backend avec authentification JWT (port 3001)
- **Volumes persistants** : Données et logs sauvegardés

**Vérification du démarrage :**
```bash
docker-compose logs backend | Select-String "successfully started"
```

**Message attendu :** `LOG [NestApplication] Nest application successfully started`

### Initialisation de la base de données

**Application des migrations Prisma :**
```bash
docker-compose exec backend npx prisma migrate deploy
docker-compose exec backend npx prisma generate
```

**Vérification de la structure :**
```bash
docker-compose exec backend npx prisma studio
```

### Création d'un administrateur

```bash
curl -X POST http://localhost:3001/user/admin \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@raphaelle-massage.com","password":"motdepasse123","firstname":"Admin","name":"System"}'
```

### Validation du déploiement

**Tests de validation complets :**
```bash
# Tests unitaires (93 tests)
docker-compose exec backend npm run test

# Tests d'intégration E2E (22 tests)
docker-compose exec backend npm run test:e2e

# Vérification du linting
docker-compose exec backend npm run lint
```

**Test de connectivité API :**
```bash
curl http://localhost:3001/massages
```

**Résultats attendus :** 
- 93/93 tests unitaires passants
- 22/22 tests E2E validés  
- 0 erreur ESLint
- API répond en JSON

---

## Manuel d'Utilisation

### Accès à l'API

**URL de base en développement :** `http://localhost:3001`

**URL de Swagger**
`http://localhost:3001/api`

**Endpoints principaux :**

#### Endpoints Publics
- `GET /massages` - Liste des massages disponibles
- `POST /auth/login` - Authentification utilisateur
- `POST /user` - Inscription nouveau client

#### Endpoints Authentifiés
- `GET /planning/creneaux` - Créneaux horaires disponibles
- `POST /planning/reservations` - Créer une réservation
- `GET /planning/mes-rendez-vous` - Mes réservations
- `DELETE /planning/reservations/:id/annuler` - Annuler réservation

#### Endpoints Administrateur  
- `POST /user/admin` - Créer un administrateur
- `GET /planning/reservations` - Toutes les réservations
- `POST /massages` - Créer un nouveau massage
- `POST /planning/creneaux` - Créer nouveaux créneaux

### Authentification JWT

**Connexion utilisateur :**
```bash
curl -X POST http://localhost:3001/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email":"user@example.com",
    "password":"motdepassE123!"
  }'
```

**Utilisation du token d'authentification :**
```bash
curl -H "Authorization: Bearer <jwt_token>" \
     http://localhost:3001/planning/mes-rendez-vous
```

**Gestion des rôles :**
- `CLIENT` : Gestion de ses propres réservations et de son compte
- `ADMIN` : Accès complet à toutes les fonctionnalités

### Gestion des erreurs et codes de statut

**Codes de réponse standards :**
- **200** : Succès de l'opération
- **201** : Ressource créée avec succès
- **204** : Ressource supprimée avec succès
- **400** : Données invalides ou malformées
- **401** : Token d'authentification manquant ou invalide
- **403** : Accès interdit (rôle insuffisant)
- **404** : Ressource non trouvée
- **500** : Erreur interne du serveur

**Format des erreurs :**
```json
{
  "statusCode": 400,
  "message": "Description de l'erreur",
  "error": "Bad Request"
}
```

### Surveillance et monitoring

**Logs en temps réel :**
```bash
# Tous les services
docker-compose logs -f

# Backend uniquement
docker-compose logs -f backend

# Base de données
docker-compose logs -f db
```

**Logs Winston en temps réel (depuis le container) :**
```bash
#voir tous les fichiers de logs:
docker-compose exec backend ls -la logs/

# Logs généraux du jour
docker-compose exec backend tail -f logs/combined-2025-08-21.log #changer la date

# Logs d'erreurs uniquement
docker-compose exec backend tail -f logs/error-2025-08-21.log #changer la date

# Logs de sécurité
docker-compose exec backend tail -f logs/security-2025-08-21.log #changer la date
```

**Fichiers de logs persistants :**
- `logs/combined-YYYY-MM-DD.log` : Logs généraux
- `logs/error-YYYY-MM-DD.log` : Erreurs uniquement  
- `logs/security-YYYY-MM-DD.log` : Événements sécuritaires

**Tests de santé de l'application :**
```bash
# Vérification API disponible
curl http://localhost:3001/ -> renvoi 404.

# Statut des containers
docker-compose ps

# Utilisation des ressources
docker stats
```

---

## Manuel de Mise à Jour

### Sauvegarde préventive

**Sauvegarde base de données :**
```bash
# Création du backup
docker-compose exec db mysqldump -u root -p${DB_ROOT_PASSWORD} \
  raphaelle-massage > backup-$(date +%Y%m%d-%H%M).sql

# Vérification du backup
ls -la backup-*.sql
```

**Arrêt sécurisé des services :**
```bash
docker-compose down --timeout 30
```

### Mise à jour via images Docker versionnées

**Option A : Mise à jour automatique (latest) :**
```bash
# Pull de la dernière image depuis GitHub Container Registry
docker pull ghcr.io/damienlandois/block-2-landois-damien-backend:latest

# Tag local pour docker-compose  
docker tag ghcr.io/damienlandois/block-2-landois-damien-backend:latest \
  massage-backend:latest

# Redémarrage avec nouvelle version
docker-compose up -d --no-deps backend
```

**Option B : Mise à jour vers version spécifique :**
```bash
# Pull d'une version particulière
docker pull ghcr.io/damienlandois/block-2-landois-damien-backend:1.2

# Utilisation de cette version
docker tag ghcr.io/damienlandois/block-2-landois-damien-backend:1.2 \
  massage-backend:latest
  
docker-compose up -d --no-deps backend
```

### Application des migrations et vérifications

**Vérification des migrations en attente :**
```bash
docker-compose exec backend npx prisma migrate status
```

**Application des nouvelles migrations :**
```bash
docker-compose exec backend npx prisma migrate deploy
```

**Régénération du client Prisma :**
```bash
docker-compose exec backend npx prisma generate
```

### Tests de validation post-mise à jour

**Suite complète de tests :**
```bash
# Tests unitaires (93 tests)
docker-compose exec backend npm run test

# Tests d'intégration E2E (22 tests)  
docker-compose exec backend npm run test:e2e

# Vérification qualité code
docker-compose exec backend npm run lint
```
---