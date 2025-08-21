# Architecture du projet
**Compétence** : C2.2.3 – Développer en veillant à l'évolutivité

Le backend repose sur le framework NestJS avec TypeScript, utilisant Prisma ORM pour l'accès aux données MySQL et une architecture modulaire sécurisée.

## Organisation générale du dépôt

```
coverage/ — Rapports de couverture de tests
dist/ — Fichiers TypeScript compilés (build)
logs/ — Logs applicatifs Winston (rotation quotidienne)
node_modules/ — Dépendances Node.js (exclu du dépôt)
prisma/ — Schémas et migrations de base de données
src/ — Code source principal de l'API
test/ — Tests End-to-End et unitaires
uploads/ — Fichiers uploadés (images massages)
.env — Variables d'environnement développement
.env.production — Variables d'environnement production
docker-compose.yml — Orchestration des conteneurs
Dockerfile — Image Docker de l'application
eslint.config.mjs — Configuration ESLint
nest-cli.json — Configuration NestJS CLI
package.json — Dépendances et scripts npm
README.md — Documentation principale
tsconfig.json — Configuration TypeScript
```

## Détail du dossier src/

### 1. **Modules métiers principaux**

#### `auth/` — Module d'authentification et autorisation
```
auth.controller.ts — Endpoints login/register/logout
auth.service.ts — Logique métier authentification
auth.module.ts — Module NestJS
decorators/ — Décorateurs personnalisés (@Public, @Roles)
guards/ — Guards de protection (@AuthGuard, @RolesGuard)
interfaces/ — Interfaces TypeScript (JwtPayload, etc.)
strategies/ — Stratégies Passport (JWT, Local)
*.spec.ts — Tests unitaires
```

#### `user/` — Gestion des utilisateurs
```
user.controller.ts — CRUD utilisateurs
user.service.ts — Logique métier utilisateurs
user.module.ts — Module NestJS
dto/ — Data Transfer Objects (CreateUserDto, UpdateUserDto)
enums/ — Énumérations (UserRole: CLIENT, ADMIN)
*.spec.ts — Tests unitaires
```

#### `massage/` — Gestion des services de massage
```
massage.controller.ts — CRUD massages et services
massage.service.ts — Logique métier massages
massage.module.ts — Module NestJS
decorators/ — Décorateurs spécifiques (@ValidateMassage)
dto/ — DTOs (CreateMassageDto, UpdateMassageDto)
interceptors/ — Intercepteurs (logging, transformation)
*.spec.ts — Tests unitaires
```

#### `planning/` — Système de réservation et planning
```
planning.controller.ts — Gestion créneaux et réservations
planning.service.ts — Logique métier planning
planning.module.ts — Module NestJS
dto/ — DTOs (CreateBookingDto, UpdateTimeSlotDto)
*.spec.ts — Tests unitaires
```

#### `email/` — Service d'envoi d'emails
```
email.service.ts — Service SMTP (confirmations, notifications)
email.module.ts — Module NestJS
templates/ — Templates emails HTML/text
*.spec.ts — Tests unitaires
```

### 2. **Modules techniques transversaux**

#### `common/` — Utilitaires et services partagés
```
logger/ — Service de logs Winston personnalisé
logger.service.ts — Logs sécurisés, rotation, audit
security/ — Middleware et utilitaires sécurité
```

#### `prisma/` — Accès aux données
```
prisma.service.ts — Service Prisma Client
prisma.module.ts — Module NestJS pour injection
```

### 3. **Fichiers racine du src/**
```
main.ts — Point d'entrée, configuration app (CORS, Helmet, Swagger)
app.module.ts — Module racine, orchestration de tous les modules
```

## Base de données (Prisma)

### Structure du dossier `prisma/`
```
schema.prisma — Définition du schéma de données
migrations/ — Historique des migrations
  migration_lock.toml — Verrou de migration
  20250808120105_init/ — Migration initiale
  20250816090706_add_user_roles/ — Ajout système de rôles
  20250816191649_add_position_to_massage/ — Position massages
  20250816201952_add_planning_system/ — Système planning
  20250817120000_make_image_nullable/ — Images optionnelles
  20250818191102_rename_massage_title_to_name/ — Rename colonnes
  20250818204153_add_booking_times/ — Heures réservation
  20250818221516_remove_unique_constraint_timeslot_massage/ — Contraintes
```

### Entités principales
- **User** : Utilisateurs (clients/admin) avec roles
- **Massage** : Services de massage avec prix/durée
- **TimeSlot** : Créneaux horaires disponibles
- **Booking** : Réservations clients
- **Planning** : Organisation temporelle

## Dossier test/

### Structure organisée par type de test
```
app.e2e-spec.ts — Test de l'endpoint racine
booking-workflow.e2e-spec.ts — Tests fonctionnels workflow complet
security.e2e-spec.ts — Tests de sécurité
rate-limiting.e2e-spec.ts — Tests limitation débit (2 tests)
jest-e2e.json — Configuration Jest E2E
jest-setup.ts — Setup global des tests
utils/
  test-logger.service.ts — Logger pour tests
```

### Types de tests couverts
- **Fonctionnels** : Workflow de réservation, gestion conflits
- **Sécurité** : Injection SQL, authentification, autorisation
- **Performance** : Rate limiting, charge
- **Intégration** : API endpoints, base de données

**Total : 115 tests automatisés (22 E2E + 93 unitaires)**

## Configuration et outils

### **Variables d'environnement**
- `.env` — Développement (NODE_ENV=development)
- `.env.production` — Production (NODE_ENV=production)

### **Qualité de code**
- `eslint.config.mjs` — Linting TypeScript/NestJS
- `.prettierrc` — Formatage code
- `tsconfig.json` — Configuration TypeScript stricte

### **DevOps et déploiement**
- `docker-compose.yml` — MySQL + Backend + volumes
- `Dockerfile` — Image multi-stage optimisée
- Scripts npm adaptés par environnement

## Pourquoi cette architecture ?

### ** Sécurité renforcée**
- **Authentification JWT** avec refresh tokens
- **Hachage bcrypt** pour les mots de passe
- **Guards NestJS** pour contrôle d'accès
- **Validation stricte** avec class-validator
- **Rate limiting** contre brute force
- **Logs d'audit** Winston avec sanitisation

### ** Performance et évolutivité**
- **Architecture modulaire** NestJS découplée
- **ORM Prisma** avec requêtes optimisées
- **Logs structurés** avec rotation quotidienne
- **Tests automatisés** garantissant la régression

### ** Maintenabilité**
- **Separation of Concerns** : chaque module a sa responsabilité
- **Injection de dépendances** facilitant les tests
- **DTOs typés** pour validation automatique
- **Documentation auto-générée** avec Swagger
- **Migration de BDD** versionnées avec Prisma

### ** Production Ready**
- **Configuration par environnement** (dev/prod)
- **Containerisation Docker** pour déploiement
- **Monitoring** avec logs centralisés
- **Variables d'environnement** sécurisées
- **Build optimisé** TypeScript compilé