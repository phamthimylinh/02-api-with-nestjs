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
