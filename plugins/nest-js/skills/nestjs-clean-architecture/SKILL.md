---
name: nestjs-clean-architecture
description: Use when building, refactoring, reviewing, testing, or documenting NestJS backends. Enforces clean modular architecture, dependency injection, DTO validation, guards, pipes, interceptors, filters, configuration, security defaults, tests, and consistent NestJS code style.
---

# NestJS Clean Architecture Skill

Используй этот скилл для задач по NestJS: создание фич, рефакторинг, ревью, тесты, конфигурация, API, модули, DI, валидация, безопасность и подготовка к продакшену.

Цель: писать NestJS как архитектурный фреймворк, а не как Express в декоративной шляпе. Код должен быть понятным, модульным, тестируемым и скучно-надежным.

## 1. Сначала разведка, потом код

Перед изменениями обязательно осмотри репозиторий:

1. Проверь `package.json`: версии `@nestjs/*`, тестовый раннер, ORM, HTTP adapter, scripts.
2. Проверь `nest-cli.json`, `tsconfig*.json`, `eslint*`, `prettier*`, `jest*` или `vitest*`.
3. Найди существующую структуру модулей, naming, стиль DTO, подход к ошибкам, логированию и тестам.
4. Не обновляй major-версии NestJS, ORM, Jest/Vitest, Swagger, Express/Fastify без явного запроса.
5. Если проект на NestJS 10 или 11, учитывай различия маршрутизации и зависимостей. Не пиши код, который требует другой major-версии, пока не проверил текущую.
6. Сохраняй текущий стиль проекта, если он не ломает архитектуру, безопасность или тестируемость.

Если стиль проекта плохой, не устраивай революцию с вилами. Делай минимально нужное улучшение, а архитектурный долг явно отмечай в итоговом ответе.

## 2. Главные принципы

- Модульность важнее “одного удобного сервиса на всё”. God service пахнет подвалом.
- Контроллеры тонкие: HTTP-вход, параметры, DTO, вызов use-case/service, возврат ответа.
- Бизнес-логика живет в service/use-case/application слое, а не в controller, guard, interceptor или entity ORM.
- Доступ к базе изолирован в repository/persistence слое или в явно выделенном сервисе данных.
- DTO не равен entity, entity не равен domain model, domain model не обязан знать про NestJS.
- Валидация входа обязательна на границе системы.
- DI через constructor. Ручной `new` для зависимостей Nest почти всегда архитектурная крошка в клавиатуре.
- Круговые зависимости сначала лечатся дизайном, `forwardRef()` только как временный костыль с комментарием.
- Общий код должен быть действительно общим. `common/` не помойка для “пока не знаю куда”.
- Тесты пишутся на поведение, а не на внутренние шестеренки.

## 3. Рекомендуемая организация проекта

Следуй существующему варианту, но для новых проектов предпочитай feature-first структуру.

Вариант для обычного backend API:

```text
src/
  main.ts
  app.module.ts
  config/
    app.config.ts
    database.config.ts
    validation.ts
  common/
    decorators/
    exceptions/
    filters/
    guards/
    interceptors/
    pipes/
    utils/
  modules/
    users/
      users.module.ts
      users.controller.ts
      users.service.ts
      dto/
        create-user.dto.ts
        update-user.dto.ts
        user-response.dto.ts
      entities/              # ORM entities, если TypeORM/Mongoose style
      repositories/          # ports/adapters or concrete repositories
      users.service.spec.ts
      users.controller.spec.ts
    auth/
      auth.module.ts
      auth.controller.ts
      auth.service.ts
      strategies/
      guards/
      dto/
test/
  users.e2e-spec.ts
  jest-e2e.json
```

Для сложной предметной области можно разделять фичу по слоям:

```text
src/modules/orders/
  orders.module.ts
  presentation/
    orders.controller.ts
    dto/
  application/
    create-order.use-case.ts
    cancel-order.use-case.ts
    ports/
  domain/
    order.entity.ts
    order.policy.ts
    order.errors.ts
  infrastructure/
    persistence/
      prisma-orders.repository.ts
    messaging/
```

Правило выбора: простой CRUD не нужно превращать в собор с десятью слоями. Сложная бизнес-логика заслуживает отдельный domain/application слой. Архитектура должна защищать код, а не требовать жертвоприношений.

## 4. Модули

Каждый feature module инкапсулирует близкую функциональность:

- `imports`: только реальные зависимости модуля.
- `controllers`: HTTP/RPC/WebSocket входы фичи.
- `providers`: services, use cases, repositories, factories, policies.
- `exports`: только узкий публичный API, который нужен другим модулям.

