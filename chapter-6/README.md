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
