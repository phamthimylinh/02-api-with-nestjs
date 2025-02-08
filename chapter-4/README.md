NestJs shine when it comes to handling errors and validating data. A lot of that is thanks to using decorator. In this article, we go through features that NestJS provides us with, such as Exception filters and Validation pipes.

The code from this series results in [this repository](https://github.com/mwanago/nestjs-typescript). It aims to be an extended version of the [official Nest framework TypeScript starter](https://github.com/nestjs/typescript-starter).

# Exception filters
NestJS has an **exception filter** is call `BaseExceptionFilter`. We can look into the source code of NestJS and inspect its behavior.

>> nest/packages/core/exceptions/base-exception-filter.ts
```typescript
export class BaseExceptionFilter<T = any> implements ExceptionFilter<T> {
    // .....
    catch(exception: T, host: ArgumentsHost) {
        //.,,,
        if (!(exception instanceof HttpException)) {
            return this.handleUnknownError(exception, host, applicationRef)
        }
        const res = exception.getResponse();
        const message = isObject(res)
        ? res
        : {
            statusCode: exception.getStatus(),
            message: res,
        }
    }

    public handleUnknownError(
        exception: T,
        host: ArgumentsHost,
        applicationRef: AbstractHttpAdapter | HttpServer,
    ) {
        const body = {
            statusCode: HttpStatus.INTERNAL_SERVER_ERROR,
            message: MESSAGES.UNKNOWN_EXCEPTION_MESSAGE
        }
        //...
    }
}
```
Every time there is an error in our application, the `catch` method runs. There are a few essential things we can get from the above code

## HttpException
NestJS expects us to use the `HttpException` class. If we don't, it interpret the errors as unintentional and responds with **500 Internal Server Error**.

We've used `HttpException` quite a bit in the previous parts of this series:

```typescript
throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
```

The constructor takes two required arguments: the response body, and the status code. For the latter, we can use the provided `HttpStatus` enum.

If we provide a string as the definition of the response, NestJS serialized it into an object containing two properties:

- `statusCode`: contains the HTTP code that we've chose
- `message`: the description that we've provided

![response exception](https://wanago.io/wp-content/uploads/2020/05/Screenshot-from-2020-05-31-15-01-51.png)

> We can override the above behavior by providing an object as the first argument of the `HttpException` constructor.

We can often find ourseleves throwing similar exceptions more than once. To avoid code duplication, we can create custom exceptions. To do so, we need to extend the `HttpException` class.

>> posts/exception/postNotFund.exception.ts
```typescript
import { HttpException, HttpStatus } from '@nestjs/common';

class PostNotFoundException extends HttpException {
    constructor(postId: number) {
        super(`Post with id ${postId} not found`, HttpStatus.NOT_FOUND);
    }
}
```

Our custom `PostNotFoundException` calls the constructor of the `HttpException`. Therefore, we clean up our code by not having to define the message every time we want to throw an error.
NestJS has a set of exceptions that extends the `HttpException`. One of them is `NotFoundException`. We can refactor the above code and use it.

> We can find the full list of built-in HTTP exceptions in the documentation.

>> posts/exception/postNotFund.exception.ts

```typescript
import { NotFoundException } from '@nestjs/common';

class PostNotFoundException extends NotFoundException {
    constructor(postId: number) {
        super(`Post with id ${postId} not found`);
    }
}
```

The first argument of the `NotFoundException` class is an additional `error` property. This way, our `message` is defined by `NotFoundException` and is based on the status.
![response NotFoundException](https://wanago.io/wp-content/uploads/2020/05/Screenshot-from-2020-05-31-15-37-16.png)

## Extending the BaseExceptionFilter
The default `BaseExceptionFilter` can handle most of the regular cases. However, we might want to modifiy it in some way. The easies way to do so is to create a filter that extends it.
>> utils/exceptionsLogger.filter.ts
```typescript
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class ExceptionsLoggerFilter extends BaseExceptionFilter {
    catch(exception: unknow, host: ArgumentsHost) {
        console.log('Exception thrown', exception);
        super.catch(exception, host)
    }
}
```

The  `@Catch()` decorator means that we want our filter to catch all exceptions. We can provide it with a single exception type or a list.

The ArgumentsHost hives us access to the [**execution context** of the application](https://docs.nestjs.com/fundamentals/execution-context). We explore it in the upcoming parts of this series.

We can use our new filter in three ways. The first one is use it globally in all our routes through `app.useGlobalFilters`.

>> main.ts
```typescript
import { HttpAdapterHost, NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as cookieParser from 'cookie-parser';
import { ExceptionsLoggerFilter } from './utils/exceptionsLogger.filter';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);

    const { httpAdapter } = app.get(HttpAdapterHost);
    app.useGlobalFilters(new ExceptionsLoggerFilter(httpAdapter));

    app.use(cookieParser);
    await app.listen(3000);
}
```
A better way to inject our filter globally is to add it to our `AppModule`. Thanks to that, we could inject additional dependencies into our filter.

```typescript
import { Module } from '@nestjs/common';
import { ExceptionsLoggerFilter } from './utils/exceptionsLogger.filter';
import { APP_FILTER } from '@nestjs/core';

@Module({
    //...
    providers: [
        {
            provide: APP_FILTER,
            useClass: ExceptionsLoggerFilter,
        }
    ]
})
export class AppModule {}
```

The third way to bind filters is to attach the `@UseFilters` decorator. We can provide it with a single filter, or a list of them.
```TypeScript
@Get(':id')
@UseFilters(ExceptionsLoggerFilter)
getPostById(@Param('id') id: string) {
    return this.postsService.getPostById(Number(id));
}
```
The above is not the best approach to logging exceptions. NestJs has a built-in [Logger](https://docs.nestjs.com/techniques/logger) that we cover in the upcoming parts of this series.

## Implementing the ExceptionFilter Interface

If we need a fully customized behavior for errors, we can build our filter from scratch. It needs to implement the `ExceptionFilter` interface. Let's look into an example:
```typescript
import { ExceptionFilter, Catch, ArgumentsHost, NotFoundException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(NotFoundException)
export class HttpExceptionFilter implements ExceptionFilter {
    catch(exception: NotFoundException, host: ArgumentsHost) {
        const context = host.switchToHttp();
        const response = context.getResponse<Response>();
        const request = context.getRequest<Request>();
        const status = exception.getStatus();
        const message = exception.getMessage();

        response
        .status(status)
        .json({
            message,
            statusCode: status,
            time: new Date(.toISOString()),
        })
    }
}
```
There are a few notable things above. Since we use `@Catch(NotFoundException)`, this filter runs only for `NotFoundException`.
The `host.switchToHttp` method returns the `HttpArgumentsHost` object with information about the HTTP context. We explore it a lot in the upcoming parts of this series when discussing the [execution context](https://docs.nestjs.com/fundamentals/execution-context).

# Validation
We definitely should validate the upcoming data. In the [typescript express](http://wanago.io/2018/12/17/typescript-express-error-handling-validation/) series, we use the [class-validator library](https://www.npmjs.com/package/class-validator). NestJs also incorporates it.
NestJs comes with a set built-in **pipes**. Pipes are usually used to either transform the input data or validate it. Today we only use the predefinded pipes, but in the upcoming parts of this series, we might look into creating custom ones.

To start validating data, we need the `ValidationPipe`.
>> main.ts
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as cookieParser from 'cookie-parse';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    app.useGlobalPipes(new ValidationPipe());
    app.use(cookieParser());
    await app.listen(3000);
}

bootstrap();
```
In the [first part of this series](http://wanago.io/2020/05/11/nestjs-api-controllers-routing-module/), we've created Data Transfer Objects. The define the format of the data sent in a request. They are a perfact place to attach validation.

```typescript
npm install class-validator clas-transformer
```

> For the `ValidationPipe` to work we also need the [class-transformer](https://www.npmjs.com/package/class-transformer) library.

>>auth/dto/register.dto.ts

```typescript
import { IsEmail, IsString, IsNotEmpty, MinLength } from 'class-validator';

export class RegisterDto {
    @IsEmail()
    email: string;

    @IsString()
    @IsNotEmpty()
    name: string;

    @IsString()
    @IsNotEmpty()
    @MinLength(7)
    password: string;
}

export default ResgisterDto;
```

Thanks to the fact that we use the above `ResgisterDto` with the `@Body()` decorator, the `ValidationPipes` now checks the data.

```typescript
@Post('resgister')
async register(@Body() registrationData: RegisterDto) {
    return this.authenticationService.register(resgisterData);
}
```
![response validate](https://wanago.io/wp-content/uploads/2020/05/Screenshot-from-2020-05-31-18-56-58.png)

There are a lot more decorators that we can use. For a full list, check out the [class-validator documentation](https://github.com/typestack/class-validator). You can also [create custom validation decorators](https://github.com/typestack/class-validator#custom-validation-decorators).

## Validating params
We can also the class-validator library to validate params.
>> utils/findOneParams.ts
```typescript
import { IsNumberString } from 'class-validator';
class FindOneParams {
    @IsNumberString()
    id: string;
}
```
```typescript
@Get(':id')
getPostById(@Param() { id }: FindOneParams) {
    return this.postsService.getPostById(Number(id));
}
```
Please note that we don't use `@Param(':id')` anymore here. Instead, we destructure the whole params object.

> If you use MongoDB instead of Postgres, the `@IsMongoId` decorator might prove to be useful for you here.

# Handling PATCH
In the [Typescript Express series](http://wanago.io/2020/04/27/typescript-express-put-vs-patch-mongodb-mongoose/), we discuss the difference between the PUT and PATCH method. Summing it up, PUT replaces an entity, while PATCH applies a partial modification. When performing partial changes, we need to skip missing properties.

The most straightforward way to handle PATCH is to pass `skipMissingProperties` to our `ValidationPipe`.

```typescript
app.useGlobalPipes(new ValidationPipe({ skipMissingProperties: true }))
```

Unfortunately, this would skip missing properties in all of our DTOs. We don't want to do that when posting data. Instead, we could add `IsOptional` to all properties when updating data.
```typescript
import { IsString, IsNotEmpty, IsNumber, IsOptional } from 'class-validator';

export class UpdatePostDto {
    @IsString()
    @IsOptional()
    id: number;

    @IsString()
    @IsNotEmpty()
    @IsOptional()
    content: string;

    @IsString()
    @IsNotEmpty()
    @IsOptional()
    title: string;
}
```
Unfortunately, the above solution is not very clean. There are some solutions provided to override the default behavior of the `ValidationPipe` [here](https://github.com/nestjs/nest/issues/2390).
> In the upcoming parts of this series we look into how we can implement PUT instead of PATCH

# Summary
In this article, we've looked into how error handling and validation works in NestJS. Thanks to looking into how the default `BaseExceptionFilter` works under the hood, we now know how to handle various exceptions properly. We know also know how to change the default behavior if there is such a need.
We've also how to use the `ValidationPipe` and the class-validator library to validate incoming data.

There is still a lot to cover in the NestJS framework, so stay tuned!.
