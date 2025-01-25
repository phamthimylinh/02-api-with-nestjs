Nestjs is a framework for building Node.js applications. It is somewhate opinionated and forces us to follow its vision of how an application should look like to some extent. That might be viewed as a good thing that helps us to keep consistency across our application and forces us to follow good practices.

Nestjs uses Express.js under the hood by default. If you're familiar with [typescript express series](http://wanago.io/2018/12/03/typescript-express-tutorial-routing-controllers-middleware/) and you're enjoyed it, there is a great chance you will like nestjs too. Also, knowledge of the Express framework will come in handy.

![Nest is the fastest-growing Node.js technology in terms of stars on Github in 2019](https://wanago.io/wp-content/uploads/2020/05/Screenshot-from-2020-05-09-20-51-51.png)

An important note is that the [documentation of Nestjs](https://docs.nestjs.com/) is comprehansive, and you would benefit from looking it up. Here, we attempt to put the knowledge in order, but we also sometimes lik to the official docs. We also refer to the Express framework to highlight the advantages of using Nestjs. To benefit from this article more, some experience with Express might be useful, but not necessary.

> If you want to look into the core of Node.js, i recommend checking out the [Node.js Typescript series](http://wanago.io/2019/02/11/node-js-typescript-modules-file-system/). It covers topics such as streams, event loop, multiple processes and multithreading with worker threads. Also, knowing how to create an API without any frameworks such as Express and Nestjs makes us apprieciate them even more.

# Getting started with Nestjs

The most straightforward way of getting started is to clone the official Typescript starter repository. Nest is built with Typescript and fully supports it. You could use Javascript instead, but here we focus on Typescript.

```bash
git clone git@github.com:nestjs/typescript-starter.git
```

A thing worth looking into in the above repository is the `tsconfig.json` file. I highly recommend adding the `alwaysStrict` and `noImplicitAny` options.

The above repository contains the most basic packages. we also get the fundamental types of files to get us started, so let's review them.

> All of the code from this series can be found in [this repository](https://github.com/mwanago/nestjs-typescript). Hopefully, it can later serve as a Nestjs boilerplate with some built-in features. It is a fork of an offical [typescript-starter](https://github.com/nestjs/typescript-starter). Feel free to give both of them a star.

# Controllers
Controllers handle incoming requests and return responses to the client. The `typescript-starter` repository contains out first controller. Let's create a more robust one:
> posts.controller.ts

```typescript
import { Body, Controller, Delete, Get, Param, Post, Put } from '@nestjs/common';
import PostsService from './posts.service';
import CreatePostDto from './dto/createPost.dto';
import UpdatePostDto from './dto/updatePost.dto';

@Controller('posts')
export default class PostsController {
  constructor(
    private readonly postsService: PostsService
  ) {}

  @Get()
  getAllPosts() {
    return this.postsService.getAllPosts();
  }

  @Get(':id')
  getPostById(@Param('id') id: string) {
    return this.postsService.getPostById(Number(id));
  }

  @Post()
  async createPost(@Body() post: CreatePostDto) {
    return this.postsService.createPost(post);
  }

  @Put(':id')
  async replacePost(@Param('id') id: string, @Body() post: UpdatePostDto) {
    return this.postsService.replacePost(Number(id), post);
  }

  @Delete(':id')
  async deletePost(@Param('id') id: string) {
    this.postsService.deletePost(Number(id));
  }
}
```
> post.interface.ts

```typescript
export interface Post {
  id: number;
  content: string;
  title: string;
}
```
The first thing that we can notice is that NestJS uses decorators a lot. To mark a class to be a controller, we use the `@Controller()` decorator. We pass an optional argument to it. It acts as a path prefix to all of the routes within the controller.

# Routing
The next set of decorators connected to routing in the above controller are `@Get()`, `@Post()`, `@Delete()`, and `@Put()`. They tell Nest to create a handler for a specific endpoint for HTTP requests. The above controller creates a following set of endpoints:
`GET /posts` - Return all posts
`GET /posts/{id}` - Return a post with a given id
`POST /posts` - Create a new post
`PUT /posts/{id}` - Replace a post with a given id
`DELETE /posts/{id}` - Remove a post with a given id

By default, NestJS responds with a **200 OK** status code with the exception of **201 Created** for the POST. [We can easily change that](https://docs.nestjs.com/controllers#status-code) with the `@HttpCode()` decorator.

When we implement an API, we often need to refer to a specific element. We can do so with with **route parameters**. They are special URL segments used to capture values specified at their position. To create a route parameter, we need to prefix its name with the `:` sign.

The way to extract the value of a route parameter is to use the `@Param()` decorator. Thanks to it, we have access to it in the arguments of our route handler.
> We can use an optional argument to refer to a specific parameter, for example `@Param('id')`. Otherwise, we get access to the `params` object with all parameters.

Since route parameters are strings and our ids are number, we need to convert the params first.
> We can also use **pipes** to transform the route params. Pipes are built-in feature in NestJS and we conver them later.

# Accessing the body of a request

When we handle POST and PUT in the controller above, we also need to access the body of a request. By doing so, we can use it to populate our database.

NestJS provides a  `@Body()` decorator that gives us easy access to the body. Just as in the [Typescript Express series](http://wanago.io/2018/12/17/typescript-express-error-handling-validation/), we introduce the concept of a Data Transfer Object (DTO). It defines the format of the data sent in a request. It can be either an interface or a class, but using the latter gives us more possibilities and we explore them later on.

> createPost.dto.ts

```typescript
export class CreatePostDto {
  content: string;
  title: string;
}
```

> updatePost.dto.ts

```typescript
export class UpdatePostDto {
  id: number;
  content: string;
  title: string;
}
```

# The argurememts of a handler function
Let's look into the arguments of the handler function a bit more.

```typescript
async replacePost(@Body() post: UpdatePostDto, @Param('id') id: string) {
  return this.postsService.replacePost(Number(id), post);
}
```
By using the method argument decorators, we tell NestJS to inject particular arguments into our methods. NestJs is built around the concept of Dependency Injection and Inversion of Control. We elaborate on in a lot as we go through various features.

> Dependency Injection is one of the techniques that implement Inversion of Control. If you want read a bit more about IoC, check out [Applying SOLID principles to your Typescript code](http://wanago.io/2020/02/03/applying-solid-principles-to-your-typescript-code/)

An important note is that reversing their order yields the same result, which might seem counter-intuitive at first.

```typescript
async replacePost(@Param('id') id: string, @Body() post: UpdatePostDto) {
  return this.postsService.replacePost(Number(id), post);
}
```

# Advantages of NestJS over Express
NestJs gives us a lot of things out of the box and expects us to design our API using **controllers**. ExpressJS, on the other hand, leaves us more flexibility but does not equip us with such tools to maintain the readability of our code.

We are free to implement controllers on our own with ExpressJS. We do it in the [Typescript Express series](http://wanago.io/2018/12/03/typescript-express-tutorial-routing-controllers-middleware/)

```typescript
import { Request, Response, Router } from 'express';
import Controller from '../../interfaces/controller.interface';
import PostsService from './posts.service';
import CreatePostDto from './dto/createPost.dto';
import UpdatePostDto from './dto/updatePost.dto';

export default class PostController implements Controller {
  private path = '/posts';
  private router = Router();
  private postsService = new PostsService();

  constructor() {
    this.initializeRoutes();
  }

  initializeRoutes() {
    this.router.get(this.path, this.getAllPosts);
    this.router.get(`${this.path}/:id`, this.getPostById);
    this.router.post(this.path, this.createPost);
    this.router.put(`${this.path}/:id`, this.replacePost);
  }

  private getAllPosts = (request: Request, response: Response) => {
    const posts = this.postsService.getAllPosts();
    response.send(posts);
  }

  private getPostById = (request: Request, response: Response) => {
    const id = request.params.id;
    const post = this.postsService.getPostById(Number(id));
    response.send(post);
  }

  private createPost = (request: Request, response: Response) => {
    const post: CreatePostDto = request.body;
    const createPost = this.postsService.createPost(post);
    response.send(createPost);
  }

  private replacePost = (request: Request, response: Response) => {
    const id = request.params.id;
    const post: UpdatePostDto = request.body;
    const replacePost = this.postsService.replacePost(Number(id), post);
    response.send(replacePost);
  }

  private deletePost = (request: Request, response: Response) => {
    const id = request.params.id;
    this.postsService.deletePost(Number(id));
    response.sendStatus(200);
  }
}
```
Above, we can see a similar controller created in pure Express. There are quite a few notable differences.
First, we need to handle the routing of the controller ourselves. We don't have such convenient decoractors that we can depend on to do it for us. The way NestJS works here resembles a bit Spring Framework written for Java.

In the [Typescript Express series](http://wanago.io/2018/12/03/typescript-express-tutorial-routing-controllers-middleware/), we use an `Application` class that attaches the routing to the app.

```typescript
  class Application {
    // ...
    private initializeContrllers(controllers: Controller[]) {
      controllers.forEach((controller) => {
        this.app.use('/', controller.router);
      })
    }
    // ...   
  }
```

Another big advantage of NestJS is that it provides us with an elegant way of handling the **Request** and **Response** objects. Decorators such as `@Body()` and `@Param()` help to improve the readability of our code.

One of the most useful things NestJS has to offer is how it handles responses. Our route handlers can return primitive types (for example, string), promises, or even RxJS observable streams. We don't need to handle it manually every time and use the `response.send` function. NestJS also makes it easy to handle errors in our application, and we explore it in the upcoming parts of this series.

> When using NestJS, we can also manipulate the [Request](https://docs.nestjs.com/controllers#request-object) and [Response](https://docs.nestjs.com/controllers#appendix-library-specific-approach) objects directly.
Handling responses ourselves strips us from some of the advantages of NestJS though.

There is also a difference in how we can handle dependencies in pur Express and NestJS.
In the above Express controller, we create a new `postsService` directly in the `PostsController`.
Unfortunately, it breaks the **Dependency inversion principle** from the SOLID principles. One of the issues that can cause is some trouble with writing tests.

NestJS, on the other hand, cares about compliance with the Dependency inversion principle a lot by implementing Dependency Injection.

# Services
The `typescript-starter` repository also contains our first service. A job of a services is to separate the business logic from controllers, making it cleaner and more comfortable to test.
Let's create a simple service for our posts.

> posts.service.ts

```typescript
import { HttpException, HttpStatus, Injectable } from '@nestjs/common';
import CreatePostDto from './dto/createPost.dto';
import Post from './post.interface';
import UpdatePostDto from './dto/updatePost.dto';

@Injectable()
export default class PostsService {
  private lastPostId = 0;
  private posts: Post[] = [];

  getAllPosts() {
    return this.posts;
  }

  getPostById(id: number) {
    const post = this.posts.find(post => post.id === id);
    if (post) {
      return post;
    }

    throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
  }

  replacePost(id: number, post: UpdatePostDto) {
    const postIndex = this.posts.findIndex(p => p.id === id);

    if (postIndex > -1) {
      this.posts[postIndex] = post;
      return post;
    }

    throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
  }

  createPost(post: CreatePostDto) {
    const newPost = {
      id: ++this.lastPostId,
      ...post
    };

    this.posts.push(newPost);
    return newPost;
  }

  deletePost(id: number) {
    const postIndex = this.posts.findIndex(p => p.id === id);

    if (postIndex > -1) {
      this.posts.splice(postIndex, 1);
    } else {
      throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
    }
  }
}
```
White the above logic is straightforward, there are a few noteworthy lines there.

We use the built-in `HttpException` class to throw errors that NestJS can understand. When we throw `HttpException('Post not found', HttpStatus.NOT_FOUND)`, it gets propagated to the **global exception filter**, and a proper response is sent to client. We explore this topic more in the upcoming parts of this series.
![](https://wanago.io/wp-content/uploads/2020/05/Screenshot-from-2020-05-10-17-39-24.png)

The `@Injectable()` decorator tells Nest that this class is a provider. Thanks to that, we can add it in to a module

# Modules
we use modules to organize our application. Our  `PostsController` and `PostsService` are closely related and belong to the same application domain. Therefore, it is appropriate to put them in a module together.

By doing so, we organize our code per feature. This is especially useful as your application grows.
> posts.module.ts

```typescript
import { Module } from '@nestjs/common';
import PostsController from './posts.controller';
import PostsService from './posts.service';

@Module({
  imports: [],
  controllers: [PostsController],
  providers: [PostsService],
})

export default class PostsModule {}
```
 Also, every application needs a **root module**. It is a starting point for NestJS when building the application.

 > app.module.ts
 ```typescript
import { Module } from '@nestjs/common';
import { PostModule } from './posts/posts.module';

@Module({
  imports: [PostModule],
  controllers: [PostsController],
  providers: [PostsService],
})

export default class AppModule {}
```
The module contains:
- imports 
  - imported modules - NestJS uses the `PostModule` thanks to importing it in our `AppModule`.

- controllers
  - controllers to instantiate.

- providers
  - providers to instantiate - they may be used at least across this module

- exports
  - a subset of providers that are available in other modules.

# Summary
By doing all of the above, our **src** directory ends up like that:
```bash
├── src
│   ├── app.module.ts
│   ├── main.ts
│   └── posts
│       ├── dto
│       │   ├── createPost.dto.ts
│       │   └── updatePost.dto.ts
│       ├── post.interface.ts
│       ├── posts.controller.ts
│       ├── posts.module.ts
│       └── posts.service.ts
```

In this article, we've just got started with NestJS. We've figured out what is a **Controlelr** and how to handle elementary routing in our application. We've also briefly touched the topic of **Services** and **Modules**. In the upcoming parts of this series, we will spend quite some time discussing the applications structure in NestJS.

All of above knowledge is just the tip of the NestJS iceberg. Hopefully, it convinces you that it is worth looking into this framework as it provides lots of value. There is a lot to say about features that NestJS delivers, such as neat error handling and dependency injection. We will also look into the PostgreSQL database and how to use it both through ORM and SQL statements.
You can expect it in this series, and more, so stay tuned!
