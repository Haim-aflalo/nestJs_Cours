# COURS NESTJS COMPLET
## Sections 11-13 : Performance, Architecture, Déploiement

---

## SECTION 11 : PERFORMANCE ET SCALABILITÉ

### 11.1 Fastify vs Express

NestJS fonctionne par défaut avec Express.

**Installer Fastify :**
```
npm install @nestjs/platform-fastify fastify
```

**Utiliser Fastify :**
```typescript
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  await app.listen(3000);
}
bootstrap();
```

**Avantages de Fastify :**
- 2-3x plus rapide qu'Express
- Consommation mémoire réduite
- Validation JSON intégrée

### 11.2 Cache avec Redis

**Installation :**
```
npm install @nestjs/cache-manager cache-manager redis
```

**Configuration :**
```typescript
import { CacheModule } from '@nestjs/cache-manager';
import * as redisStore from 'cache-manager-redis-store';

@Module({
  imports: [
    CacheModule.register({
      isGlobal: true,
      store: redisStore,
      host: 'localhost',
      port: 6379,
      ttl: 5 * 60, // 5 minutes
    }),
  ],
})
export class AppModule {}
```

**Utilisation :**
```typescript
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class UsersService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async findAll() {
    // Vérifier le cache
    const cached = await this.cacheManager.get('users:all');
    if (cached) {
      return cached;
    }

    // Récupérer de la base de données
    const users = await this.usersRepository.findAll();

    // Stocker en cache (5 minutes)
    await this.cacheManager.set('users:all', users, 5 * 60 * 1000);

    return users;
  }

  async invalidateCache() {
    await this.cacheManager.del('users:all');
  }
}
```

### 11.3 Rate Limiting

**Installation :**
```
npm install @nestjs/throttler
```

**Configuration :**
```typescript
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        ttl: 60000,  // 1 minute
        limit: 10,   // 10 requêtes max
      },
    ]),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
export class AppModule {}
```

Maintenant, max 10 requêtes par minute par IP.

### 11.4 Queues avec Bull

**Installation :**
```
npm install @nestjs/bull bull
```

**Créer une queue :**
```typescript
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';

@Module({
  imports: [
    BullModule.forRoot({
      redis: { host: 'localhost', port: 6379 },
    }),
    BullModule.registerQueue({ name: 'email' }),
  ],
})
export class AppModule {}
```

**Producer (ajouter un job) :**
```typescript
@Injectable()
export class EmailService {
  constructor(@InjectQueue('email') private emailQueue: Queue) {}

  async sendWelcomeEmail(email: string) {
    await this.emailQueue.add(
      'send-welcome',
      { email },
      { delay: 5000 }, // Attendre 5 secondes
    );
  }
}
```

**Consumer (traiter les jobs) :**
```typescript
@Processor('email')
export class EmailProcessor {
  @Process('send-welcome')
  async sendWelcome(job: Job<{ email: string }>) {
    console.log(`Sending email to ${job.data.email}`);
    // Logique d'envoi d'email
    return { success: true };
  }
}
```

---

## SECTION 12 : ARCHITECTURE PROPRE ET MAINTENABLE

### 12.1 Organisation des dossiers

```
src/
├── modules/
│   ├── users/
│   │   ├── controllers/
│   │   │   └── users.controller.ts
│   │   ├── services/
│   │   │   └── users.service.ts
│   │   ├── repositories/
│   │   │   └── users.repository.ts
│   │   ├── dto/
│   │   │   ├── create-user.dto.ts
│   │   │   └── update-user.dto.ts
│   │   ├── entities/
│   │   │   └── user.entity.ts
│   │   └── users.module.ts
│   ├── products/
│   ├── orders/
│   └── auth/
├── common/
│   ├── guards/
│   ├── interceptors/
│   ├── filters/
│   └── decorators/
├── config/
│   ├── database.config.ts
│   └── jwt.config.ts
└── app.module.ts
```

**Principes :**
✓ Regrouper par domaine métier
✓ Séparer les responsabilités
✓ Centraliser le code partagé
✓ Faciliter la navigation

### 12.2 SOLID Principles

**S - Single Responsibility**
Chaque classe a une seule responsabilité.

```typescript
// Mauvais
@Injectable()
export class UsersService {
  async create(dto: CreateUserDto) {
    // Créer l'utilisateur
    // Envoyer un email
    // Logger
    // Tout ensemble !
  }
}

// Bon
@Injectable()
export class UsersService {
  async create(dto: CreateUserDto) {
    return this.usersRepository.create(dto);
  }
}

@Injectable()
export class EmailService {
  async sendWelcome(email: string) {
    // Uniquement l'envoi d'email
  }
}

@Injectable()
export class LoggerService {
  log(message: string) {
    // Uniquement le logging
  }
}
```

**O - Open/Closed**
Ouvert à l'extension, fermé à la modification.

```typescript
// Utiliser des interfaces
interface PaymentStrategy {
  pay(amount: number): Promise<void>;
}

@Injectable()
export class PayPalStrategy implements PaymentStrategy {
  async pay(amount: number) {
    // Logique PayPal
  }
}

@Injectable()
export class StripeStrategy implements PaymentStrategy {
  async pay(amount: number) {
    // Logique Stripe
  }
}

@Injectable()
export class PaymentService {
  constructor(private strategy: PaymentStrategy) {}

  async processPayment(amount: number) {
    return this.strategy.pay(amount);
  }
}
```

