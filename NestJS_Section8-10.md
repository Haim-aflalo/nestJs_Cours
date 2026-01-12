# COURS NESTJS COMPLET
## Sections 8-10 : Données, Configuration, Tests

---

## SECTION 8 : ACCÈS AUX DONNÉES

### 8.1 Séparation Controller / Service / Repository

**Architecture en 3 couches :**

**Contrôleur** - Gère les requêtes HTTP
```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }
}
```

**Service** - Contient la logique métier
```typescript
@Injectable()
export class UsersService {
  constructor(private readonly usersRepository: UsersRepository) {}

  async findOne(id: string): Promise<User> {
    const user = await this.usersRepository.findById(id);
    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }
}
```

**Repository** - Accès aux données
```typescript
@Injectable()
export class UsersRepository {
  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async create(data: CreateUserData): Promise<User> {
    return this.prisma.user.create({ data });
  }
}
```

### 8.2 Connexion avec Prisma

**Installation :**
```
npm install @prisma/client
npm install -D prisma
npx prisma init
```

**Schéma (prisma/schema.prisma) :**
```
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

**Service Prisma :**
```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

**Module Prisma :**
```typescript
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

### 8.3 Pagination

```typescript
export class FindAllUsersDto {
  page: number = 1;
  limit: number = 10;
}

@Injectable()
export class UsersService {
  async findAll(dto: FindAllUsersDto) {
    const skip = (dto.page - 1) * dto.limit;
    
    const [users, total] = await Promise.all([
      this.prisma.user.findMany({
        skip,
        take: dto.limit,
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.user.count(),
    ]);

    return {
      data: users,
      pagination: {
        total,
        page: dto.page,
        limit: dto.limit,
        pages: Math.ceil(total / dto.limit),
      },
    };
  }
}
```

### 8.4 Transactions

```typescript
@Injectable()
export class OrdersService {
  constructor(private prisma: PrismaService) {}

  async createOrderWithItems(orderData, items) {
    return this.prisma.$transaction(async (tx) => {
      const order = await tx.order.create({
        data: { userId: orderData.userId, status: 'pending' },
      });

      const createdItems = await tx.orderItem.createMany({
        data: items.map((item) => ({
          orderId: order.id,
          productId: item.productId,
          quantity: item.quantity,
        })),
      });

      return { order, createdItems };
    });
  }
}
```

Si erreur → tout est annulé automatiquement.

---

## SECTION 9 : CONFIGURATION & ENVIRONNEMENT

### 9.1 Variables d'environnement

**Fichier .env :**
```
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
JWT_SECRET=my-secret-key-123
PORT=3000
NODE_ENV=development
```

**Accès simple :**
```typescript
const port = process.env.PORT || 3000;
const secret = process.env.JWT_SECRET;
```

### 9.2 @nestjs/config

**Installation :**
```
npm install @nestjs/config
```

**Configuration :**
```typescript
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env',
    }),
  ],
})
export class AppModule {}
```

**Utilisation :**
```typescript
import { ConfigService } from '@nestjs/config';

@Injectable()
export class DatabaseService {
  constructor(private configService: ConfigService) {}

  getConnectionString(): string {
    return this.configService.get<string>('DATABASE_URL');
  }
}
```

### 9.3 Configuration globale typée

```typescript
// config/database.config.ts
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT) || 5432,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
}));
```

**Utilisation :**
```typescript
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [databaseConfig],
    }),
  ],
})
export class AppModule {}
```

---

## SECTION 10 : TESTS

### 10.1 Tests unitaires

**Installation :**
```
npm install --save-dev @nestjs/testing jest @types/jest ts-jest
```

**Configuration jest.config.js :**
```javascript
module.exports = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  testEnvironment: 'node',
};
```

**Test d'un service :**

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { UsersRepository } from './users.repository';
import { NotFoundException } from '@nestjs/common';

describe('UsersService', () => {
  let service: UsersService;
  let repository: UsersRepository;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: UsersRepository,
          useValue: {
            findById: jest.fn(),
            create: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get<UsersRepository>(UsersRepository);
  });

  describe('findOne', () => {
    it('should return a user when found', async () => {
      const userId = '1';
      const user = { id: userId, name: 'John', email: 'john@example.com' };

      jest.spyOn(repository, 'findById').mockResolvedValue(user);

      const result = await service.findOne(userId);

      expect(result).toEqual(user);
      expect(repository.findById).toHaveBeenCalledWith(userId);
    });

    it('should throw NotFoundException when user not found', async () => {
      jest.spyOn(repository, 'findById').mockResolvedValue(null);

      await expect(service.findOne('999')).rejects.toThrow(NotFoundException);
    });
  });
});
```

**Exécuter les tests :**
```
npm run test
npm run test:watch   # Mode watch
npm run test:cov     # Avec couverture
```

### 10.2 Tests e2e

Tests complets de l'API HTTP.

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from './../src/app.module';

describe('Users (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe());
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('POST /users', () => {
    it('should create a user', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({
          name: 'John Doe',
          email: 'john@example.com',
          password: 'Password123',
        })
        .expect(201)
        .expect((res) => {
          expect(res.body).toHaveProperty('id');
          expect(res.body.email).toBe('john@example.com');
        });
    });

    it('should fail with invalid email', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({
          name: 'Jane Doe',
          email: 'invalid-email',
          password: 'Password123',
        })
        .expect(400);
    });
  });

  describe('GET /users/:id', () => {
    it('should return 404 for non-existent user', () => {
      return request(app.getHttpServer())
        .get('/users/999')
        .expect(404);
    });
  });
});
```

---

**Fin des sections 8-10**

Continuez à la section 11 pour la performance.