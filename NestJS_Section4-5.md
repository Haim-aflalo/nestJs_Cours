# COURS NESTJS COMPLET
## Sections 4-5 : Routes HTTP et Validation

---

## SECTION 4 : ROUTES HTTP ET REQUÊTES

### 4.1 Les verbes HTTP

**GET - Récupérer des données**
```typescript
@Get()
findAll() {
  return this.usersService.findAll();
}

@Get(':id')
findOne(@Param('id') id: string) {
  return this.usersService.findOne(id);
}
```

Utilisation :
- GET /users → Tous les utilisateurs
- GET /users/123 → Utilisateur avec id 123

**POST - Créer des données**
```typescript
@Post()
create(@Body() createUserDto: CreateUserDto) {
  return this.usersService.create(createUserDto);
}
```

Requête :
```json
POST /users
{
  "name": "Jean",
  "email": "jean@example.com"
}
```

**PUT - Remplacer complètement**
```typescript
@Put(':id')
update(
  @Param('id') id: string,
  @Body() updateUserDto: UpdateUserDto
) {
  return this.usersService.update(id, updateUserDto);
}
```

**DELETE - Supprimer**
```typescript
@Delete(':id')
remove(@Param('id') id: string) {
  return this.usersService.remove(id);
}
```

**PATCH - Modification partielle**
```typescript
@Patch(':id')
patch(
  @Param('id') id: string,
  @Body() updateUserDto: Partial<UpdateUserDto>
) {
  return this.usersService.patch(id, updateUserDto);
}
```

### 4.2 Paramètres et requêtes

**Paramètres URL (@Param)**
```typescript
@Get(':id')
findOne(@Param('id') id: string) {
  return this.usersService.findOne(id);
}

// Utilisation : GET /users/123
// id = "123"
```

Plusieurs paramètres :
```typescript
@Get(':userId/posts/:postId')
findPost(
  @Param('userId') userId: string,
  @Param('postId') postId: string
) {
  return this.usersService.findPost(userId, postId);
}

// Utilisation : GET /users/123/posts/456
```

**Query string (@Query)**
```typescript
@Get()
findAll(@Query() query: any) {
  return this.usersService.findAll(query);
}

// Utilisation : GET /users?page=1&limit=10
// query = { page: "1", limit: "10" }
```

Avec DTO :
```typescript
export class FindAllUsersDto {
  page: number = 1;
  limit: number = 10;
  search?: string;
}

@Get()
findAll(@Query() query: FindAllUsersDto) {
  return this.usersService.findAll(query);
}
```

**Corps de requête (@Body)**
```typescript
@Post()
create(@Body() createUserDto: CreateUserDto) {
  return this.usersService.create(createUserDto);
}

// Requête POST /users
// Body : { "name": "Jean", "email": "jean@example.com" }
```

**Headers (@Headers)**
```typescript
@Get()
findAll(@Headers() headers: any) {
  console.log(headers['user-agent']);
}

// Accéder à un header spécifique
@Get()
findAll(@Headers('authorization') auth: string) {
  console.log(auth);
}
```

### 4.3 Codes de statut HTTP

**Codes courants :**
- 200 OK : Succès
- 201 Created : Créé
- 400 Bad Request : Requête invalide
- 401 Unauthorized : Non authentifié
- 403 Forbidden : Non autorisé
- 404 Not Found : Introuvable
- 500 Internal Server Error : Erreur serveur

**Définir le code de statut :**

```typescript
import { HttpCode } from '@nestjs/common';

@Post()
@HttpCode(201)
create(@Body() createUserDto: CreateUserDto) {
  return this.usersService.create(createUserDto);
}
```

Ou avec HttpStatus :
```typescript
import { HttpStatus } from '@nestjs/common';

@Post()
@HttpCode(HttpStatus.CREATED)
create(@Body() createUserDto: CreateUserDto) {
  return this.usersService.create(createUserDto);
}
```

---

## SECTION 5 : VALIDATION ET DONNÉES

### 5.1 DTOs (Data Transfer Objects)

Les DTOs définissent la structure des données.

**Exemple :**
```typescript
export class CreateUserDto {
  name: string;
  email: string;
  age: number;
}
```

**Avec le package class-validator :**

Installation :
```
npm install class-validator class-transformer
```

**DTO avec validations :**
```typescript
import { IsString, IsEmail, IsNumber, Min, Max } from 'class-validator';

export class CreateUserDto {
  @IsString()
  name: string;

  @IsEmail()
  email: string;

  @IsNumber()
  @Min(0)
  @Max(120)
  age: number;
}
```

Validateurs courants :
- @IsString() : Doit être une chaîne
- @IsEmail() : Doit être un email
- @IsNumber() : Doit être un nombre
- @IsInt() : Doit être un entier
- @IsBoolean() : Doit être booléen
- @IsArray() : Doit être un tableau
- @IsOptional() : Optionnel
- @Min(n) : Minimum
- @Max(n) : Maximum
- @Length(min, max) : Longueur
- @Matches(regex) : Expression régulière
- @IsEnum(enum) : Valeur d'une énumération

**Valider automatiquement :**

```typescript
import { ValidationPipe } from '@nestjs/common';

@Module({})
export class AppModule {}

// Dans main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
```

Maintenant, les requêtes invalides rejettent automatiquement.

### 5.2 Update DTOs

Différencier création et modification :

```typescript
// Création (tous les champs obligatoires)
export class CreateUserDto {
  @IsString()
  name: string;

  @IsEmail()
  email: string;

  @IsNumber()
  age: number;
}

// Modification (tous les champs optionnels)
export class UpdateUserDto {
  @IsString()
  @IsOptional()
  name?: string;

  @IsEmail()
  @IsOptional()
  email?: string;

  @IsNumber()
  @IsOptional()
  age?: number;
}
```

Ou utiliser PartialType :
```typescript
import { PartialType } from '@nestjs/mapped-types';

export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

### 5.3 Entités

Les entités représentent les données persistantes.

```typescript
export class User {
  id: string;
  name: string;
  email: string;
  age: number;
  createdAt: Date;
  updatedAt: Date;
}
```

**Différence :**
- DTO : Communication API (requête/réponse)
- Entité : Représentation en base de données

### 5.4 Mapper DTO → Entité

```typescript
@Injectable()
export class UsersService {
  create(createUserDto: CreateUserDto): User {
    const user: User = {
      id: Date.now().toString(),
      ...createUserDto,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    return user;
  }

  toResponseDto(user: User): UserResponseDto {
    return {
      id: user.id,
      name: user.name,
      email: user.email,
    };
  }
}
```

---

**Fin des sections 4-5**

Continuez à la section 6 pour la sécurité.