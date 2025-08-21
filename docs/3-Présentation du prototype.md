# Présentation du prototype

## Contexte de la compétence

**C2.2.1** : Concevoir un prototype de l'application en tenant compte des spécificités ergonomiques et des équipements ciblés (ex : web, mobile…) afin de répondre aux fonctionnalités attendues et aux exigences en termes de sécurité.

## Présentation du prototype réalisé

### **Prototype développé : API REST Backend**
- **Type** : API REST backend sécurisée
- **Framework** : NestJS avec TypeScript
- **Base de données** : MySQL avec Prisma ORM
- **Architecture** : Modulaire et évolutive
- **Sécurité** : Authentification JWT + protection OWASP Top 10

### **Équipements ciblés**
- **Serveurs web** : Compatible Linux/Windows/Docker
- **Clients** : Applications web, mobiles, SPA React/Vue
- **Interfaces** : API REST JSON + documentation Swagger
- **Environnements** : Développement, test, production

## Architecture logicielle structurée

### **Architecture modulaire NestJS**

```
src/
├── auth/           # Module authentification JWT
├── user/           # Gestion utilisateurs et rôles
├── massage/        # Catalogue services de massage
├── planning/       # Système de réservation
├── email/          # Notifications SMTP
├── common/         # Services transversaux
│   ├── logger/     # Logs Winston sécurisés
│   └── security/   # Middleware sécurité
└── prisma/         # Accès données ORM
```

### **Patterns de développement implémentés**

#### **Dependency Injection (IoC)**
```typescript
@Controller('auth')
export class AuthController {
  constructor(private readonly auth: AuthService) {}
}
```

#### **Repository Pattern via Prisma**
```typescript
@Injectable()
export class UserService {
  constructor(private prisma: PrismaService) {}
  
  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }
}
```

#### **Guard Pattern pour sécurité**
```typescript
  @UseGuards(JwtAuthGuard, RolesGuard)
  @AdminOnly()
  create(
    @Body() createMassageDto: CreateMassageDto,
    @UploadedFile(
      new ParseFilePipeBuilder()
        .addFileTypeValidator({
          fileType: /(jpg|jpeg|png|gif)$/,
        })
        .addMaxSizeValidator({
          maxSize: 5 * 1024 * 1024,
        })
        .build({
          fileIsRequired: false,
          errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY,
        }),
    )
    image?: Express.Multer.File,
  ) {
    const imageName = image ? image.filename : null;
    return this.massageService.create(createMassageDto, imageName);
  }
```

#### **DTO Pattern pour validation**
```typescript

export class CreateMassageDto {
  @ApiProperty({
    description: 'Nom du massage',
    example: 'Massage relaxant',
  })
  @IsString()
  name: string;

  @ApiProperty({
    description: 'Description du massage',
    example: 'Un massage relaxant pour détendre les muscles',
  })
  @IsString()
  description: string;

  @ApiProperty({
    description: 'Durée du massage en minutes',
    example: 60,
    minimum: 1,
    maximum: 300,
  })
  @IsNumber()
  @Min(1)
  @Max(300)
  @Transform(({ value }) => parseInt(value as string, 10))
  duration: number;

  @ApiProperty({
    description: 'Prix du massage en euros',
    example: 50,
    minimum: 0,
  })
  @IsNumber()
  @Min(0)
  @Transform(({ value }) => parseFloat(value as string))
  price: number;

  @ApiProperty({
    description: 'Position du massage dans la liste',
    example: 1,
    minimum: 1,
  })
  @IsNumber()
  @Min(1)
  @Transform(({ value }) => parseInt(value as string, 10))
  position: number;

  @ApiProperty({
    type: 'string',
    format: 'binary',
    description: 'Image du massage (optionnel)',
    required: false,
  })
  @IsOptional()
  @Transform(({ value }): unknown => {
    if (value === '' || value === null) {
      return undefined;
    }
    return value;
  })
  image?: any;
}

```

## Framework et paradigmes utilisés

### **Framework principal : NestJS**
- **Avantages** : Architecture modulaire, TypeScript natif, écosystème riche
- **Paradigmes** : MVC, Dependency Injection, Decorators
- **Compatibilité** : Express.js sous-jacent pour performance

### **ORM : Prisma**
- **Type-safety** : Génération automatique des types TypeScript
- **Migrations** : Versioning de schéma automatisé
- **Performance** : Requêtes optimisées et cache intégré

### **Paradigmes de développement**

#### **Programmation orientée objet (POO) - Architecture principale**
```typescript
export class AuthService {
  private readonly logger = new Logger(AuthService.name);
  
  async validateUser(email: string, password: string): Promise<User | null> {
    // Logique métier encapsulée dans la classe
  }
}
```

