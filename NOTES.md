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

## 5. General Installation

### 5.1. Install Bcryptjs

- [Install Bcryptjs](https://www.npmjs.com/package/bcryptjs)

```bash
npm install bcryptjs
```

and

```bash
npm install @types/bcryptjs -D
```
