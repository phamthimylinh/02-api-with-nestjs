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


























