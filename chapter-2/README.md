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
# Connecting a NestJS application with PostgreSQL
A first thing to do once we have our database running is to define a connection between our application and the database.
To do so, we use `TypeOrmModule`.
> npm install @nestjs/typeorm typeorm pg

To keep our code clean, i suggest creating a database module.
> database.module.ts

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        host: configService.get('POSTGRES_HOST'),
        port: configService.get('POSTGRES_PORT'),
        username: configService.get('POSTGRES_USER'),
        password: configService.get('POSTGRES_PASSWORD'),
        database: configService.get('POSTGRES_DB'),
        entities: [
          __dirname + '/**/*.entity{.ts,.js}',
          synchronize: true
        ],
      })}
    })
  ]
)}

export default class DatabaseModule {}
```
> The synchronize flag above is very important. We will elaborate on it a lot later

An essential thing above is that we use the `ConfigModule` and `ConfigService`. The `useFactory` method can access the environment variables thank to providing the `imports` and `inject` arrays.
We elaborate on these mechanisms in the upcoming parts of this series.

Now, we need to import our `DatabaseModule`

> app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { PostModule } from './posts/posts.module';
import { ConfigModule } from '@nestjs/config';
import * as Joi from '@hapi/joi';
import { DatabaseModule } from './database/database.module';

@Module({
  imports: [
    PostsModule,
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        POSTGRES_HOST: Joi.string().required(),
        POSTGRES_PORT: Joi.number().required(),
        POSTGRES_USER: Joi.string().required(),
        POSTGRES_PASSWORD: Joi.string().required(),
        POSTGRES_DB: Joi.string().required(),
        PORT: Joi.number(),
      })
    }),
    DatabaseModule,
  ],
  controllers: [],
  providers: [],
})

export default class AppModule {}
```

# Entities
The most crucial concept to grasp when using TypeORM is the **entity**. It is a class that maps to a database table. To create it, we use the `@Entity()` decorator.

> post.entity.ts

```typescript
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
class Post {
  @PrimaryGeneratedColumn()
  public id: number;

  @Column()
  public title: string;
  
  @Column()
  public content: string;
}

export default Post;
```
A neat thing about TypeORM is that it integrates well with Typescript because it is written in it. To define our columns, we can use various decorators.

## @PrimaryGeneratedColumn()
A **primary key** is a column used to identify a row uniquely in a table. Althrough we might use an existing column and make it primary, we usually create an **id** column.
By choosing `PrimaryGeneratedColumn` from TypeORM, we create an **integer** primary column that has a value generated automatically.

## @Column()
The `@Column()` decorator marks a property as column. When using it, we have two possible approaches.

