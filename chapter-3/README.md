Authentication is a crucial part of almost every web application. There are many ways to approach it, and we've handled it manually in our [Typesript Express series](http://wanago.io/2018/12/24/typescript-express-registering-authenticating-jwt/). This time we look into the **passport**, which is the most popular Node.js authentication library. We also register users and make their passwords secure by **hashing**.

You can find all of the code from this series in [this repository](https://github.com/mwanago/nestjs-typescript). Feel free to give it a star.

# Defining the User entity
The first thing to do when considering authentication is to **register** our users. To do so, we need to define an entity for our users.

> users/user.entity.ts
```typescript
import {Column, Entity, PrimayGenaratedColumn } from 'typeorm';

@Entity()
class User {
  @PrimayGenaratedColumn()
  public id?: number;

  @Column({ unique: true})
  public email: string;

  @Column()
  public name: string;

  @Column()
  public password: string;
}

export default User;
```
The only new thing above is the **unique** flag. It indicates that there should not be two users with the same email.
This  functionally is built into PostgreSQL and helps us to keep consistency of our data. Later, we depend on emails being unique when authenticating.

We need to perform a few operations on our users. To do so, let's create a service.

> users/users.service.ts
```typescript
import { HttpException, HttpStatus, injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import User from './user.entity';
import CreateUserDto from './dto/createUser.dto';

@injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>
  ) {}

  async getByEmail(email: string) {
    const user = await this.usersRepository.findOne({ email });
    if (user) {
      return user;
    }
    throw new HttpException('User with this email does not exist', HttpStatus.NOT_FOUND);
  }

  async create(userData: CreateUserDto) {
    const newUser = await this.usersRepository.create(userData);
    await this.usersRepository.save(newUser);
    return newUser;
  }
}
```
> users/dto/createUser.dto.ts
```typescript
export class CreateUserDto {
  email: string;
  name: string;
  password: string;
}

export default CreateUserDto;
```
All of the above is wrapped using a module.

> users/users.module.ts

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';
import { TypeOrmModule } from '@nestjs/typeorm';
import User from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])]
  providers: [UsersService],
  exports: [UsersService]
})

export class UsersModule {}
```

# Handling passwords
An essential thing about registration is that we don't want to save passwords in plain text. If at any time our database gets breached, our passwords would have been directly exposed.

To make passwords more secure, we **hash** them. In this process, the hashing algorithmn transforms one string into another string. If we change just one character of a string, the outcome is entirely different.

The above operation can only be performed one way and can't be reversed easily. This means that we don't know the passwords of our users. When the user attempts to log in, we need to perform this operation once again. Then, we compare the outcome with the one saved in the database.

Since hashing the same string twice gives the result, we use **salt**. It prevents users that have the same password from having the same hash. Salt is a random string added to the original password to achieve a different result every time.

## Using bcrypt
We use the **bcrypt** hashing algorithmn implemented by the [bcrypt npm package](https://www.npmjs.com/package/bcrypt). It takes care of hashing the strings, comparing plain string with hashes, and appending salt.

Using **bcrypt** might be an intensive task for CPU. Fortunately, our bcrypt implementation uses a thread pool that allows it run in an additional thread. Thanks to that, our application can perform other tasks while genarating the hash.

> npm install @type/bcrypt bcrypt

When we use bcrypt, we define **salt rounds**. It boils down to being a cost factor and controls the time needed to receive a result.
Increasing it by one doubles the time. The bigger the cost factor, the more difficult it is to reverse the hash with brute-forcing. Generally speaking, 10 salt rounds should be fine.

The **salt** use for hashing is a part of the result, so no need to keep it separately

```typescript
const passwordInPlainText = '12345678';
const hashedPassword = await bcrypt.hash(passwordInPlainText, 10);

