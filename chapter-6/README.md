NestJs strives to focus on the maintainability and testability of the code. To do so, it implements various mechanisms such as the Dependency Injection. In this article, we inspect how NestJS can resolve dependencies by looking into the output of the TypeScript. We also strive to understand the modular architecture that NestJS is built with.
You can find the code from this series in [this repository](https://github.com/mwanago/nestjs-typescript).

# Reasons to implement Dependency Injection
You might be familiar with my [Typescript Express series](http://wanago.io/2018/12/03/typescript-express-tutorial-routing-controllers-middleware/). It adopted a rather simplistic approach to resolving dependencies.

```typescript
import { Router } from 'express';
import Controller from '../interfaces/controller.interface';
import AuthenticationService from './authentication.service';

class AuthenticationController implements Controller {
    public path = '/auth';
    public router = Router();
    public authenticationService = new AuthenticationService();
    //...
}
```
There are a few drawbacks to the above, unfortunately. Every time we create an instance of the `AuthenticationController`, we also create a new `AuthenticationService`. If we use the above approach in all of controllers that need the `AuthenticationService`, each of them receives a separate instance. It might become hard to maintain over time.

Also, testability suffers. While it is possible to mock the `AuthenticationService` before the initialization of the above controller, it is a solution not perceived as ideal.

One of the SOLID principles is called the Inversion of Control (IoC). It states that high-level modules should not depend on the low-level modules. A straightforward way to achieve that is to create instances of dependencies first, and then provide them through a constructor.

```typescript
import { Router } from 'express';
import Controller from '../interfaces/controller.interface';
import AuthenticationService from './authentication.service';

class AuthenticationController implements Controller {
    public path = '/auth';
    public router = Router();
    public authenticationService: AuthenticationService;

    constructor(authenticationService: AuthenticationService) {
        this.authenticationService = authenticationService;
    }
}
```

```typescript
const authenticationService = new AuthenticationService();
const authenticationController = new AuthenticationController(authenticationService);
```
>> If you want to know more about IoC, check out [Applying SOLID principles to your Typescript code](http://wanago.io/2020/02/03/applying-solid-principles-to-your-typescript-code/)

While doing the above helps us overcome the mentioned issues, it is far from convenient. This is why NestJS implements a **Dependency Injection** mechanism that provides all of the necessary dependencies automatically.

# Dependency Injection in NestJS under the hood
Let's look at a similar controller that we've built in the [third part of this series](http://wanago.io/2020/05/25/api-nestjs-authenticating-users-bcrypt-passport-jwt-cookies/).

```typescript
import { Controller} from '@nestjs/common';
import { AuthenticationService } from './authentication.service';

@Controller('authentication')
export class AuthenticationController {
    constructor(
        private readonly authenticationService: AuthenticationService
    ) {}
}
```
> Thanks to using the `private readonly`, we don't need to asign the `authenticationService` in the body of the constructor.

The `@Controller` decorator, among other things, ensures that the metadata about our class is saved. The `@Injectable` decorator also does this.

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AuthenticationService {
    constructor(
        private readonly usersService: UsersService,
        private readonly jwtService: JwtService,
        private readonly configService: ConfigService
    ) {}
}
```
TypeScript compiler emits the metadata that NestJS can later use to figure out what dependencies do we need. Let's inspect the output of the `AuthenticationService`.
```typescript
AuthenticationService = __decorate([
    common_1.Injectable(),
    __metadata("design:paramtypes", [
        users_service_1.UsersService,
        jwt_service_1.JwtService,
        config_service_1.ConfigService
    ])
], AuthenticationService);
```
The `design:paramtypes` is a key describing parameter ty metadata. Thanks to it, we can obtain an array of references to classes that we need in the constructor of the `AuthenticationService`. We can perceive it as extracting the dependencies of the `AuthenticationService` at the compiler time.
NestJS uses the [reflect-metadata](https://www.npmjs.com/package/reflect-metadata) package under the hood to work with the above metadata.

When a NestJS application starts, it resolves all the metadata the `AuthenticationController` needs. It might get quite complex under the hood, as [it can deal with circular dependencies](), for example.
> If you want to dig deeper into how NestJS supplies the required dependencies, check out [this talk by Kamil Mysliwiec](https://www.youtube.com/watch?v=vYFhHVMetPg), the creator of NestJS.

# Modules
A module is a part of our application that holds functionalities that revolve around a particular feature.

Every NestJS application has a root module. It serves as a starting point for NestJS when creating an **application graph**.

```typescript
import { Module } from '@nestjs/common';
import { PostsModule } from './posts/posts.module';
import { ConfigModule } from '@nestjs/config';
import { DatabaseModule } from './database/database.module';
import { AuthenticationModule } from './authentication/authentication.module';
import { UsersModule } from './users/users.module';

@Module({
    imports: [
        PostsModule,
        ConfigModule.forRoot({
            // ...
        }),
        DatabaseModule,
        AuthenticationModule,
        UsersModule
    ],
    providers: [],
    controllers: []
})
export class AppModule {}
```
NestJS uses the root module to resolve other modules along with their dependencies. For example, we have the `AuthenticationModule` that takes care of the authentication in our application. We create it in the [third part of this series](http://wanago.io/2020/05/25/api-nestjs-authenticating-users-bcrypt-passport-jwt-cookies/) to handle the process of registering and verifying users.
```

├── authentication
│   ├── authentication.controller.ts
│   ├── authentication.module.ts
│   ├── authentication.service.ts
│   ├── dto
│   │   ├── logIn.dto.ts
│   │   └── register.dto.ts
│   ├── jwtAuthentication.guard.ts
│   ├── jwt.strategy.ts
│   ├── localAuthentication.guard.ts
│   ├── local.strategy.ts
│   ├── requestWithUser.interface.ts
│   └── tokenPayload.interface.ts
```
As you can see from the `authentication` directory, a module can contain many things. In the above, it wraps the controller, service, and some other files connected to the authentication process.

During authentication, we also need to read and create users. To endcapsulate this process, we created a separate `UsersModule`. This shows that the modules are useful in dividing ourt app into pieces that work together.
Let's inspect how we can use the `UsersService` within the `AuthenticationModule`. To do so, we first need to look into the `UsersModule`.

>> users/users.module.ts
```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';

@Module({
    imports: [TypeOrmModule.forFeature([User])],
    providers: [UsersService],
    exports: [UsersService]
})
export class UsersModule {}
```
The crucial part here are the `providers` and `exports` arrays.

A provider is comething that can `inject` dependencies. An example of such is a service. We put the `UsersService` into the `providers` array of the `UsersModule` to state that it belongs to that module.
As you can see above, a module can also import other modules. By putting the `UsersService` into the `exports` array, we indicate that the module expose it. We can think of it as a public interface of a module.

>> authentication/authentication.module.ts
```typescript
import { Module } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';
import { UsersModule } from '../users/users.module';
import { AuthenticationController } from './authentication.controller';
import { LocalStrategy } from './local.strategy';
import { JwtStrategy } from './jwt.strategy';

@Module({
    imports: [
        UsersModule,
        //...
    ],
    providers: [AuthenticationService, LocalStrategy, JwtStrategy],
    controllers: [AuthenticationController]
})

export class AuthenticationModule {}
```
Now, when we import the `UsersModule`, we have access to all of the exported provides. Thanks to that, we can use `UsersService` within the `AuthenticationModule`.

The significant thing is that in NestJS, modules are **singletons**. It means that the instance of the `UsersService` is shared across all modules. The above would be especially crucial when considering techniques like in-memory catching.

## The @Global() decorator
Although creating global modules might be discouraged and perceived as a bad design decision, it definitely is possible.
When we want a set of providers to be available everywhere, we can use the `@Global()` decorator.

```typescript
@Global()
@Module({
    imports: [TypeOrmModule.forFeature([User])],
    providers: [UsersService],
    exports: [UsersService]
})
export class UsersModule {}
```
Now, we don't need to import the `UsersModule` to use the `UsersService`.
We should register a global module once, and the bes place for that is the root module.

# Summary
In this article, we've dug deeper into how NestJS works with modules and how it resolves their dependencies. We've inspected a bit on what principles the Dependency Injection works in NestJS. Learning about some of the inner mechanisms in the framework can help us to understand it better and, therefore, create a more sophisticated application structure.
