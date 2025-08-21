# Plan de Correction des Bogues

## 1. Procédure de traitement des bogues (Jira)

### Signalement
- Création d'un ticket Jira "Bug" (avec message discord si criticité élévé)
- Informations requises : titre, description, contexte, logs/captures, environnement
- Types de tickets : Bug

### Qualification
- Sévérité évaluée par le lead dev ou par le product owner, ticket assigné à un développeur
- Priorités Jira : Highest, High, Medium, Low

### Analyse
- Lecture des logs Winston (`logs/error-*.log`)
- Reproduction du bug dans un container en local avec `docker compose up -d --build`
- Documentation de la cause racine dans le ticket

### Correction
- Correction sur la branche feature actuelle
- Convention de commit : `fix: correction authentification admin` ou `update: amélioration validation`
- Tests ajoutés/modifiés pour couvrir le cas
- Code review obligatoire par 2 développeur avant merge

### Validation
- Passage en "À tester", QA ou PO valide (tests automatisés/manuels)
- Tests E2E automatisés : `docker compose exec backend npm run test:e2e` (22 tests),
              suivie de : `docker compose exec backend npm run test` (93 tests),
              suivie de : `docker compose exec backend npm run lint`
- Si validé : ticket passé en "Corrigé" puis "Fermé"

---

## Classification des Bogues

| Criticité | Délai espéré| Exemples |
|-----------|-------------|----------|
| **CRITIQUE** | < 4h | Crash API, faille sécurité, corruption données |
| **MAJEUR** | < 24h | Authentification KO, réservation impossible |
| **MINEUR** | < 1 semaine | Message d'erreur incorrect, validation trop stricte |
| **AMÉLIORATION** | Next sprint | Performance, ergonomie, logs |

---

## Exemples de Tickets Corrigés

### Ticket RMA-101 - Erreur authentification admin
**État :** Corrigé  
**Résolution :** Fix publié

### Ticket RMA-102 - Code de retour incorrect pour login
**État :** Corrigé  
**Criticité :** MINEUR  
**Problème :** L'endpoint POST `/auth/login` retournait un code HTTP 201 (Created) au lieu de 200 (OK) lors d'une authentification réussie  
**Cause racine :** De base nest renvoi un code 201 pour ses requete POST
**Solution appliquée :**
Ajout d'un décorateur pour HttpCode pour le code 200
```typescript
// AVANT (incorrect)
@Post('login')
async login(@Body() loginDto: LoginDto) {
  return this.authService.login(loginDto);
  // Retournait automatiquement 201
}

// APRÈS (corrigé)
@Post('login')
@HttpCode(200)
async login(@Body() loginDto: LoginDto) {
  return this.authService.login(loginDto);
  // Retourne maintenant 200 comme attendu
}
``` 
**Validation :** Tests E2E passent avec le bon code de retour

---