#### **Éléments fonctionnels - Transformations et validations**
```typescript
// Fonctions pures pour transformation de données
@Transform(({ value }) => value.trim())
@Transform(({ value }) => parseInt(value as string, 10))
email: string;

// Composition de validateurs
@IsEmail()
@IsNotEmpty()
@Min(0)
price: number;
```

## Fonctionnalités principales implémentées

### **Système d'authentification complet**
- Inscription utilisateur avec validation
- Connexion JWT avec refresh token
- Gestion des rôles (CLIENT/ADMIN)
- Protection des endpoints sensibles

### **Gestion des services de massage**
- CRUD complet des massages
- Upload et gestion d'images
- Prix et durées configurables
- Ordering et positionnement

### **Système de réservation intelligent**
- Création de créneaux horaires
- Réservation avec gestion des conflits
- Validation automatique des disponibilités
- Annulation et modification

### **Notifications et communications**
- Emails automatiques de confirmation
- Templates HTML personnalisés
- Service SMTP configurable

## Exigences de sécurité satisfaites

### **Authentification et autorisation**
```typescript
// JWT sécurisé avec expiration courte
@UseGuards(JwtAuthGuard)
async getProfile(@Req() req) {
  return req.user; // Utilisateur authentifié
}

// Contrôle d'accès basé sur les rôles
@Roles(UserRole.ADMIN)
@UseGuards(RolesGuard)
async adminEndpoint() { ... }
```

### **Protection OWASP Top 10**
. **A01 - Contrôle d'accès** : Guards NestJS + JWT
. **A02 - Cryptographie** : bcrypt niveau 11 + HTTPS
. **A03 - Injection** : Prisma ORM + ValidationPipe
. **A04 - Conception** : Architecture modulaire sécurisée
. **A05 - Configuration** : Variables d'environnement
. **A06 - Composants** : Dépendances à jour
. **A07 - Identification** : Rate limiting + sessions courtes
. **A08 - Intégrité** : Validation stricte des données
. **A09 - Logging** : Winston avec audit trail
. **A10 - SSRF** : Pas de requêtes externes user-controlled

### **Validation et sanitisation**
```typescript
// Validation automatique des entrées
@Post('register')
async register(@Body() createUserDto: CreateUserDto) {
  // Auto-validation via class-validator
  // Sanitisation automatique
}
```

### **Logs de sécurité**
```typescript
// Traçabilité complète
this.logger.logSecurityEvent('FAILED_LOGIN_ATTEMPT', {
  ip: req.ip,
  userAgent: req.headers['user-agent'],
  email: loginDto.email
});
```

## Composants d'interface fonctionnels

### **API REST endpoints documentés**

#### **Authentification** (`/auth`)
- `POST /auth/login` - Connexion utilisateur
- `POST /auth/logout` - Déconnexion
- `POST /auth/register` - Inscription

#### **Utilisateurs** (`/user`)
- `GET /user` - Liste utilisateurs (Admin)
- `GET /user/:id` - Détail utilisateur
- `POST /user` - Création utilisateur
- `PUT /user/:id` - Modification
- `DELETE /user/:id` - Suppression

#### **Massages** (`/massages`)
- `GET /massages` - Catalogue complet
- `GET /massages/:id` - Détail massage
- `POST /massages` - Création (Admin)
- `PUT /massages/:id` - Modification (Admin)
- `DELETE /massages/:id` - Suppression (Admin)

#### **Planning** (`/planning`)
- `GET /planning/timeslots` - Créneaux disponibles
- `POST /planning/timeslots` - Création créneau (Admin)
- `POST /planning/bookings` - Nouvelle réservation
- `GET /planning/bookings/me` - Mes réservations
- `PUT /planning/bookings/:id` - Modification réservation
- `DELETE /planning/bookings/:id` - Annulation

### **Documentation Swagger intégrée**
```typescript
@ApiTags('Authentification')
@ApiOperation({ summary: 'Connexion utilisateur' })
@ApiResponse({ status: 200, description: 'Connexion réussie' })
@ApiResponse({ status: 401, description: 'Identifiants invalides' })
```

### **Gestion d'erreurs standardisée**
```typescript
// Réponses HTTP cohérentes
{
  "statusCode": 400,
  "message": "Validation failed",
  "error": "Bad Request"
}
```

## Tests et validation

### **22 tests automatisés**
- **1 test** app.e2e-spec.ts (endpoint racine)
- **9 tests** booking-workflow.e2e-spec.ts (workflow complet)
- **10 tests** security.e2e-spec.ts (sécurité OWASP)
- **2 tests** rate-limiting.e2e-spec.ts (performance)

### **Couverture fonctionnelle**
```bash
# Tests de workflow complet
npm run test:functional

# Tests de sécurité
npm run test:security

# Tests de performance
npm run test:e2e
```

