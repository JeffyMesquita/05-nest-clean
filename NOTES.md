# Clean Architecture - NestJs

## 1. NestJs

- NestJs is a framework for building efficient, scalable Node.js server-side applications. It uses progressive JavaScript, is built with TypeScript (preserves compatibility with pure JavaScript) and combines elements of OOP (Object Oriented Programming), FP (Functional Programming), and FRP (Functional Reactive Programming).

## 2. Using Docker

- Docker is a tool designed to make it easier to create, deploy, and run applications by using containers. Containers allow a developer to package up an application with all of the parts it needs, such as libraries and other dependencies, and ship it all out as one package.

### 2.1. Install Docker

- [Install Docker](https://docs.docker.com/engine/install/)
- Create a file `docker-compose.yml` in the root of the project with the following content:

```yml
version: '3.7'

services:
  postgres:
    image: postgres:12.0-alpine
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - 5432:5432
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
```

- Create a folder `data/postgres` in the root of the project
- Run command `docker-compose up -d` to start the container

## 3. Using Prisma

- Prisma is a data layer that replaces traditional ORMs. It sits between your database and application and allows you to query your database directly with GraphQL.

### 3.1. Install Prisma

- [Install Prisma](https://www.prisma.io/docs/getting-started/setup-prisma/start-from-scratch-typescript-postgres)

```bash
npm install prisma -D
```

and

```bash
npm install @prisma/client
```

- After, run the following command to initialize Prisma:

```bash
npx prisma init
```

- Create your database schema in the `prisma/schema.prisma` file:

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

- Change the `DATABASE_URL` in the `.env` file to `postgresql://postgres:postgres@localhost:5432/postgres?schema=public`

- Run the following command to create the database tables for your models:

```bash
npx prisma migrate dev
```

- Run the following command to see the database data:

```bash
npx prisma studio
```

### 3.2. Using Prisma Client

- in `src` folder, create a folder `prisma` and inside create a file `prisma.service.ts` with the following content:

```ts
import { Injectable } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor() {
    // you can pass config here
    super({
      // log: ['query'] for debugging
      log: ['error', 'warn', 'info', 'query'],
    });
  }

  // this is required to make Prisma Client work
  // in a multi-threaded context like NestJS application
  onModuleInit() {
    return this.$connect();
  }

  onModuleDestroy() {
    return this.$disconnect();
  }
}
```

- in `src/app.module.ts` file, add the following code:

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { PrismaService } from './prisma/prisma.service';

@Module({
  imports: [],
  controllers: [AppController],
  // add PrismaService to the providers array
  providers: [AppService, PrismaService],
})
export class AppModule {}
```

### 3.3. Using Zod as a validation library Pipe(as a middleware)

- After installing Zod, create a file `validation.pipe.ts` in the `src/pipes` folder with the following content:

```ts
import { BadRequestException, PipeTransform } from '@nestjs/common';
import { ZodError, ZodSchema } from 'zod';
import { fromZodError } from 'zod-validation-error';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown) {
    try {
      this.schema.parse(value);
      return value;
    } catch (error) {
      if (error instanceof ZodError) {
        throw new BadRequestException({
          message: 'Validation failed',
          statusCode: 400,
          errors: fromZodError(error),
        });
      }

      throw new BadRequestException('Validation failed');
    }
  }
}
```

## 4. General Configs

### 4.1. tsconfig.json

- In `tsconfig.json` you can add:

```json
{
  "compilerOptions": {
    //...
    "strict": true,
    "strictNullChecks": true
    //...
  }
}
```

### 4.2. Using `@nest/config`

- Run the following command to install `@nest/config`:

```bash
npm install @nestjs/config
```

- In `src` folder, create a file `env.ts` with the following content:

```ts
import { z } from 'zod';

export const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().optional().default(3333),
});

export type Env = z.infer<typeof envSchema>;
```

- In `app.module.ts` file, add the following code:

```ts
//...
import { ConfigModule } from '@nestjs/config';
//...
import { envSchema } from './env';
//...

@Module({
  imports: [
    //...
    ConfigModule.forRoot({
      // using zod as a validation library
      validate: (env) => envSchema.parse(env),
      isGlobal: true,
    }),
    //...
  ],
  //...
})

```

- For example, in `main.ts` file, you can use:

```ts
//...

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    // logger: false
  });

  const configService = app.get<ConfigService<Env, true>>(ConfigService);
  const port = configService.get('PORT', { infer: true });

  await app.listen(port);
}
bootstrap();
```

### 4.3. Using JWT for Auth with `@nestjs/jwt` and `@nestjs/passport`

- Run the following command to install `@nestjs/jwt` and `@nestjs/passport`:

```bash
npm install @nestjs/jwt @nestjs/passport
```

- In `src` folder, create a folder `auth` and inside create a file `auth.module.ts` with the following content:

```ts
import { Module } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { Env } from 'src/env';

@Module({
  imports: [
    PassportModule,
    JwtModule.registerAsync({
      inject: [ConfigService],
      global: true,
      useFactory(config: ConfigService<Env, true>) {
        const privateKey = config.get('JWT_PRIVATE_KEY', { infer: true });
        const publicKey = config.get('JWT_PUBLIC_KEY', { infer: true });

        return {
          signOptions: {
            algorithm: 'RS256',
            allowInsecureKeySizes: true,
          },
          privateKey: Buffer.from(privateKey, 'base64'),
          publicKey: Buffer.from(publicKey, 'base64'),
        };
      },
    }),
  ],
})
export class AuthModule {}
```

- You need to generate a private and public key for JWT. You can use the following command:

```bash
ssh-keygen -t rsa -b 4096 -m PEM -f jwtRS256.key
```

- After, you need to generate the public key:

```bash
openssl rsa -in jwtRS256.key -pubout -outform PEM -out jwtRS256.key.pub
```

- Add the following code in the `.env` file:

```env
//...
JWT_PRIVATE_KEY=your_private_key
JWT_PUBLIC_KEY=your_public_key
```

- In `src/app.module.ts` file, add the following code:

````ts
//...
import { AuthModule } from './auth/auth.module';
//...

@Module({
  imports: [
    //...
    AuthModule,
    //...
  ],
  controllers: [...AnyController, YourAuthController],
  //...
})


## 5. General Installation

### 5.1. Install Bcryptjs

- [Install Bcryptjs](https://www.npmjs.com/package/bcryptjs)

```bash
npm install bcryptjs
````

and

```bash
npm install @types/bcryptjs -D
```

### 5.2. Install Zod

- [Install Zod](https://www.npmjs.com/package/zod)

```bash
npm install zod
```

and

```bash
npm install zod-validation-error
```