Не экспортируй всё “на всякий случай”. Это превращает DI-граф в лапшу с дипломом.

Хорошие правила:

- `AppModule` собирает приложение, но не содержит бизнес-логику.
- `SharedModule` или `CommonModule` не должен зависеть от feature modules.
- Feature module не должен лезть напрямую в repository другой фичи.
- Если одной фиче нужны данные другой, экспортируй узкий query/application service или создай явный port.
- Dynamic modules используй для конфигурируемой инфраструктуры: clients, SDK, database, messaging, storage.
- Global modules применяй редко: config, logger, tracing, database client. Всё остальное импортируй явно.

## 5. Controllers

Контроллер отвечает за транспортный слой:

- route decorators: `@Get`, `@Post`, `@Patch`, `@Delete`.
- request params/query/body через DTO и pipes.
- HTTP status, headers, serialization, Swagger decorators при необходимости.
- вызов application service/use case.

В контроллере нельзя:

- делать SQL/Prisma/Mongoose запросы напрямую;
- содержать бизнес-правила;
- собирать сложные транзакции;
- тащить `Request` глубоко в service без крайней нужды;
- возвращать ORM entity наружу без маппинга, если entity содержит внутренние поля.

Шаблон контроллера:

```ts
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  async create(@Body() dto: CreateUserDto): Promise<UserResponseDto> {
    const user = await this.usersService.create(dto);
    return UserResponseDto.from(user);
  }
}
```

## 6. Services, use cases и providers

Сервис должен иметь одну понятную ответственность. Если файл разросся до “в нём живет весь бизнес”, режь по use cases или policies.

Хороший service:

- принимает DTO или command object;
- проверяет бизнес-инварианты;
- вызывает repositories/clients через DI;
- управляет транзакцией, если это application-level операция;
- возвращает domain/result/response model, но не привязывается к HTTP.

Плохой service:

- знает про `@Req()`, `Response`, cookies, HTTP status;
- сам создает зависимости через `new`;
- ловит все ошибки и возвращает `null`;
- смешивает auth, database, email, filesystem и billing в одном котле.

Для абстракций используй injection tokens:

```ts
export const USERS_REPOSITORY = Symbol('USERS_REPOSITORY');

export interface UsersRepository {
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<User>;
}
```

Но не абстрагируй всё подряд. Если проект маленький и Prisma используется напрямую как infrastructure choice, дополнительный repository может быть лишней мебелью. Абстракция нужна, когда она реально снижает связность, упрощает тесты или защищает домен.

## 7. DTO, validation и transformation

Входящие данные валидируй DTO-классами и `ValidationPipe`.

