Nestjs is a framework for building Node.js applications. It is somewhate opinionated and forces us to follow its vision of how an application should look like to some extent. That might be viewed as a good thing that helps us to keep consistency across our application and forces us to follow good practices.

Nestjs uses Express.js under the hood by default. If you're familiar with [typescript express series](http://wanago.io/2018/12/03/typescript-express-tutorial-routing-controllers-middleware/) and you're enjoyed it, there is a great chance you will like nestjs too. Also, knowledge of the Express framework will come in handy.

![Nest is the fastest-growing Node.js technology in terms of stars on Github in 2019](https://wanago.io/wp-content/uploads/2020/05/Screenshot-from-2020-05-09-20-51-51.png)

An important note is that the [documentation of Nestjs](https://docs.nestjs.com/) is comprehansive, and you would benefit from looking it up. Here, we attempt to put the knowledge in order, but we also sometimes lik to the official docs. We also refer to the Express framework to highlight the advantages of using Nestjs. To benefit from this article more, some experience with Express might be useful, but not necessary.

> If you want to look into the core of Node.js, i recommend checking out the [Node.js Typescript series] (http://wanago.io/2019/02/11/node-js-typescript-modules-file-system/). It covers topics such as streams, event loop, multiple processes and multithreading with worker threads. Also, knowing how to create an API without any frameworks such as Express and Nestjs makes us apprieciate them even more.

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