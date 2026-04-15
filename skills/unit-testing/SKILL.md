---
name: unit-testing
description: Write behavioural unit tests for service methods. Supports NestJS/Mongoose and Express/Typegoose stacks.
---

# Unit Testing

## Determine your stack

Check `package.json` for framework dependencies. Also look for existing test files — NestJS projects use `*.service.spec.ts`, Express projects use `*.service.test.ts`.

**NestJS + Mongoose** (`@nestjs/core` in `package.json`) → follow `references/nestjs-mongoose.md`.

**Express + Typegoose** (`express` in `package.json`, no `@nestjs/core`) → follow `references/express-typegoose.md`.
