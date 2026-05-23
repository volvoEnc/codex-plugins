---
name: nestjs-feature-design
description: Use when designing, scoping, decomposing, or specifying a NestJS backend feature or refactor before implementation. Bridges optional `$superpowers:brainstorming` with the NestJS clean architecture checklist, while falling back to a local lightweight design pass when Brainstorming is not available in the current skill set.
---

# NestJS Feature Design

Use this skill before implementing non-trivial NestJS features, refactors, API changes, persistence changes, auth changes, queue/event flows, or module boundary changes.

The goal is to turn an idea into a NestJS design that is clear enough to implement without discovering the architecture by archaeology.

## Brainstorming Availability Gate

Before invoking any external brainstorming workflow, inspect the current available skills.

- If `$superpowers:brainstorming` or `$brainstorming` is available in the current skill list, use it as the primary design workflow and apply the NestJS overlay below.
- If Brainstorming is not available, do not invoke it, do not ask the user to install it, and do not simulate that it was used. Run the fallback design pass in this skill instead.
- If the user explicitly says not to use Brainstorming for the task, respect that and use the fallback design pass.

This skill must not make Brainstorming a hard dependency. Optional means optional, not "optional until the first inconvenient moment."

## NestJS Overlay For Brainstorming

When Brainstorming is available and allowed, let it control the design conversation and gates. During that process, add these NestJS-specific checks.

In project exploration, inspect:

- `package.json`, lockfiles, `nest-cli.json`, `tsconfig*.json`, lint/format/test config.
- Current NestJS major version, HTTP adapter, ORM/data layer, auth stack, config pattern, logger, and test runner.
- Related modules, controllers, services/use cases, DTOs, guards, pipes, interceptors, filters, repositories, entities/schemas, migrations, and tests.

Clarifying questions should uncover:

- API contract: route/RPC/event shape, request DTO, response DTO, status codes, error shape.
- Module ownership: new module, existing module, exports, imports, provider boundaries.
- Business rules: invariants, permissions, validation, idempotency, transactions.
- Persistence: schema/entity changes, migrations, indexes, N+1 risk, soft delete or tenant isolation.
- Side effects: events, queues, external clients, mail, files, cache, search, webhooks.
- Testing: unit, integration, e2e, fixtures, database strategy, mocks/stubs.

Approaches should usually compare:

- Minimal local change in existing controller/service.
- Feature-module change with explicit service/use-case boundaries.
- Layered domain/application/infrastructure split for complex business logic.

The final design must explicitly cover:

- Module structure and ownership.
- Controller, service/use-case, repository/client responsibilities.
- DTO validation and response mapping.
- Error handling and public error contract.
- Auth/authorization and relevant guards/policies.
- Persistence, migrations, transactions, and data compatibility.
- Observability and lifecycle concerns when relevant.
- Test plan by level: unit, integration, e2e.

After the design is approved, use `$nestjs-clean-architecture` for implementation guidance if it is available.

## Fallback Design Pass

Use this path when Brainstorming is unavailable or explicitly disabled.

1. Explore the project context before proposing code.
2. Ask only the minimum clarifying questions needed. Prefer one question at a time for ambiguous API, persistence, auth, or testing decisions.
3. Present 2-3 implementation approaches with trade-offs and a recommendation.
4. Present a concise design and wait for user approval before implementation when the change is non-trivial.
5. For a tiny, low-risk change, present a short design summary and proceed only when the user has already given clear implementation intent.

Fallback designs should include:

- Scope and non-goals.
- Files/modules likely to change.
- API/data contract changes.
- Validation, auth, and error behavior.
- Persistence/migration impact.
- Test commands or test gaps.

## Refusal Policy

Push back once when a requested design would damage the project, such as:

- Adding a global module or shared service that becomes a dependency landfill.
- Returning ORM entities directly through public API.
- Skipping runtime validation for user-controlled input.
- Changing major framework or ORM versions without a migration reason.
- Solving circular dependencies with `forwardRef()` before trying to fix the boundary.

Offer a safer alternative with the smallest useful scope. If the user repeats the same request, proceed only after making the risk explicit.
