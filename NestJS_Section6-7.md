# COURS NESTJS COMPLET
## Sections 6-7 : Sécurité et Gestion des Erreurs

---

## SECTION 6 : SÉCURITÉ ET CONTRÔLE D'ACCÈS

### 6.1 Authentification JWT

JWT (JSON Web Tokens) pour authentifier les utilisateurs.

**Installation :**
```
npm install @nestjs/jwt @nestjs/passport passport passport-jwt
npm install -D @types/passport-jwt
```

**Configuration :**

```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';

@Module({
  imports: [
    JwtModule.register({
      secret: 'your-secret-key',
      signOptions: { expiresIn: '1h' },
    }),
  ],
})
export class AuthModule {}
```

**Service d'authentification :**

```typescript
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(private jwtService: JwtService) {}

  async register(email: string, password: string) {
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = { email, password: hashedPassword };
    return user;
  }

  async login(user: any) {
    const payload = { email: user.email, sub: user.id };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }

  async validateUser(email: string, password: string) {
    const user = await this.usersService.findByEmail(email);
    if (user && await bcrypt.compare(password, user.password)) {
      return user;
    }
    return null;
  }
}
```

**Contrôleur d'authentification :**

```typescript
import { Controller, Post, Body } from '@nestjs/common';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('register')
  async register(@Body() registerDto: any) {
    return this.authService.register(
      registerDto.email,
      registerDto.password
    );
  }

  @Post('login')
  async login(@Body() loginDto: any) {
    return this.authService.login(loginDto);
  }
}
```

### 6.2 Guards (Protéger les routes)

Guards contrôlent l'accès aux routes.

**JWT Guard :**

```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

**Utilisation :**

```typescript
import { UseGuards } from '@nestjs/common';

@Controller('users')
export class UsersController {
  @Get()
  @UseGuards(JwtAuthGuard)
  findAll() {
    return this.usersService.findAll();
  }
}
```

Maintenant, GET /users requiert un JWT valide.

**Guard personnalisé :**

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user) return false;
    if (user.role !== 'admin') return false;

    return true;
  }
}

// Utilisation
@UseGuards(RolesGuard)
@Delete(':id')
remove(@Param('id') id: string) {
  return this.usersService.remove(id);
}
```

### 6.3 Décorateurs pour l'utilisateur

Accéder à l'utilisateur courant :

**Décorateur personnalisé :**

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

**Utilisation :**

```typescript
@Get('profile')
@UseGuards(JwtAuthGuard)
getProfile(@CurrentUser() user: any) {
  return user;
}
```

### 6.4 Permissions et rôles

**Enum des rôles :**

```typescript
export enum Role {
  ADMIN = 'admin',
  USER = 'user',
  MODERATOR = 'moderator',
}
```

**Guard avec rôles :**

```typescript
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Role } from './role.enum';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<Role[]>(
      'roles',
      context.getHandler(),
    );

    if (!requiredRoles) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!requiredRoles.includes(user.role)) {
      throw new ForbiddenException('Accès refusé');
    }

    return true;
  }
}
```

**Décorateur @Roles :**

```typescript
import { SetMetadata } from '@nestjs/common';
import { Role } from './role.enum';

export const Roles = (...roles: Role[]) => SetMetadata('roles', roles);
```

**Utilisation :**

```typescript
@Delete(':id')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(Role.ADMIN)
remove(@Param('id') id: string) {
  return this.usersService.remove(id);
}
```

---

## SECTION 7 : GESTION DES ERREURS

### 7.1 Exceptions HTTP

NestJS fournit des exceptions HTTP intégrées.

**Exceptions courantes :**

```typescript
import {
  BadRequestException,
  NotFoundException,
  ConflictException,
  UnauthorizedException,
} from '@nestjs/common';

// Bad Request (400)
throw new BadRequestException('Données invalides');

// Not Found (404)
throw new NotFoundException('Utilisateur introuvable');

// Conflict (409)
throw new ConflictException('Email déjà existant');

// Unauthorized (401)
throw new UnauthorizedException('Non authentifié');
```

**Utilisation dans un service :**

```typescript
async findOne(id: string) {
  const user = await this.usersRepository.findById(id);
  if (!user) {
    throw new NotFoundException(`User with ID ${id} not found`);
  }
  return user;
}
```

### 7.2 Exceptions personnalisées

Créer des exceptions métier spécifiques :

```typescript
import { HttpException, HttpStatus } from '@nestjs/common';

export class EmailAlreadyExistsError extends HttpException {
  constructor(email: string) {
    super(
      {
        statusCode: HttpStatus.CONFLICT,
        message: 'Email already exists',
        email,
      },
      HttpStatus.CONFLICT,
    );
  }
}
```

**Utilisation :**

```typescript
async create(createUserDto: CreateUserDto) {
  const existing = await this.usersRepository.findByEmail(
    createUserDto.email
  );
  if (existing) {
    throw new EmailAlreadyExistsError(createUserDto.email);
  }
  return this.usersRepository.create(createUserDto);
}
```

### 7.3 Filtres d'exceptions

Centralisez la gestion des erreurs avec les filtres.

**Filtre HTTP :**

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: exception.getResponse(),
    });
  }
}
```

**Application globale :**

```typescript
// main.ts
app.useGlobalFilters(new HttpExceptionFilter());
```

### 7.4 Logging des erreurs

```typescript
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  async findOne(id: string) {
    try {
      const user = await this.usersRepository.findById(id);
      if (!user) {
        this.logger.warn(`User not found: ${id}`);
        throw new NotFoundException('User not found');
      }
      return user;
    } catch (error) {
      this.logger.error(`Error finding user: ${error.message}`, error.stack);
      throw error;
    }
  }
}
```

---

**Fin des sections 6-7**

Continuez à la section 8 pour l'accès aux données.