const isPasswordMatching = await bcrypt.compare(passwordInPlainText, hashedPassword);
console.log(isPasswordMatching) // true
```

## Creating is the authentication service
With all of the above knowledge, we can start implementing basic registering and logging in functionalities. To do so, we need to define an authentication service first.

> **Authentication** means checking the identity of user. It providers an answer to a question: who is the user?

> **Authentication** is about access to resources. It answers the question: is user authorized to perform this operation?

> authentication/authentication.service.ts

```typescript
export class AuthenticationService {
  constructor(
    private readonly usersService: UsersService
  ) {}

  public async register(registrationData: RegisterD) {
      const hashedPassword = await bcrypt.hash(registrationData.password, 10);
      try {
        const createdUser = await this.usersService.create({
          ...registrationData,
          password: hashedPassword
        });
        createdUser.password = undefined;
        return createdUser;
      } catch(error) {
        if (error?.code === PostgreErrorCode.UniqueViolation) {
          throw new HttpException('User with that email already exists', HttpStatus.BAD_REQUEST);
        }
        throw new HttpException('Something went wrong', HttpStatus.INTERNAL_SERVER_ERROR);
      }
  }
}
```

> `createdUser.password = undefined` is not the cleanest way to not send the password in a response. In the upcoming parts of this series we explore mechainsms that help us with that.

A few notable things are happening above. We create a hash and pass it to the `usersService.create` method along with the rest of the data. We use a `try ... catch` statement here because there is an important case when it might fail.
If a user with that email already exist, the `usersService.create` method throws an error.
Since our unique column cases it the error comes from Postgres.

To understand the error, we need to look into the [PostgreSQL Error Codes documentation page](https://www.postgresql.org/docs/9.2/errcodes-appendix.html). Since the code for **uniqe_violation** is **23505**, we can create an enum to handle it cleanly.

> database/postgresErrorCodes.enum.ts
```typescript
enum PostgreErrorCode {
  UniqueViolation = '23505';
}
```

> Since in the above service we state explicitly that a user with this email already exists, it might a good idea to implement a mechainsms preventing attackers from brute-forcing our API in order to get a list registered emails

The thing left for us to do is to implement the logging in.

> authentication/authentication.service.ts

```typescript
export class AuthenticationService {
  constructor(
    private readonly usersService: UsersService
  ) {}

  public async getAuthenticatedUser(email: string, hashedPassword: string) {
    try {
      const user = await this.usersService.getByEmail(email);
      const isPasswordMatching = await bcrypt.compare(
        hashedPassword,
        user.password
      )
      if(!isPasswordMatching) {
        throw new HttpException('Wrong credentials provided', HttpStatus.BAD_REQUEST);
      }
      user.password = undefined;
      return user;
    } catch(err) {
      throw new HttpException('Wrong credentials provided',HttpStatus.BAD_REQUEST);
    }
  }
}
```

An import thing above is that we return the same error, whether the email or password is wrong. Doing so prevents some attacks that would aim to get a list of emails registered in our database.

There is one small thing about the above code that we might want to improve. Within our `logIn` method, we throw an exception that we then catch locally. It might be considered confusing. Let's create a separate method to verify the password:
```typescript
public async getAuthenticatedUser(email: string, plainTextPassword: string)
{
  try {
    const user = await this.usersService.getByEmail(emai);
    await this.verifyPassword(plainTextPassword, user.password);
    user.password = undefined;
    return user;
  } catch (err) {
    throw new HttpException('Wrong credentials provided', HttpStatus.BAD_REQUEST);
  }
}

