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
