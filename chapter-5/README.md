Sometimes we need to perform additional operations on the outcoming data. We might not want to expose specific properties or modify the response in some other way. In this article, we look into various solutions NestJS provides us with to change the data we send in the response.

You can find the code from in [this repository](https://github.com/mwanago/nestjs-typescript)

# Serialization
The first thing to look into is the **serialization**. It is a procees of transforming the response data before returning it to the user.

In the previous parts of this series, we've removed the password in the various parts of our API. A better approach would be using the [class-transformer](https://github.com/typestack/class-transformer).

>> users/user.entity.ts
```typescript
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';
import { Exclude } from 'class-transformer';

@Entity()
class User {
    @PriamryGeneratedColumn()
    public id?: number;

    @Column({ unique: true })
    public email: string;

    @Column()
    @Exclude()
    public password: string;
}

export default User;
```

NestJS comes equipped with `ClassSerializerInterceptor` that uses [class-transformer](https://github.com/typestack/class-transformer) under the hood. To apply the above transformation, we need to use it in our controller.

```typescript
@Controller('authentication')
@UseInterceptors(ClassSerializerInterceptor)
class AuthenticationController
```
If we find ourselves adding `ClassSerializerInterceptor` to a lot of controllers, we can configure it globally instead.

>> main.ts

```typescript
import { NestFactoru, Reflector } from '@nestjs/core';
import { AppModule } from './app.module';
import * as cookieParser from 'cookie-parser';
import { ClassSerializerInterceptor } from '@nestjs/common';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    app.useGlobalPipes(new ValidationPipe());
    app.useGlobalInterceptors(new ClassSerializerInterceptor(
        app.get(Reflector)
    ));
    app.use(cookieParser());
    await app.listen(300);
}
boostrap();
```
> The `ClassSerializerInterceptor` needs the [Reflector](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata) when initializing.

By default, all properties of our entities are exposed. We can change this strategy by providing additional options to the class-transformer.
To do so, we need to use the `@SerializeOptions()` decorator.

```typescript
@Controller('authentication')
@SerializeOptions({
    strategy: 'excludeAll'
})
export class AuthenticationController
```

>> users/user.entity.ts

```typescript
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';
import { Expose } from 'class-transformer';

@Entity()
class User() {
    @PrimaryGeneratedColumn()
    public id?;

    @Column({ unique: true })
    @Expose()
    public email: string;

    @Column()
    @Expose()
    public name: string;

    @Column()
    public password: string
}

export default User;
```
![response with custom response with serialization](https://wanago.io/wp-content/uploads/2020/06/Screenshot-from-2020-06-07-11-15-18.png)

The `@SerializeOption()` decorator has more options that you might find useful. It matches the options that you can provide for the `classToPlain` method in the [class-transformer](https://github.com/typestack/class-transformer).
The class-transformer library has quite a few useful features. Another noteworthy one is the ability to transform values. To demonstrate it, let's create a *nullable* column:
```typescript
@Entity()
class Post {
    //...
    @Column({ nullable: true })
    public category?: string;
}
```

Since the `category` is a nullable column, it is optional, its value is null until we set it. This means sending null values in the response:
![null able response with class transformer](https://wanago.io/wp-content/uploads/2020/06/Screenshot-from-2020-06-07-12-25-27.png)
The above behavior might be considered undesirable and the most straightforward way to fix it is use the `@Transform` decorator. If the value equals null, we don't want to send in the response.
```typescript
@Column({ nullable: true })
@Transform(value => {
    if(value !== null) {
        return value;
    }
})
public category?: string;
```
## Issue with using the @Res() decorator
In the [previous part of this series](http://wanago.io/2020/05/25/api-nestjs-authenticating-users-bcrypt-passport-jwt-cookies/), we've used the `@Res()` decorator to access the Express Response object. Thanks to that, we were able to attach cookies to the response.
```typescript
@HttpCode(200)
@UseGuards(LocalAuthenticationGuard)
@Post('log-in')
async logIn(@Req() request: RequestWithUser, @Res() response: Response) {
    const {user} = request;
    const cookie = this.authenticationService.getCookieWithJwtToken(user.id);
    response.setHeader('Set-Cookie', cookie);
    user.password = undefined;
    return respose.send(user);
}
```
Using the `@Res()` decorator strips us from some advantages of using NestJS. Unfortunately, it interferes with the `ClassSerializerInterceptor`. To prevent that, we can follow some [advice from the creator of NestJS](). If we use the `request.res` object instead of the `@Res()` decorator, we don't put NestJS into the express-specific mode.

```typescript
@HttpCode(200)
@UseGuards(LocalAuthenticationGuard)
@Post('log-in')

async logIn(@Res() request: RequestWithUser) {
    const {user} = request;
    const cookie = this.authenticationService.getCookieWithJwtToken(user.id);
    request.res.setHeader('Set-Cookie', cookie);
    return user;
}
```
The above is a neat little trick that we use to take advantage of the mechanisms built into NestJS while accessing the Response object directly.


# Custom interceptors

Above, we use the `@Transform` decorator to skip a single property if it equals null. Doing so for every nullable property does not seem like a clean approach.