private async verifyPassword(plainTextPassword: string, password: string)
{
  const isPasswordMatching = await bcrypt.compare(
    plainTextPassword,
    password
  )

  if (!isPasswordMatching) {
    throw new HttpException('Wrong credentials provided', HttpStatus.BAD_REQUEST);
  }
}
```
# Integrating our authentication with passport
In the [Typescript Express series](http://wanago.io/2018/12/24/typescript-express-registering-authenticating-jwt/), We've handled the whole authentication process manually. [NestJS documentation](https://docs.nestjs.com/techniques/authentication) suggests using the Passport library and provides us with the means to do so. Passport gives us an abstraction over the authentication, thus relieving from some heavy lifting. Also, it is heavy tested in production by many developers.
> Diving into how to implement the authentication manually without Passport is still a good idea. By doing so, we can get an even better understanding of this process.

Applications have different approaches to authentication. Passport calls those mechainsms **strategies**.
The first strategy that we want to implement is the **passport-local** strategy. It is a strategy for authenticating with a username and password

> npm install @nest/passport passport @type/passport-local passport-local @type/express

To configure a strategy, we need to provide a set of options specific to a particular strategy. In NestJS, we do it by extending the `PassportStrategy` class.
> authentication/authentication.ts

```typescript
import { Stategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { injectable } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';
import User from '../users/user.entity';

@injectable()
export class LocalStategy extends PassportStrategy(Stategy) {
  constructor(private authenticationService: AuthenticationService) {
    super({
      usernameField: 'email'
    })
  }

  async validate(email: string, password: string): Promise<User> {
    return this.authenticationService.getAuthenticatedUser(email, password);
  }
}
```
For very strategy, Passport calls the `validate` function using a set of parameter specific for a particular stategy. For the local stategy, Passport needs a method with username and a password.

We also need to configure our `AuthenticationModule` to use passport

> authentication/authentication.module.ts
```typescript
import { Module } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';
import { UsersModule } from '../user/users.module';
import { AuthenticationController } from './authentication.controller';
import { PassportModule } from '@nestjs/passport';
import { LocalStategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthenticationService, LocalStategy],
  controllers: [AuthenticationController]
})
export class AuthenticationModule {}
```

## Using built-in Passport Guards
The above module uses the [Guards](https://docs.nestjs.com/guards). Guard is reposible for determining whether the route handler handles the request or not. In its nature, it is similar to **Express.js middleware** but is more powerful.

> We focus on guards quite a bit in the upcoming parts of this series and create custom guards. Today we only use the existing guards though.

> authentication/authentication.controller.ts

```typescript
import { Body, Req, Controller, HttpCode, Post, UseGuards } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';
import RegisterDto from './dto/register.dto';
import RequestWithUser from './requestWithUser.interface';
import { LocalAuthenticationGuard } from './localAuthentication.guard';

@Controller('authentication')
export class AuthenticationController {
  constructor(
    private readonly authenticationService: AuthenticationService
  ) {}

  @Post('register')
  async register(@Body() registrationData: RegisterDto) {
    return this.authenticationService.register(registrationData);
  }

  @HttpCode(200)
  @UseGuards(LocalAuthenticationGuard)
  @Post('log-in')
  async logIn(@Req() request: RequestWithUser) {
    const user = request.user;
    user.password = undefined;
    return user;
  }
}
```

> Above we use `@HttpCode(200)` because NestJS responds with **201 Created** for **POST** requests by default

> authentication/localAuthentication.guard.ts
```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthenticationGuard extends AuthGuard('local') {}
```
>> Passing the strategy name directly into `AuthGuard()` in the controller might not be considered a clean approach. Instead, we create our own class.

> authentication/requestWithUser.interface.ts
```typescript
import { Request } from 'express';
import User from '../users/user.entity';

interface RequestWithUser extends Request {
  user: User;
}
 
