# COURS NESTJS COMPLET
## Sections 1-3 : Fondations

---

## SECTION 1 : INTRODUCTION GÉNÉRALE

### 1.1 Qu'est-ce que NestJS

NestJS est un framework Node.js progressif conçu pour construire des applications serveur efficaces et maintenables.

**Caractéristiques principales :**
- Architecture complète avec structure imposée
- Système de modules, contrôleurs, services
- Injection de dépendances intégrée
- Inspiré d'Angular

**Différence avec Express :**
- Express = liberté totale d'organisation
- NestJS = conventions strictes

### 1.2 Pourquoi NestJS existe

**Problèmes résolus :**
1. Désorganisation des projets Express
2. Chaque développeur organise différemment
3. Difficile à maintenir à long terme

**Solutions :**
- Architecture standardisée
- Composants pré-construits
- Injection de dépendances
- Tests facilités

### 1.3 Node.js vs Express vs NestJS

**Node.js :**
- Environnement d'exécution JavaScript
- Aucune structure imposée
- Bas niveau

**Express :**
- Framework minimal
- Liberté maximale
- Communauté immense
- Courbe d'apprentissage douce

**NestJS :**
- Framework complet
- Structure stricte
- Production-ready
- Courbe d'apprentissage plus raide

**Quand utiliser NestJS :**
✓ Applications d'entreprise
✓ APIs complexes
✓ Équipes multiples
✓ Besoin de structure

**Quand utiliser Express :**
✓ Prototypes rapides
✓ Microservices simples
✓ Projets légers
✓ Besoin de flexibilité

---

## SECTION 2 : FONDATIONS TECHNIQUES

### 2.1 Prérequis

Avant NestJS, vous devez maîtriser :

**Node.js et npm :**
- Installation de packages
- Scripts dans package.json
- node_modules

**TypeScript :**
- Types et interfaces
- Décorateurs
- Génériques
- Enums

**JavaScript moderne :**
- Async/await
- Promises
- Classes
- Modules ES6

**HTTP et REST :**
- Verbes HTTP (GET, POST, PUT, DELETE)
- Status codes (200, 404, 500...)
- Headers et body

### 2.2 Installation de l'environnement

**Étape 1 : Node.js**
Installer Node.js 18+ depuis nodejs.org

Vérifier l'installation :
```
node --version
npm --version
```

**Étape 2 : Installer NestJS CLI**
```
npm install -g @nestjs/cli
```

Vérifier l'installation :
```
nest --version
```

**Étape 3 : Créer un projet**
```
nest new mon-app
cd mon-app
```

Choisir npm comme gestionnaire de packages.

**Étape 4 : Démarrer le serveur**
```
npm run start
```

L'application sera disponible sur http://localhost:3000

### 2.3 Structure d'un projet NestJS

```
mon-app/
├── src/
│   ├── main.ts          (Point d'entrée)
│   ├── app.module.ts    (Module principal)
│   ├── app.controller.ts
│   ├── app.service.ts
│   └── users/           (Dossier par domaine)
│       ├── users.controller.ts
│       ├── users.service.ts
│       ├── users.module.ts
│       └── dto/
├── test/                (Tests e2e)
├── dist/                (Build compilé)
├── package.json
├── tsconfig.json
└── nest-cli.json
```

**main.ts :** Point d'entrée de l'application

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

### 2.4 Concepts clés

**Module :**
- Groupe fonctionnalités liées
- Importe/exporte d'autres modules
- Déclare contrôleurs et services

**Contrôleur :**
- Gère les routes HTTP
- Extrait les paramètres
- Appelle les services

**Service :**
- Contient la logique métier
- Réutilisable
- Injecté dans les contrôleurs

**Provider :**
- Tout ce qui peut être injecté
- Services, repositories, factories

**Injection de dépendances :**
- NestJS crée et injecte les instances
- Facilite les tests
- Réduit le couplage

---

## SECTION 3 : ARCHITECTURE NESTJS

### 3.1 Modules

Les modules organisent l'application en blocs fonctionnels.

**Exemple : Module Users**

```typescript
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

**Propriétés :**
- `controllers` : Liste des contrôleurs
- `providers` : Services et autres providers
- `exports` : Exposer les providers aux autres modules
- `imports` : Importer d'autres modules

**Module principal (AppModule) :**

```typescript
import { Module } from '@nestjs/common';
import { UsersModule } from './users/users.module';
import { ProductsModule } from './products/products.module';

@Module({
  imports: [UsersModule, ProductsModule],
})
export class AppModule {}
```

### 3.2 Contrôleurs

Les contrôleurs gèrent les routes HTTP.

**Anatomie d'un contrôleur :**

```typescript
import { Controller, Get, Post, Body, Param } from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}
```

**Décorateurs HTTP :**
- @Get() : GET
- @Post() : POST
- @Put() : PUT
- @Delete() : DELETE
- @Patch() : PATCH

**Décorateurs de paramètres :**
- @Param('id') : Paramètre URL
- @Query() : Query string
- @Body() : Corps de requête
- @Headers() : Headers
- @Req() : Objet requête complet
- @Res() : Objet réponse complet

### 3.3 Services

Les services contiennent la logique métier.

**Exemple :**

```typescript
import { Injectable } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';
import { User } from './entities/user.entity';

@Injectable()
export class UsersService {
  private users: User[] = [];

  create(createUserDto: CreateUserDto): User {
    const user: User = {
      id: Date.now().toString(),
      ...createUserDto,
    };
    this.users.push(user);
    return user;
  }

  findAll(): User[] {
    return this.users;
  }

  findOne(id: string): User | undefined {
    return this.users.find(user => user.id === id);
  }

  update(id: string, updateUserDto: Partial<CreateUserDto>): User {
    const user = this.findOne(id);
    if (!user) return null;
    Object.assign(user, updateUserDto);
    return user;
  }

  remove(id: string): boolean {
    const index = this.users.findIndex(user => user.id === id);
    if (index === -1) return false;
    this.users.splice(index, 1);
    return true;
  }
}
```

### 3.4 Injection de dépendances

NestJS gère automatiquement les dépendances.

**Comment ça fonctionne :**

```typescript
// Service A
@Injectable()
export class DatabaseService {
  connect() {
    console.log('Connexion à la base');
  }
}

// Service B dépend de A
@Injectable()
export class UsersService {
  constructor(private db: DatabaseService) {}

  getUsers() {
    this.db.connect();
    return [];
  }
}

// Module
@Module({
  providers: [DatabaseService, UsersService],
})
export class AppModule {}
```

NestJS crée DatabaseService, puis l'injecte dans UsersService automatiquement.

**Avantages :**
✓ Testabilité : Facile de mocker
✓ Réutilisabilité : Même instance partout
✓ Découplage : Pas de dépendances directes

---

**Fin des sections 1-3**

Continuez à la section 4 pour les routes HTTP et requêtes.