**D - Dependency Inversion**
Dépendre d'abstractions, pas de concrétions.

```typescript
// Mauvais
@Injectable()
export class UsersService {
  private repository = new UsersRepository();
}

// Bon
@Injectable()
export class UsersService {
  constructor(private repository: UsersRepository) {}
}
```

### 12.3 Nommage cohérent

**Conventions :**
- Services : `XxxService`
- Controllers : `XxxController`
- Repositories : `XxxRepository`
- DTOs : `CreateXxxDto`, `UpdateXxxDto`
- Entities : `Xxx`
- Modules : `XxxModule`

```typescript
// Cohérent
src/users/
  ├── users.service.ts
  ├── users.controller.ts
  ├── users.repository.ts
  ├── dto/
  │   ├── create-user.dto.ts
  │   └── update-user.dto.ts
  ├── entities/
  │   └── user.entity.ts
  └── users.module.ts
```

### 12.4 Logging professionnel

```typescript
import { Logger } from '@nestjs/common';

@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  async create(dto: CreateUserDto) {
    this.logger.log(`Creating user with email ${dto.email}`);
    
    try {
      const user = await this.usersRepository.create(dto);
      this.logger.debug(`User created with id ${user.id}`);
      return user;
    } catch (error) {
      this.logger.error(`Error creating user: ${error.message}`, error.stack);
      throw error;
    }
  }
}
```

---

## SECTION 13 : DÉPLOIEMENT

### 13.1 Build pour la production

**Dans package.json :**
```json
{
  "scripts": {
    "build": "nest build",
    "start": "node dist/main",
    "start:prod": "NODE_ENV=production node dist/main"
  }
}
```

**Compiler le code :**
```
npm run build
```

Génère le dossier `dist/` avec le code JavaScript compilé.

### 13.2 Variables d'environnement production

**.env.production :**
```
NODE_ENV=production
DATABASE_URL=postgresql://prod_user:prod_password@prod-db/prod_db
JWT_SECRET=super-secret-key-with-strong-entropy
PORT=3000
LOG_LEVEL=warn
```

**Bonnes pratiques :**
- Ne JAMAIS commiter les .env
- Utiliser AWS Secrets Manager ou Vault
- Valider les variables requises au démarrage

### 13.3 PM2 (Gestionnaire de processus)

**Installation :**
```
npm install -g pm2
```

**Fichier ecosystem.config.js :**
```javascript
module.exports = {
  apps: [
    {
      name: 'nestjs-app',
      script: './dist/main.js',
      instances: 'max',
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3000,
      },
    },
  ],
};
```

**Commandes :**
```
pm2 start ecosystem.config.js
pm2 save
pm2 startup
pm2 monit  # Monitorer
```

### 13.4 Docker

**Dockerfile :**
```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

**docker-compose.yml :**
```yaml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://user:password@postgres:5432/mydb
      JWT_SECRET: secret-key
    depends_on:
      - postgres
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
volumes:
  postgres_data:
```

**Démarrer :**
```
docker-compose up
```

### 13.5 Checklist avant production

**Sécurité :**
- ✓ JWT secret fort
- ✓ Pas de secrets en dur
- ✓ HTTPS activé
- ✓ CORS configuré correctement

**Performance :**
- ✓ Caching implémenté
- ✓ Base de données indexée
- ✓ Pagination en place
- ✓ Rate limiting actif

**Qualité :**
- ✓ Tests passent (npm run test)
- ✓ Linter OK (npm run lint)
- ✓ Build sans erreurs (npm run build)
- ✓ Pas de warnings

**Monitoring :**
- ✓ Logs configurés
- ✓ Erreurs tracées
- ✓ Métriques disponibles

---

## RÉSUMÉ COMPLET

### Points clés NestJS :

1. **Architecture modulaire** - Modules, Controllers, Services
2. **Injection de dépendances** - Gestion automatique des dépendances
3. **Décorateurs** - Syntaxe expressive et déclarative
4. **TypeScript** - Typage statique et sécurité
5. **Tests** - Testing intégré et facile
6. **Performance** - Fastify, Cache, Rate Limiting
7. **Production-ready** - Docker, PM2, Déploiement
8. **Sécurité** - JWT, Guards, Validation
9. **Base de données** - Prisma, Transactions, Pagination
10. **Maintenabilité** - SOLID, Organisation, Logging

### Pour aller plus loin :

- Documentation officielle : https://docs.nestjs.com
- GitHub : https://github.com/nestjs/nest
- Awesome NestJS : https://github.com/juliandavidmr/awesome-nestjs

---

**FIN DU COURS COMPLET NESTJS**

Vous maîtrisez maintenant :
✅ Les fondations de NestJS
✅ L'architecture complète
✅ La sécurité et authentification
✅ L'accès aux données
✅ Les tests
✅ Le déploiement en production

Bravo et bon développement ! 🚀