Базовый production-friendly вариант в `main.ts`:

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
    transformOptions: { enableImplicitConversion: false },
  }),
);
```

Правила DTO:

- DTO классы живут рядом с фичей: `dto/create-user.dto.ts`.
- Используй `class-validator` decorators: `@IsEmail()`, `@IsString()`, `@IsUUID()`, `@MinLength()`.
- Для query DTO явно описывай pagination, filters, sort.
- Для route params используй built-in pipes: `ParseIntPipe`, `ParseUUIDPipe`, `ParseBoolPipe`, `ParseArrayPipe`.
- Не используй DTO как domain entity.
- Не доверяй TypeScript типам как runtime validation. TypeScript исчезает после компиляции, как честность в плохом legacy.

Для partial DTO используй mapped types. Если проект использует Swagger plugin, импортируй mapped types из `@nestjs/swagger`, чтобы схема документации корректно собиралась.

## 8. Ошибки и exception handling

Выбирай слой ошибки правильно:

- Domain/application слой выбрасывает domain-specific errors или возвращает Result, если такой стиль уже принят.
- Controller/filter мапит domain errors в HTTP exceptions.
- Nest HTTP exceptions (`BadRequestException`, `NotFoundException`, `ConflictException`, `UnauthorizedException`) подходят на транспортной границе.
- Global exception filter форматирует ответ и логирует, но не прячет баги под “успешный false”.

Не делай так:

```ts
catch (error) {
  return null;
}
```

Делай так:

```ts
if (!user) {
  throw new NotFoundException('User not found');
}
```

Для публичных API не отдавай stack trace, SQL errors, секреты, raw provider errors. Логируй детали внутри, клиенту возвращай стабильный error contract.

## 9. Guards, pipes, interceptors, filters, middleware

Выбирай инструмент по назначению:

- Middleware: простая обработка до Nest route context, например raw logging, request id, body parser, Helmet/CORS.
- Guard: разрешить или запретить выполнение handler. Auth, roles, permissions, feature flags.
- Pipe: validation и transformation входных аргументов перед handler.
- Interceptor: логика вокруг handler: logging, metrics, serialization, cache, response mapping, timeout.
- Filter: преобразование exceptions в response.

Не пихай authorization в interceptor, validation в service, response formatting в каждый controller. У Nest есть крючки, пользуйся ими как инструментами, а не как ящиком с проводами.

## 10. Configuration

Используй `@nestjs/config` и typed config. Не читай `process.env` по всему проекту.

Рекомендуемый подход:

- `ConfigModule.forRoot({ isGlobal: true, validate })` или schema validation через Joi/Zod/класс.
- Все обязательные env переменные валидируются при старте.
- Config-файлы группируются по доменам: `app`, `database`, `auth`, `redis`, `mail`.
- Секреты не логируются.
- В тестах env задается явно, без зависимости от локальной машины.

Пример идеи:

```ts
export default registerAs('auth', () => ({
  jwtSecret: process.env.JWT_SECRET,
  accessTtl: process.env.ACCESS_TTL ?? '15m',
}));
```

В сервисе используй `ConfigService` или typed config provider, но не строковую магию в каждом методе.

## 11. Database и persistence

Nest database-agnostic, поэтому сначала смотри, что уже выбрано: Prisma, TypeORM, MikroORM, Mongoose, Sequelize, raw SQL.

Общие правила:

- Persistence слой не должен протекать наружу ORM-specific деталями.
- Не возвращай наружу записи с password hash, internal flags, deletedAt, provider tokens.
- Для списков всегда думай о pagination и лимитах.
- Для связей думай о N+1, indexes, select/include, eager loading.
- Транзакции должны быть явными там, где операция меняет несколько агрегатов/таблиц.
- Миграции должны быть частью процесса, а не “я руками в prod поправил”.
- Soft delete, audit fields, tenant isolation, optimistic locks добавляй только при реальной потребности.

Для Prisma обычно держи `PrismaService` в infrastructure/database module, а бизнесовые запросы группируй в repositories или feature-specific data services. Для TypeORM не размазывай `Repository<Entity>` по всем слоям, если домен сложный.

## 12. Security defaults

Для HTTP API проверь и, если уместно, настрой:

- `helmet()` как ранний middleware.
- CORS с явными origins, methods, headers. Не используй wildcard вместе с credentials.
- `ValidationPipe` с whitelist и запретом лишних полей.
- Rate limiting для login, password reset, OTP, search, export, webhooks.
- Auth через guards/strategies, authorization через отдельные policies/guards.
- Password hashing только современными алгоритмами и с настройками проекта.
- Secrets, tokens, cookies, Authorization headers не логировать.
- CSRF защиту, если используется cookie-based auth для браузера.
- File upload валидировать по size, mime/type, extension и бизнес-правилам.

Безопасность не должна жить только в README. Она должна быть в коде, тестах и конфиге.

## 13. Logging, observability, lifecycle

- Используй Nest `Logger` или проектный logger, не `console.log`.
- Для production предпочитай structured logs, особенно в контейнерах.
- Добавляй correlation/request id, если проект уже поддерживает distributed tracing.
- Interceptors хороши для request/response timing и metrics, но не логируй тела с PII.
- Для внешних клиентов добавляй timeout, retry/backoff, circuit breaker там, где нужно.
- Используй lifecycle hooks: `onModuleInit`, `onApplicationBootstrap`, `onModuleDestroy`, `beforeApplicationShutdown`, если нужно управлять подключениями.
- В `main.ts` включай graceful shutdown, если приложение держит DB, queues, sockets или consumers.

## 14. OpenAPI / Swagger

Если проект использует `@nestjs/swagger`:

- Документируй публичные endpoints, статусы, auth scheme, pagination, error shape.
- DTO файлы называй с суффиксом `.dto.ts`, чтобы CLI plugin мог их обрабатывать.
- Runtime validation всё равно требует `class-validator` decorators. Swagger schema не валидирует запросы.
- Не засыпь весь код ручными `@ApiProperty()`, если включен CLI plugin и можно описать типами/комментариями.
- Response DTO должен отражать публичный контракт, а не ORM entity.

## 15. Code style

Следуй стилю репозитория. Если стиля нет, используй эти правила:

- TypeScript strict-friendly, минимум `any`. Для неизвестного используй `unknown` и сузить тип.
- Файлы в kebab-case с Nest suffix: `users.service.ts`, `create-user.dto.ts`.
- Классы в PascalCase: `UsersService`, `CreateUserDto`.
- Методы и переменные в camelCase.
- Constructor injection: `constructor(private readonly usersService: UsersService) {}`.
- Prefer `readonly` для injected dependencies и immutable config.
- Не используй barrel files (`index.ts`) для module/provider imports внутри той же фичи, особенно если есть риск circular dependency.
- Не смешивай named/default exports без причины. В Nest классические providers обычно named exports.
- Не оставляй мертвый код, commented-out code, debug logs.
- Комментарии объясняют “почему”, а не пересказывают “что”.
- Не добавляй новые libraries без явной необходимости и проверки существующих зависимостей.
- Не меняй formatting вручную против Prettier/ESLint.

## 16. Testing strategy

Пирамида тестов:

1. Unit tests: services, use cases, guards, pipes, mappers, policies. Быстро, изолированно, зависимости mocked/faked.
2. Integration tests: module + реальные providers или тестовая БД, проверка wiring, transactions, repositories.
3. E2E tests: HTTP поведение через `INestApplication` и Supertest, глобальные pipes/filters/guards как в production.

Файлы:

- `*.spec.ts` рядом с кодом или в принятой структуре проекта.
- `*.e2e-spec.ts` обычно в `test/`.
- Test helpers/builders держи в `test/` или рядом с фичей, но не в production bundle.

Unit test шаблон:

```ts
describe('UsersService', () => {
  let service: UsersService;
  const usersRepository = {
    findByEmail: jest.fn(),
    save: jest.fn(),
  };

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: USERS_REPOSITORY, useValue: usersRepository },
      ],
    }).compile();

    service = moduleRef.get(UsersService);
    jest.clearAllMocks();
  });

  it('creates a user when email is unique', async () => {
    usersRepository.findByEmail.mockResolvedValue(null);
    usersRepository.save.mockResolvedValue({ id: 'user-id', email: 'a@b.com' });

    await expect(service.create({ email: 'a@b.com' })).resolves.toMatchObject({
      email: 'a@b.com',
    });
  });
});
```

E2E правила:

- Инициализируй приложение почти как production: global prefix, pipes, filters, interceptors.
- Внешние сервисы mock/stub через `overrideProvider`, но не mock весь Nest pipeline.
- Закрывай app: `await app.close()`.
- Проверяй happy path и хотя бы один плохой payload/unauthorized case.
- Для БД используй isolated test database, transaction rollback, truncate или testcontainers, в зависимости от проекта.

Тесты должны ловить баги, а не ласково гладить implementation details.

## 17. Checklist для новой фичи

Перед кодом:

- Понял bounded context и публичный API.
- Нашел существующий модуль или создал новый.
- Определил DTO, service/use-case, repository/client dependencies.
- Понял auth/permissions и validation rules.
- Понял, нужен ли transaction, event, queue или external call.

После кода:

- Контроллер тонкий.
- DTO валидируют вход.
- Service не знает про HTTP.
- Ошибки имеют стабильный контракт.
- Нет circular imports.
- Добавлены или обновлены unit/e2e тесты.
- Запущены доступные `lint`, `test`, `test:e2e`, `build` или явно указано, почему не запускались.

## 18. Checklist для рефакторинга

- Сначала зафиксируй текущее поведение тестами, если тестов нет.
- Меняй структуру малыми шагами.
- Не смешивай refactor и feature, если можно разделить.
- Удаляй устаревшие exports/providers/routes.
- Проверяй DI errors после переноса providers.
- Следи за circular dependencies и barrel imports.
- Финальный diff должен быть понятнее исходника. Если нет, это не рефакторинг, а декорация болота.

## 19. Что Codex должен делать в конце задачи

В финальном ответе кратко сообщи:

- что изменено;
- какие архитектурные решения приняты;
- какие тесты/команды запускались;
- что не удалось проверить;
- какие риски или follow-up остались.

Не говори, что всё идеально, если тесты не запускались. Честность дешевле rollback.

## 20. Красные флаги

Остановись и пересмотри решение, если видишь:

- controller с SQL/ORM запросами;
- service на 800 строк;
- `forwardRef()` без попытки развязать дизайн;
- `process.env` внутри бизнес-метода;
- DTO, который одновременно input, output, entity и database schema;
- `any` в публичных методах;
- глобальный module, экспортирующий половину приложения;
- `catch` без rethrow/logging;
- тесты, которые проверяют только “service is defined”;
- e2e тесты без реального global validation pipe;
- CORS `*` с credentials;
- logs с токенами, паролями, cookies или PII.

Когда сомневаешься, выбирай простоту, явные зависимости, маленькие модули и тестируемое поведение. NestJS любит порядок. Хаос он тоже переварит, но потом пришлет счет.
