---
title: "NestJS Request Logger"
date: 2021-05-22
draft: false
---

Je partage ici un retour d'expérience que j'ai fait à mon travail chez [Koala](https://www.hikoala.co/).

## Les besoins

Je cherche à avoir un HTTP Logger qui trace les HTTP Requests et les HTTP Responses.

Je veux pouvoir logger aussi bien les requests 404 (aucune route définies) que les requête qui succeed ou même celles qui throw une erreur.

## Comment ça marche aujourd'hui

Aujourd'hui on a un `LoggerInterceptor` qui intercepte la requête pour les logger, et attend la réponse pour la logger aussi.

```typescript
@Injectable({ scope: Scope.REQUEST })
export class LoggerInterceptor implements NestInterceptor {
  public constructor(private readonly httpLoggerService: HttpLoggerService) {}

  public intercept(
    context: ExecutionContext,
    next: CallHandler<unknown>,
  ): Observable<unknown> | Promise<Observable<unknown>> {
    const httpContext = context.switchToHttp();
    const { originalUrl: url, method, params, query, body, headers, ip } = httpContext.getRequest<Request>();
    const response = httpContext.getResponse<Response>();

    const obfuscatedHeaders = obfuscateHeaders(headers);

    this.httpLoggerService.logRequest({ url, method, params, query, headers: obfuscatedHeaders, body, ip });

    return next.handle().pipe(
      tap((responseBody) =>
        this.httpLoggerService.logResponse({
          statusCode: response.statusCode,
          headers: response.getHeaders(),
          body: responseBody,
        }),
      ),
    );
  }
}
```

On a un `AnyExceptionFilter` qui intercepte toutes les exceptions qui remonte dans NestJS pour les logger.

```typescript
@Catch()
export class AnyExceptionFilter implements ExceptionFilter {
  public constructor(private readonly moduleRef: ModuleRef) {}

  async catch(exception: unknown, host: ArgumentsHost): Promise<void> {
    const ctx = host.switchToHttp();
    const request = ctx.getRequest();
    const response = ctx.getResponse<Response>();

    const httpException =
      exception instanceof HttpException ? exception : new InternalServerErrorException('Internal Server Error');

    const contextId = ContextIdFactory.create();
    this.moduleRef.registerRequestByContextId(request, contextId);
    const httpLoggerService = await this.moduleRef.resolve(HttpLoggerService, contextId);
    const statusCode = httpException.getStatus();
    httpLoggerService.logResponse({ statusCode, body: exception });

    response.status(statusCode).send(httpException.getResponse());
  }
}
```

## Le problème

Si je fais une requête classique tout va bien:
```JSON
{"requestId":"...d76","type":"REQUEST",...}
{"requestId":"...d76","type":"RESPONSE",...}
```

Si je tape sur une route qui n'existe pas:
```JSON
{"requestId":"...59e","statusCode":404,"body":"NotFoundException [Error]: ...","type":"RESPONSE",...}
```
Là je n'ai pas la requête.


Et maintenant, si j'ai une erreur qui est thrown dans le controlleur:
```JSON
{"requestId":"...ef4","type":"REQUEST",...}
{"requestId":"...c5d","statusCode":500,"body":"Error: ...","type":"RESPONSE",...}
```

L'id de la request n'est pas la même l'id de la réponse.

## L'explication

Le `LoggerInterceptor` est scoppé à une request `@Injectable({ scope: Scope.REQUEST })` or `AnyExceptionFilter` ne l'est pas.

Un composant scoppé à une request ne va être instancié que lorsque qu'une route est résolue. Du coup en cas de 404, le `LoggerInterceptor` n'est pas instancié et donc la requête n'est pas logger. J'ai besoin de ce scope, car c'est comme ça que je peux définir un id par request (c'est le travail du `RequestContextService`).

`AnyExceptionFilter` a lui aussi besoin du context de la request pour construire un `HttpLoggerService`. Il le récupère par un mécanisme documenté par NestJS: [https://docs.nestjs.com/fundamentals/module-ref#retrieving-instances](https://docs.nestjs.com/fundamentals/module-ref#retrieving-instances)

Le problème c'est qu'au lieu d'essayer de récupérer un `HttpLoggerService` existant, on crée systématiquement un nouveau contexte et on demande le `HttpLoggerService` pour ce contexte:

```typescript
const contextId = ContextIdFactory.create();
this.moduleRef.registerRequestByContextId(request, contextId);
const httpLoggerService = await this.moduleRef.resolve(HttpLoggerService, contextId);
```

Ça explique le fait que la request et la response error n'ont sont pas le même id.

## La solution

En creusant un peu dans les internals, j'ai trouvé que NestJS avait son propre mécanisme pour identifier des requests. C'est pas documenté et j'ai trouvé des issues qui en parlent, mais pas d'autre solution que le workaround que j'ai trouvé.

J'ai fait une PR pour ça: [https://github.com/hikoala/monorepo/pull/358](https://github.com/hikoala/monorepo/pull/358)

Concrètement je vais récupérer le internalId que NestJS set sur l'objet request pour retrouver le `HttpLoggerService` de la request:

```typescript
let contextId = request[REQUEST_CONTEXT_ID];
const contextIdWasNotDefined = contextId === undefined;

if (contextIdWasNotDefined) {
  contextId = ContextIdFactory.create();
  request[REQUEST_CONTEXT_ID] = contextId;
  this.moduleRef.registerRequestByContextId(request, contextId);
}

const httpLoggerService = await this.moduleRef.resolve(HttpLoggerService, contextId);
```

Si le internalId n'est pas set, c'est que le context n'a pas été créé par NestJS (404). Je crée donc un internalId que je set sur la request et j'enregistre la request avec cet id dans NestJS.

C'est pas idéal,  je préfèrerais passé par des helpers fournis par NestJS pour trouver l'id d'un context à partir d'une request. Mais ça n'existe visiblement pas. Donc en attendant... 😅