export default RequestWithUser;
```
Thanks to doing all of the above, our `log-in` oute is handled by Passport.  The data of the user is attached to the `request` object, and this is why we extend the `Request` interface.

If the user authenticates successfully, we return his data. Otherwise, we throw an error.

# Using JSON Web Tokens
We aim to restrict some parts of application. By doing so, only authenticated users can access them. We don't want to need to authenticate for every request. Instead, we need a way to let the users indicate that they have already logged in successfully.

A simple way to do so is use JSON Web Tokens. **JWT** is a string that is created on our server using a secret key, and only we can decode it. We want to give it to the user upon logging in so that it can be sent back on every request. If the token is valid, we can trust the identity of the user.

```typescript
npm install @nestjs/jwt passport-jwt @types/passport-jwt cookie-parser @types/cookie-parser
```
The first thing to do is to add two new environment variables: `JWT_SECRET` and `JWT_EXPIRATION_TIME`.

We can use any thing as a JWT secret key. It is important to keep it secret and not to share it. We use it to encode and decode tokens in your application.

We describe our expiration time in seconds to increase security. If someone's token is stolen, the attacker has access to the application in a similar way to having a password. Due to the expiry time, the issue is partially dealt with because the token will expire.

>> app.module.ts
```typescript
ConfigModule.forRoot({
  validationSchema: Joi.object({
    //...
    JWT_SECRET: Joi.string().required(),
    JWT_EXPIRATION_TIME: Joi.string().required(),
  })
})
```

## Generating tokens
In this article, we want to the users to store the JWT in **Cookies**. It has a certain advantage over storing tokens in the web storage thanks to the HttpOnly directive. It can't be accessed directly through JavaScript in the browser, making it more secure and resistant to attacks like [cross-site scripting](https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)).

> If you want to know more about cookies, check out [Cookie: explaining document.cookie and the Set-cookie header](http://wanago.io/2018/06/18/cookies-explaining-document-cookie-and-the-set-cookie-header/)

Now, let's configure the `JwtModule`.

>> authentication/authentication.module.ts

```typescript
import { Module } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';
import { UsersModule } from '../users/users.module';
import { authenticationController } from './authentication.controller';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    UserModule,
    PassportModule,
    ConfigModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: async (configService: ConfigService) => ({
        secret: configService.get('JWT_SECRET'),
        signOptions: {
          expiresIn: `${configService.get('JWT_EXPIRATION_TIME')s}`,
        }
      })
    })
  ],
  provides: [AuthenticationService, LocalStategy],
  controllers: [AuthenticationController]
})
export class AuthenticationModule {}
```
Thanks to that, we can now uwe `JwtService` in our `AuthenticationService`.

>> authentication/authentication.service.ts

```typescript
@Injectable()
export class AuthenticationService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService,
  ) {}

  //...
  public getCookieWithJwtToken(userId: number) {
    const payload: TokenPayload = { userId };
    const token = this.jwtService.sign(payload);
    return `Authentication=${token}; HttpOnly; Path=/; Max-Age=${this.configService.get('JWT_EXPIRATION_TIME')}`;
  }
}
```

>> authentication/tokenPayload.interface.ts
```typescript
interface TokenPayload {
  userId: number;
}
```

We need to send the token created by `getCookieJwtToken` method when the user logs in successfully. We do it by sending the Set-Cookie header. To do so, we need to directly use the `Respone` object.

```
@HttpCode(200)
@UseGuards(LocalAuthenticationGuard)
@Post('log-in')
async logIn(@Req() request: RequestWithUser, @Res() response: Respone) {
  const {user} = request;
  const cookie = this.authenticationService.getCookieWithJwtToken(user.id);
  response.setHeader('Set-Cookie', cookie);
  user.password = undefined;
  return response.send(user);
}
```
When the browser receives this response, it sets the cookie so that it can use it later.

## Receiving tokens
To be able to read cookies easily we need the `cookie-parser`.

>> main.ts
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app,pdule';
import * as cookieParser from './cookie-parser';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(cookieParser());
  await app.listen(3000);
}
bootstrap();
```

Now, we need to read the token from the Cookie header when the user requests data. To do so, we need a second **passport strategy**.

>> authentication/jwt.strategy.ts

