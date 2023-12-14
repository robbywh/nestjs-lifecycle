---
theme: penguin
colorSchema: 'auto'
layout: intro
# https://sli.dev/custom/highlighters.html
highlighter: prism
title: NestJS Lifecycle
---


# The NestJS Lifecycle

KSI sharing knowledge

<div class="pt-12">
  <span @click="next" class="px-2 p-1 rounded cursor-pointer hover:bg-white hover:bg-opacity-10">
    Press Space for next page <carbon:arrow-right class="inline"/>
  </span>
</div>

---
layout: text-image
media: ../images/lifecycle.png
caption: NestJS Lifecycle
---

# The lifecycle steps
Here’s the order of the lifecycle steps
1. Request comes in
2. Middlewares
3. Guards
4. Interceptors - Before Handler 
5. Pipes
6. Route Handler
7. Interceptors - After Handler
8. Exception Filters
9. Response

---
layout: text-window
---

# Middleware

Middleware serves as the first point of entry in the request lifecycle and allows developers to execute arbitrary code before the request is processed by the application.


::window::

```ts
import { Injectable, NestMiddleware } from '@nestjs/common'
import { Request, Response, NextFunction } from 'express'

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`Middleware: Logging request: ${req.url}`)
    next()
  }
}
```

---
layout: text-window
---

# Middleware

How to apply the Middleware ?


::window::

```ts {2,16-18}
// main.ts
app.use(LoggerMiddleware);

// app.module.ts
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { LoggerMiddleware } from './common/logger/logger.middleware';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes(`*`);
  }
}
```

---
layout: text-window
---

# Guards

Guards are used to protect routes and endpoints from unauthorized access. They can be used to enforce authentication, authorization, and other security constraints.


::window::

```ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common'
import { Observable } from 'rxjs'

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean> | Observable<boolean> {
    console.log(`Guard: Checking authentication`)
    const request = context.switchToHttp().getRequest()
    const apiKey = request.headers['x-api-key'];
    if (apiKey !== 'SECRET') {
      return false
    }
    console.log(`Guard: Passed authentication`)
    return true
  }
}
```

---
layout: text-window
---

# Guards

How to apply the Guards ?


::window::

```ts {2,9-12,20}
// main.ts
 app.useGlobalGuards(new AuthGuard())

// app.module.ts
@Module({
  controllers: [AppController],
  providers: [
    AppService,
    {
      provide: APP_GUARD,
      useClass: AuthGuard,
    },
  ],
})
export class AppModule implements NestModule {
}

// app.controller.ts
@Controller()
@UseGuards(AuthGuard)
export class AppController {}
```

---
layout: text-window
---

# Interceptors - Before Handler

If the request passes the guards, interceptors come into play. At this stage, interceptors can manipulate the request before it reaches the route handler method.

::window::

```ts
import { CallHandler, ExecutionContext, Injectable, NestInterceptor } from '@nestjs/common'
import { Observable } from 'rxjs'

@Injectable()
export class BrowserInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const browser = request.header(`user-agent`) || `Unknown`;
    request.headers.browser = browser.split(` `)[0];
    console.log(
      `Interceptor: manipulated request with new browser header: ${request.headers.browser}`,
    );
    return next.handle();
  }
}
```

---
layout: text-window
---

# Interceptors - Before Handler

How to apply the Interceptors - Before Handler ?


::window::

```ts {2,9-12,20}
// main.ts
app.useGlobalInterceptors(new BrowserInterceptor());

// app.module.ts
@Module({
  controllers: [AppController],
  providers: [
    AppService,
    {
      provide: APP_INTERCEPTOR,
      useClass: BrowserInterceptor,
    },
  ],
})
export class AppModule implements NestModule {
}

// app.controller.ts
@Controller()
@UseInterceptor(BrowserInterceptor)
export class AppController {}
```

---
layout: text-window
---

# Pipes

Pipes run after the incoming interceptor and just before the route handler.
Pipes are used for validation and transformation of incoming request data.

::window::

```ts
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';

@Injectable()
export class EmojiValidationPipe implements PipeTransform {
  transform(value?: string) {
    if (!value) return;
    console.log(`Pipe: pre-validation value: ${value}`);
    const index = parseInt(value, 10);

    if (isNaN(index)) {
      throw new BadRequestException('Emoji index must be a number.');
    }

    if (index < 0 || index >= 14) {
      // As there are 14 emojis in your list.
      throw new BadRequestException('Emoji index out of range.');
    }
    console.log(`Pipe: post-validation value: ${value}`);
    return index;
  }
}
```

---
layout: text-window
---

# Pipes

How to apply the Pipes ?

::window::

```ts {2,7-10,18,23}
// main.ts
app.useGlobalPipes(new EmojiValidationPipe());

// app.module.ts
@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: EmojiValidationPipe,
    },
  ],
})
export class AppModule implements NestModule {
}

// app.controller.ts
@Controller()
@UsePipes(new EmojiValidationPipe())
export class AppController {
  @Get()
  getEmoji(
    @Req() request: Request,
    @Query('index', EmojiValidationPipe) index?: number,
  )
}
```

