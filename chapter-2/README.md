The next important thing when learning how to create an API is how to store the data. In this article, we look into how to do so with PostgreSQL and NestJS. To make the managing of a database more convenient, we use an **Object-Relational Mapping** (ORM) tool called **TypeORM**. To have an even better understanding, we also look into some SQL queries. By doing so, we can grasp what advantages ORM gives us.

You can find all of the below code in [this repository](https://github.com/mwanago/nestjs-typescript).

# Creating a PostgreSQL database
The most straightforward way of kickstarting our development with a PostgreSQL database is to use **Docker**. Here, we use the same setup as in the [Typescript Express series](http://wanago.io/2019/01/14/express-postgres-relational-databases-typeorm/).

The first thing to do is [install Docker](http://wanago.io/2019/01/14/express-postgres-relational-databases-typeorm/) and [docker compose](https://docs.docker.com/compose/install/). Now, we need to create a docker-compose file and run it.

> docker-compose.yml

```yaml
version: '3'
services:
  postgres:
    container_name: postgres
    image: postgres:latest
    ports:
    - "5432:5432"
    volumes:
    - ./data/postgres:/data/postgres
    env_file:
    - docker.env
    networks:
    - postgres
  
  pgadmin:
    links:
    - postgres: postgres
    container_name: pgadmin
    image: dpage/pgadmin4
    ports:
    - "8080:80"
    volumes:
    - ./data/pgadmin:/root/.pgadmin
    env_file:
    - docker.env
    networks:
    - postgres

networks:
  postgres:
    driver: bridge
```

The useful thing about above configuration is that it also starts a **pgAdmin** console. It gives us the possibility to view the state of our database interact with it.

To provide credentials used by our Docker containers, we need to create the **docker.env** file. You might want to skip committing it by adding it to your **.gitignore**

>docker.env

```
POSTGRES_USER=admin
POSTGRES_PASSWORD=admin
POSTGRES_DB=nestjs
PGADMIN_DEFAULT_EMAIL=admin@admin.com
PGADMIN_DEFAULT_PASSWORD=admin
```
Once all of above is set up, we need to start the containers.
> docker-compose up

# Environment variables
A crucial thing to running our application is to set up **environment variables**. By using them to hold configuration data, we can make it easily configurable. Also, it is easier to keep sensitive data from being committed to a repository.

In the [Express series](http://wanago.io/2018/12/10/express-mongodb-typescript-env-var/), we use a library called [dotenv](https://www.npmjs.com/package/dotenv) to inject our variables. In NestJS, we have a `ConfigModule` that we can use in our application. It uses dotenv under hood.

> npm install @nestjs/config

> app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { PostsModule } from './posts/posts.module';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [PostsModule, ConfigModule.forRoot()],
  controllers: [],
  providers: [],
})

export default class AppModule {}
```

As soon as we create a **.env** file at the root of our application, NestJS injects them into a **ConfigService** that we will use soon

> .env

```text
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=admin
POSTGRES_PASSWORD=admin
POSTGRES_DB=nestjs
PORT=5000
```

## Validating environment variables
It is an excellent idea to verify our environment variables before running the application. In the [Typescript Express series](http://wanago.io/2018/12/10/express-mongodb-typescript-env-var/), we've used a library called [envalid](https://www.npmjs.com/package/envalid).

The `ConfigModule` built into NestJS supports [@hapi/joi](https://www.npmjs.com/package/@hapi/joi) that we can use to define a **validation schema** for our environment variables schema**.

> npm install @hapi/joi @types/hapi__joi

> app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { PostsModule } from './posts/posts.module';
import { ConfigModule  } from '@nestjs/config';
import * as Joi from '@hapi/joi';

@Module({
  imports: [
    PostsModule,
    ConfigModule.forRoot
      validationSchema: Joi.object({
        POSTGRES_HOST: Joi.string().required(),
        POSTGRES_PORT: Joi.number().required(),
        POSTGRES_USER: Joi.string().required(),
        POSTGRES_PASSWORD: Joi.string().required(),
        POSTGRES_DB: Joi.string().required(),
        PORT: Joi.number(),
      })
    ]
})

export default class AppModule {}
```