The first approach is not to pass the column type explicitly. When we do it, TypeORM figures out the column using out the column our typescript types. It is possible because NestJS use [reflect-metadata](https://www.npmjs.com/package/reflect-metadata) under hood

The second approach would be to pass the type of column explicitly, for example, by using `@Column('text')`. The available column types differ between databases like MySQL and PostgreSQL. You can look them in the [TypeORM documentation](https://github.com/typeorm/typeorm/blob/master/docs/entities.md#column-types-for-postgres)

It is a proper moment to discuss different ways to store strings in Postgre. Relying on TypeORM to figure out the type of string column results in the "character varying" type, also call **varchar**.

![character varying](https://wanago.io/wp-content/uploads/2020/05/Screenshot-from-2020-05-16-00-06-55.png)

Varchar is very similar to a **text** type of a column but gives us a possibility to limit the length of a string. Both types are the same performance-wise.

## SQL query
In pgAdmin, we can check a query equivalent to what TypeORM did for us under the hood.

```
CREATE TABLE public.post
(
    id integer NOT NULL DEFAULT nextval('post_id_seq'::regclass)'),
    title character varying COLLATE pg_catalog."default" NOT NULL,
    content character varying COLLATE pg_catalog."default" NOT NULL,
    CONSTAINT "PK_be5fda3aac270b134ff9c21cdee" PRIMARY KEY (id)
)
```

There are a few interesting things to notice above

Using `@PrimaryGenaratedColumn()` results in having an **int** column. It defaults to return value of a `nextval` function that returns unique ids. An alternative would be to use a [serial type instead](https://www.postgresql.org/docs/9.1/datatype-numeric.html#DATATYPE-SERIAL) and would make the query shorter, but it works the same under the hood.

Our entity has **varchar** columns that use **COLLATE**. Collation is used to specify to sort order and character classification.
To see our default collation, we can run this query:
> SHOW LC_COLLATE

> en_US.utf8

The above value is defined in a query that was used to create our database. It is UTF8  and English by default.

```
CREATE DATABASE nestjs
  WITH
  OWNER = admin
  ENDCODING = 'UTF8'
  LC_COLLATE = 'en_US.utf8'
  TABESPACE = pg_default
  CONNECTION LIMIT = -1
```

Also, out `CREATE TABLE` query puts a **constraint** on ids so that they are always unique

> PK_be5fda3aac270b134ff9c21cdee is a name of the above constaint and was generated

# Repository
With **repositories**, we can manage a particular entity. A repository has multiple functions to interact with entities. To access it, we use the `TypeOrmModule` again.

> posts.module.ts
```typescript
import { Module } from '@nestjs/common';
import PostsController from './posts.controller';
import PostsService from './posts.service';
import Post from './posts.entity';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [TypeOrmModule.forFeature([Post])],
  controllers: [PostsController],
  providers: [PostsService]
})

export class PostsModule {}
```
Now, in our `PostsService`, we can inject to Posts repository

> import { InjectRepository } form '@nestjs/typeorm'

```typescript
contructor(
  @InjectRepository(Post)
  private postsRepository: Repository<PostEntity>
){}
```

## Finding
With the `find` function, we can get multiple elements. If we don't provide ith with any options, it returns all.

```typescript
getAllPosts() {
  return this.postsRepository.find();
}
```
To get just one element we use the `findOne` function. By providing it with a number we indicate that we want an element with a particular id. If the result is undefined, it means that the element wasn't found.

```typescript
async getPostById(id: number) {
  const post = await this.postsRepository.findOne(id);
  if (post) {
    return post;
  }
  throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
}
```

## Creating 
By using the `create` function, we can instantiate a new Post. We can use the `save` function afterward to populate the database  with our new entity.

```typescript
async createPost(post: CreatePostDto) {
  const newPost = await this.postsRepository.create(post)
  await this.postsRepository.save(newPost);
  return newPost;
}
```

## Modifying
To modify an existing element, we can use `update` function. Afterward, we would use the `findOne` function to return the modify element

```typescript
async updatePost(id: number, post: updatePostDto) {
  await this.postsRepository.update(id, post);
  const updatePost = await this.postsRepository.findOne(id);
  if (updatePost) {
    return updatePost;
  }

  throw new HttpException('Post not found', HttpStatus.NOT_FOUND)
}
```

A significant thing is that it accepts a partial entity, so it acts as a PATCH, not as a PUT. If you want to read more on PUT vs PATCH (although with mongoDB), check out [Typescript Express tutorial #15 Using PUT vs PATCH in MongoDB with Mongoose](http://wanago.io/2020/04/27/typescript-express-put-vs-patch-mongodb-mongoose/)
## Deleting
To delete an element with a given id, we can use the `delete` function
```typescript
async deletePost(id: number) {
  const deleteResponse = await this.postsRepository.delete(id);
  if(!deleteReponse.affected) {
    throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
  }
}
```
By checking out the [documentation of the DELETE command](https://www.postgresql.org/docs/11/sql-delete.html), we can see that we have access to the count of removed elements. This data is available in the `affected` property. If it equals zero, we can assume that the element does not exist.

## Handling asynchronous errors
A benificial thing about NestJS controllers is that they handle asynchronous errors very well.

```typescript
@Get(':id')
getPostById(@Param('id') id: string) {
  return this.postsService.getPostById(Number(id));
}
```
If the `getPostById` function throws an error; NestJS catches it automatically and parses it. When using pure Express, we would do this ourselves:

```typescript
getAllPost = async(request: Request, response: Response, next: Next) => {
  const id = request.params.id;
  try {
    const post = await this.postsService.getPostById();
    response.send(post);
  } catch(error){
    next(error);
  }
}
```
# Summary
In this artical, we've gone through the basics of connecting our NextJS application with a PostgreSQL database. Not only, did we use TypeORM, but we've also looked into some SQL queries. NestJS and TypeORM have lots of features built-in and ready to use. In the upcoming parts of this series, we will look into them more, so stay tuned!