---
layout: text-window
---

# Route Handler

The next step in the NestJS lifecycle is the route handler. This is the actual route handler (e.g., the controller method) that executes, handles the request, and generates a response.


::window::

```ts
  @Get()
  getEmoji(
    @Req() request: Request,
    @Query('index', EmojiValidationPipe) index?: number,
  ) {
    return {
      browser: request.headers.browser,
      emoji: this.appService.getEmoji(index),
    };
  }

  @UseInterceptors(ClassSerializerInterceptor)
  @Post(':indexParams')
  postEmoji(
    @Body() payload: PostEmojiRequestDTO,
    @Param() params: PostEmojiParamsRequestDTO,
  ): PostEmojiResponseDTO {
    return new PostEmojiResponseDTO({
      payload,
      params,
    });
  }
```

---
layout: text-window
---

# Interceptors - After Handler

After the route handler has processed the request, interceptors can again manipulate the response before it’s sent back to the client.

This is really useful if you need to do some consistent formatting of the response before it’s sent back to the client, for example, wrapping any data returned from the response in a data object.

::window::

```ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class TransformResponseInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => {
        console.log(`Interceptor - after handler: manipulate response`);
        return { data };
      }),
    );
  }
}

```

---
layout: text-window
---

# Interceptors - After Handler

How to apply the Interceptors - After Handler ?


::window::

```ts {2,9-12,20}
// main.ts
app.useGlobalInterceptors(new TransformResponseInterceptor());

// app.module.ts
@Module({
  controllers: [AppController],
  providers: [
    AppService,
    {
      provide: APP_INTERCEPTOR,
      useClass: TransformResponseInterceptor,
    },
  ],
})
export class AppModule implements NestModule {
}

// app.controller.ts
@Controller()
@UseInterceptor(TransformResponseInterceptor)
export class AppController {}
```

---
layout: text-window
---

# Exception Filters

If an exception (i.e. an error) is thrown at any stage of the lifecycle, exception filters catch them and provide a way to handle errors gracefully.

They allow you to implement a centralized error-handling mechanism, which can set the appropriate response codes and format error messages consistently.

NestJS has a built-in exception filter that handles exceptions thrown by the application. It returns a 500 Internal Server Error response by default, but throws the respective HTTP exception if a built-in HTTP exception is thrown.

::window::

```ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: HttpException | unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const isHttpException = exception instanceof HttpException;
    const status = isHttpException ? exception.getStatus() : 500;

    response.status(status).json({
      message: isHttpException ? exception.message : `Internal server error`,
      error: isHttpException ? exception.name : `GenericException`,
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

---
layout: text-window
---

# Exception Filters

How to apply the Exception Filters ?


::window::

```ts {2,9-12,20}
// main.ts
app.useGlobalFilters(new AllExceptionsFilter());

// app.module.ts
@Module({
  controllers: [AppController],
  providers: [
    AppService,
    {
      provide: APP_FILTER,
      useClass: AllExceptionsFilter,
    },
  ],
})
export class AppModule implements NestModule {
}

// app.controller.ts
@Controller()
@UseFilters(AllExceptionsFilter)
export class AppController {}
```

---
layout: intro 
---

# Dependency Injection

Dependency injection is a design pattern that helps to decouple the components of an application and increase its maintainability, scalability, and testability. NestJS provides an implementation of the dependency injection pattern through its built-in Dependency Injection (DI) Container.

---
layout: text-window
---

# Dependency Injection

To use dependency injection in NestJS, you need to follow these steps:
1. Create a provider. You can create a provider by adding the @Injectable() decorator to a class.
2. Register the provider. You can do this by adding the provider to the providers array of a module.
3. Inject the provider. You can do this by adding the provider as a constructor parameter.

::window::

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class MyService {
  doSomething(): string {
    return 'Something was done';
  }
}

@Module({
  controllers: [MyController],
  providers: [MyService],
})
export class MyModule {}

@Controller()
export class MyController {
  constructor(private readonly myService: MyService) {}

  @Get()
  getSomething(): string {
    return this.myService.doSomething();
  }
}

```

---
layout: text-window
---

# Dependency Injection

While using the main.ts is convenient for setting middleware, interceptors, guards and filters globally, it has a limitation: these elements won’t have access to NestJS’s Dependency Injection (DI) system.

Here’s why:

In NestJS, the DI container handles the lifecycle of all the providers, meaning it’s responsible for their instantiation and destruction. When we define our guards, interceptors, and filters in main.ts, they are not part of this DI container. As a result, we cannot inject any other service or provider into them.

::window::

```ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class LoggerService {
  info(log: string) {
    console.log(log);
  }
}

import { LoggerService } from '../../logger.service';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly loggerService: LoggerService) {}
}
```

---
class: 'grid text-center align-self-center justify-self-center'
---

# Thank you
