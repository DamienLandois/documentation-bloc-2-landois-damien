# Cahier de recette (Fonctionnel)

## Vue d'ensemble

Ce projet contient deux types de tests conformes au cahier de recettes :

### **Tests Unitaires** (`.spec.ts`)
- Tests isolés de chaque service/contrôleur
- Exécution : `npm run test`
- Couverture : `npm run test:cov`

### **Tests Fonctionnels et de Sécurité** (`.e2e-spec.ts`)
- Tests de workflows complets
- Validation des règles métier
- Tests d'intégration bout en bout
- Protection contre les injections
- Validation des accès
- Sécurité des uploads

## Commandes de Test

### Exécuter tous les tests fonctionnels
```bash
docker compose exec backend npm run test:e2e
```

### Exécuter seulement les tests de workflow
```bash
docker compose exec backend npm run test:functional
```

### Exécuter seulement les tests de sécurité
```bash
docker compose exec backend npm run test:security
```

### Mode watch pour développement
```bash
docker compose exec backend npm run test:e2e:watch
```

## Structure des Tests Fonctionnels

### app.e2e-spec.ts (3 tests)
Tests de base de l'application :
- Vérification du module principal
- Test du endpoint racine (GET /)
- Validation de la configuration de base

### booking-workflow.e2e-spec.ts (11 tests)
Couvre les scénarios test-fonctionnel-1 à test-fonctionnel-5 du cahier de recettes :

#### test-fonctionnel-1: Workflow complet de réservation
- Création d'utilisateur admin
- Création d'utilisateur standard  
- Création de massage
- Création de créneau horaire
- Consultation des massages disponibles
- Consultation des créneaux disponibles
- Création de réservation
- Consultation des réservations utilisateur
- Consultation admin de toutes les réservations

#### test-fonctionnel-2: Gestion des conflits horaires
- Rejet des réservations en conflit direct
- Rejet des réservations avec pause insuffisante (< 30 min)
- Acceptation des réservations avec pause suffisante (≥ 30 min)

#### test-fonctionnel-3: Gestion des permissions
- Restriction des endpoints admin aux utilisateurs standards
- Restriction d'accès sans authentification

#### test-fonctionnel-4: Validation des données
- Validation des IDs de massage/créneau
- Validation des limites temporelles
- Validation des formats email/mot de passe

#### test-fonctionnel-5: Cycle de vie des réservations
- Création → Modification → Annulation → Suppression

### rate-limiting.e2e-spec.ts (4 tests)
Tests de protection contre le rate limiting :
- test-rate-limiting-1: Protection par limitation de taux
- Validation des en-têtes de rate limiting
- Gestion des erreurs 429 (Too Many Requests)  
- Réinitialisation automatique des compteurs

### security.e2e-spec.ts (4 tests)
Tests de sécurité applicative

#### test-security-1: Protection contre injection SQL
- Tentatives d'injection dans le login
- Sanitisation des caractères spéciaux

#### test-security-2: Validation stricte des données
- Validation des formats email
- Validation des mots de passe
- Validation des champs numériques

#### test-security-3: Contrôle d'accès
- Prévention de l'élévation de privilèges
- Isolation des données utilisateur

#### test-security-4: Protection des uploads
- Rejet des fichiers non-image
- Limitation de taille des fichiers
- Acceptation des images valides

#### test-security-5: Protection contre attaques temporelles
- Temps de réponse constants pour le login

## Prérequis pour les Tests

### Base de données de test
Les tests utilisent la même base de données que le développement mais :
- Nettoient automatiquement les données créer pour les tests avant/après chaque test
- Créent leurs propres jeux de données

### Environnement Docker
```bash
# Démarrer l'environnement
docker compose up -d --build

# Exécuter les tests
docker compose exec backend npm run test:e2e
```

## Résultats Attendus

### Critères de Succès
- **100%** des tests fonctionnels passent
- **0** vulnérabilité de sécurité détectée
- **Temps de réponse** < 1000ms pour les workflows complets
- **Couverture** des règles métier principales

### Métriques Validées
- Gestion des conflits horaires
- Règle des 30 minutes de pause entre chaque réservation.
- Contrôles d'accès par rôle
- Validation des données d'entrée
- Sécurité des uploads

## Intégration CI/CD

### Pipeline de Tests Automatisés
-Dans le fichier ci.yml
```yaml
- name: Run Unit Tests
  run: npm run test

- name: Run Functional Tests  
  run: npm run test:e2e

- name: Run lint
  run: npm run lint
```

## Maintenance des Tests

### Mise à jour après changements
Si vous modifiez l'API :
1. Mettre à jour le cahier de recettes
2. Adapter les tests fonctionnels correspondants
3. Vérifier que tous les tests passent
4. Mettre à jour cette documentation

### Ajout de nouveaux tests
Pour chaque nouvelle fonctionnalité :
1. Ajouter les scénarios au cahier de recettes
2. Créer les tests fonctionnels correspondants
3. Inclure les tests de sécurité si applicable
4. Mettre à jour cette documentation

---