```typescript
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Request } from 'express';
import { UsersService } from '../users/users.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Stategy) {
  constructor(
    private readonly configService: ConfigService,
    private readonly usersService: UsersService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromExtractors([(request: Request) => {
        return request?.cookies?.Authentication;
      }]),
      secretOrKey: configService.get('JWT_SECRET')
    })
  }

  async validate(payload: TokenPayload) {
    return this.usersService.getById(payload.userId);
  }
}
```

There are a few notable things above. We extend the default JWT strategy by reading the token from the cookie.

When we successfully access the token, we use the id of the user that is encode inside. With it, we can get the whole user data through the `userService.getById` method. We also need to add it to our `UserService`.

>> users/users.service.ts
```
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>
  ) {}

  async getById(id: number) {
    const user = await this.usersRepository.findOne({ id });
    if (user) {
      return user;
    }
    throw new HttpException('User with this id does not exist', HttpStatus,NOT_FOUND);
  }
}
```
Thanks to the `validate` method running under the hood when the token is encoded, we have access to all of the user data.

We now need to add our new `JwtStrategy` to the `AuthenticationModule`.

>> authentication/authentication.module.ts

```
@Module({
  // ...
  providers: [AuthenticationService, LocalStrategy, JwtStrategy]
})
export class AuthenticationModule {}
```
## Requiring authentication from our users
Now, we can require our users to authenticate when send requests to our API. To do so, we first need to create our `JwtAuthenticationGuard`.

>> authentication/jwt-authentication.guard.ts
```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export default class JwtAuthenticationGuard extends AuthGuard('jwt') {}
```

Now, we can use it every time we want ourt users to authenticate before making a request. For example, we might want to do so, when creating posts through our API.

>> posts/posts.controller.ts
```
import { Body, Controller Post, UserGuard } from '@nestjs/common';
import PostsService from './posts.service';
import CreatePostDto from './dto/createPost.dto';
import JwtAuthenticationGuard from './authentication/jwt-authentication.guard';

@Controller('posts')
export default class PostController {
  constructor(
    private readonly postsService: PostsService
  ) {}

  @Post()
  @UseGuards(JwtAuthenticationGuard)
  async createPost(@Body() post: CreatePostDto) {
    return this.postsService.createPost(post);
  }

  //...
}
```
# Logging out
JSON Web Tokens are stateless. We can't change a token to be invalid in a straightforward way. The easily way to implement logging out is just to remove the token from the browser. Since the cookies that we designed are `HttpOnly`, we need to create an endpoint that clears it.

>> authentication/authentication.service.ts

```typescript
export class AuthenticationService {
  //...

  public getCookieForLogOut() {
    return `Authentication=; HttpOnly; Path=/; Max-Age=0`;
  }
}
```

>> authentication/authentication.controller.ts

```
@Controller('authentication')
export class AuthenticationController {
  // ...
  @UserGuard(JwtAuthenticationGuard)
  @Post('log-out')
  async logOut(@Req() request: RequestWithUser, @Res() response: Respone) {
    response.setHeader('Set-Cookie', this.authenticationService.getCookieForLogOut());
    return response.sendStatus(200);
  }
}
```

# Verifying tokens
One important additional functionally that we need to is verifying JSON Web Tokens and returning user data. By doing so, the browser can check if the current token is valid and get the data of the currently logged in user.
```
@Controller('authentication')
export class AuthenticationController {
  // ...
  @UserGuard(JwtAuthenticationGuard)
  @Get()
  authenticate(@Req() request: RequestWithUser) {
    const user = request.user;
    user.password = undefined;
    return user;
  }
}
```
![authenticate](https://wanago.io/wp-content/uploads/2020/05/Screenshot-from-2020-05-25-00-52-56.png)

# Summary
In this article, we're covered registering and logging in users in NestJS. To implement it, we've used bcrypt to has passwords to sercure them. To authenticate users, we've used JSON Web Tokens. There are still ways to improve the above features. For example, we should exclude passwords more cleanly. Also, we might want to might want to implement the token refreshing functionality. Stay tuned for more article about NestJS!
