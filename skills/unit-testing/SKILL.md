---
name: unit-testing
description: Write behavioural unit tests for NestJS/Mongoose service methods following project testing standards
---

# Write Tests

Write behavioural unit tests for service methods using **Jest**, **`@nestjs/testing`** (`Test.createTestingModule()`), and **Mongoose**.

## Prerequisites

Check `package.json` Jest config before writing tests:

- **ESM packages** (e.g., `nanoid`): `"transformIgnorePatterns": ["node_modules/(?!nanoid/)"]`
- **Path aliases** (`src/` imports): `"moduleNameMapper": { "^src/(.*)$": "<rootDir>/$1" }`
- **Mongoose `@Prop` types**: Union types, enums, and const-derived types need explicit `type: String` in `@Prop()` decorators or tests will fail at import time with "Cannot determine a type".

```typescript
// ❌ Mongoose can't resolve these at runtime
@Prop({ required: true, enum: MyEnum })
field: MyEnum;

// ✅ Add type: String
@Prop({ required: true, type: String, enum: MyEnum })
field: MyEnum;
```

## Setup Template

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { getModelToken } from '@nestjs/mongoose';
import { Model, Types } from 'mongoose';
import { NotFoundException, ForbiddenException, BadRequestException } from '@nestjs/common';
import { ServiceName } from './service-name.service';
import { Document, DocumentDocument } from './document.schema';
import { DependencyService } from '../dependency/dependency.service';

// DO NOT jest.mock() models at module level — use jest.spyOn in individual tests

describe('ServiceName', () => {
  let service: ServiceName;
  let documentModel: Model<DocumentDocument>;
  let dependencyService: DependencyService;

  const mockLogger = {
    log: jest.fn(),
    warn: jest.fn(),
    error: jest.fn(),
    debug: jest.fn(),
    verbose: jest.fn(),
  };

  beforeEach(async () => {
    jest.clearAllMocks();

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ServiceName,
        {
          provide: getModelToken(Document.name),
          useValue: {
            // Include jest.fn() for every model method the service calls
            findById: jest.fn(),
            findOne: jest.fn(),
            find: jest.fn(),
          },
        },
        {
          provide: DependencyService,
          useValue: {
            // Include jest.fn() for every service method that will be spied on
            findByUserId: jest.fn(),
            findById: jest.fn(),
            create: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<ServiceName>(ServiceName);
    documentModel = module.get(getModelToken(Document.name));
    dependencyService = module.get<DependencyService>(DependencyService);

    // Logger is readonly; override via Object.defineProperty
    Object.defineProperty(service, 'logger', { value: mockLogger });
  });

  describe('methodName', () => {
    // Tests go here
  });
});
```

## Test Organization

Nest tests under `describe('methodName', () => { ... })` with three sub-groups:

### 1. Happy Paths
All successful execution paths: primary flow, alternative valid flows, edge cases that succeed.

### 2. Error Cases
All failure scenarios: missing/invalid inputs, not-found, validation failures, permission errors, dependency failures, error propagation.

### 3. Logging
Verify `mockLogger.log` on success and `mockLogger.error` on failure, both with proper context objects.

## Mocking Patterns

### Dependency service methods (async)
```typescript
jest.spyOn(dep, 'method').mockResolvedValue(data as any);         // success
jest.spyOn(dep, 'method').mockResolvedValue(null);                // not found
jest.spyOn(dep, 'method').mockRejectedValue(new Error('msg'));    // failure
```

### Mongoose models — always `jest.spyOn`, never `jest.mock`

```typescript
// Simple calls (no chaining — when service awaits model method directly)
jest.spyOn(documentModel, 'findById').mockResolvedValue(doc as any);
jest.spyOn(documentModel, 'findOne').mockResolvedValue(doc as any);
jest.spyOn(documentModel, 'find').mockResolvedValue([doc] as any);
jest.spyOn(documentModel, 'findByIdAndUpdate').mockResolvedValue(doc as any);
jest.spyOn(documentModel, 'findOneAndUpdate').mockResolvedValue(doc as any);
jest.spyOn(documentModel, 'create').mockResolvedValue(doc as any);
jest.spyOn(documentModel, 'updateMany').mockResolvedValue({} as any);
jest.spyOn(documentModel, 'deleteOne').mockResolvedValue({} as any);
jest.spyOn(documentModel, 'countDocuments').mockResolvedValue(5 as any);
jest.spyOn(documentModel, 'aggregate').mockResolvedValue([doc] as any);

// With chaining (populate, exec, sort, etc.)
jest.spyOn(documentModel, 'findById').mockReturnValue({
  populate: jest.fn().mockReturnValue({
    exec: jest.fn().mockResolvedValue(doc),
  }),
} as any);

jest.spyOn(documentModel, 'find').mockReturnValue({
  populate: jest.fn().mockReturnValue({
    sort: jest.fn().mockReturnValue({
      exec: jest.fn().mockResolvedValue([doc1, doc2]),
    }),
  }),
} as any);
```

### Model constructor (`new this.model()`)
```typescript
const mockDocInstance = {
  _id: new Types.ObjectId(),
  status: 'PENDING',
  save: jest.fn().mockResolvedValue(undefined),
};
const MockModel = jest.fn().mockImplementation(() => mockDocInstance);
service['documentModel'] = MockModel as any;

expect(MockModel).toHaveBeenCalled();
expect(mockDocInstance.save).toHaveBeenCalled();
```

## Assertions

```typescript
// Called with specific args (use expect.any() for dynamic values)
expect(dep.method).toHaveBeenCalledWith(arg1, expect.any(Types.ObjectId));

// NestJS exceptions — type, message, or both
await expect(service.method(input)).rejects.toThrow(NotFoundException);
await expect(service.method(input)).rejects.toThrow('Not found');
await expect(service.method(input)).rejects.toThrowError(
  new NotFoundException('Not found')
);

// Result value
expect(result).toEqual(expected);
expect(result).toMatchObject({ key: value });

// Logging — success
expect(mockLogger.log).toHaveBeenCalledWith(
  'Log message',
  expect.objectContaining({ key: value })
);

// Logging — failure (error.stack + context)
expect(mockLogger.error).toHaveBeenCalledWith(
  'Error message',
  expect.any(String),
  expect.objectContaining({ contextKey: contextValue })
);
```

## Behavioral Testing Approach

1. **Analyze the function** — map every execution path (if/else, try/catch), dependency call, error condition, and side effect (DB updates, service calls).
2. **Create test cases** for each behavior — what happens in this scenario? What dependencies are called? What validations occur? What is logged?
3. **Set up test data** — minimal, realistic mocks. Use `Types.ObjectId()` for MongoDB IDs. Include all required schema fields. Make mocks representative of actual data.
4. **Verify** — assert all service calls with correct arguments, return values, descriptive error messages, and logging with proper context.

## Common Patterns

### ObjectIds
Declare all IDs as named variables at the describe-block scope. Use `Types.ObjectId` instances in mock data; use `.toHexString()` for input parameters.

```typescript
describe('methodName', () => {
  const tag1Id = new Types.ObjectId();
  const tag2Id = new Types.ObjectId();
  const userId = new Types.ObjectId();
  const organizationId = new Types.ObjectId().toHexString();

  const mockInput = {
    tagIds: [tag1Id.toHexString(), tag2Id.toHexString()],
    userId: userId.toHexString(),
  };

  const mockTags = [
    { _id: tag1Id, name: 'JavaScript' },
    { _id: tag2Id, name: 'React' },
  ];

  const mockData = {
    _id: new Types.ObjectId(),
    tags: [tag1Id, tag2Id],       // ObjectId instances in data
    createdBy: userId,
    organization: organizationId,
  };

  it('should handle tag validation', async () => {
    jest.spyOn(tagModel, 'find').mockResolvedValue(mockTags as any);

    await service.method(mockInput);

    expect(tagModel.find).toHaveBeenCalledWith({
      _id: { $in: [tag1Id, tag2Id] },
    });
  });
});
```

**Key Points**:
- Declare all IDs as named variables at the top of the describe block
- Use hex strings (`toHexString()`) for input parameters and arrays when necessary
- Use ObjectId instances for mock data and database operations
- This enables precise assertions and avoids magic strings

### Error Propagation
```typescript
jest.spyOn(dep, 'method').mockRejectedValue(new Error('Dependency failed'));
await expect(service.method()).rejects.toThrow('Dependency failed');
expect(mockLogger.error).toHaveBeenCalled();
```

### Conditional Logic
Create separate tests for each branch:
- Test when condition is true
- Test when condition is false
- Test edge cases (null, undefined, empty)

### Multiple Model Operations in One Test
```typescript
it('should handle complex flow with multiple DB operations', async () => {
  jest.spyOn(modelA, 'findById').mockResolvedValue(mockA as any);
  jest.spyOn(modelB, 'find').mockResolvedValue([mockB1, mockB2] as any);
  jest.spyOn(modelA, 'findByIdAndUpdate').mockResolvedValue(updatedA as any);

  const result = await service.method(params);

  expect(modelA.findById).toHaveBeenCalledWith(id);
  expect(modelB.find).toHaveBeenCalledWith(query);
  expect(modelA.findByIdAndUpdate).toHaveBeenCalledWith(id, update, options);
});
```

## Full Example

```typescript
describe('createCustomerOrder', () => {
  const createOrderDto = {
    measurementId: measurementId.toHexString(),
    addressId: new Types.ObjectId().toHexString(),
    desiredTailorLocation: 'NG',
    description: 'Test order',
    outfitType: 'agbada',
    addOnIds: [],
  };

  describe('Happy Paths', () => {
    it('should successfully create an order for a customer', async () => {
      jest.spyOn(customerService, 'findByUserId').mockResolvedValue(mockCustomer as any);
      jest.spyOn(measurementService, 'findById').mockResolvedValue(mockMeasurement as any);
      jest.spyOn(addressService, 'findById').mockResolvedValue(mockAddress as any);

      const result = await service.createCustomerOrder(userId, createOrderDto as any);

      expect(customerService.findByUserId).toHaveBeenCalledWith(userId);
      expect(measurementService.findById).toHaveBeenCalledWith(createOrderDto.measurementId);
      expect(result).toBeDefined();
      expect(mockLogger.log).toHaveBeenCalled();
    });
  });

  describe('Error Cases', () => {
    it('should throw when customer not found', async () => {
      jest.spyOn(customerService, 'findByUserId').mockResolvedValue(null);

      await expect(
        service.createCustomerOrder(userId, createOrderDto as any)
      ).rejects.toThrow(NotFoundException);
    });

    it('should throw when measurement belongs to different customer', async () => {
      jest.spyOn(customerService, 'findByUserId').mockResolvedValue(mockCustomer as any);
      jest.spyOn(measurementService, 'findById').mockResolvedValue({
        ...mockMeasurement,
        customer: new Types.ObjectId(), // Different customer
      } as any);

      await expect(
        service.createCustomerOrder(userId, createOrderDto as any)
      ).rejects.toThrow(ForbiddenException);
    });

    it('should propagate dependency errors', async () => {
      jest.spyOn(customerService, 'findByUserId').mockRejectedValue(new Error('DB error'));

      await expect(
        service.createCustomerOrder(userId, createOrderDto as any)
      ).rejects.toThrow();
      expect(mockLogger.error).toHaveBeenCalled();
    });
  });

  describe('Logging', () => {
    it('should log info with context on success', async () => {
      jest.spyOn(customerService, 'findByUserId').mockResolvedValue(mockCustomer as any);
      jest.spyOn(measurementService, 'findById').mockResolvedValue(mockMeasurement as any);

      await service.createCustomerOrder(userId, createOrderDto as any).catch(() => {});

      expect(mockLogger.log).toHaveBeenCalledWith(
        'Creating customer order',
        expect.objectContaining({ userId })
      );
    });

    it('should log error with context on failure', async () => {
      jest.spyOn(customerService, 'findByUserId').mockRejectedValue(new Error('DB error'));

      await expect(
        service.createCustomerOrder(userId, createOrderDto as any)
      ).rejects.toThrow();

      expect(mockLogger.error).toHaveBeenCalledWith(
        expect.any(String),
        expect.any(String),
        expect.objectContaining({ userId })
      );
    });
  });
});
```

## Coverage Goals

- ✅ All execution paths tested
- ✅ All error conditions tested
- ✅ All dependencies verified with correct arguments
- ✅ All logging verified with proper context
- ✅ Edge cases covered
- ✅ Integration points validated

## Workflow

When invoked:
1. **Analyze** the target method — identify all behaviors and execution paths.
2. **Write tests** covering happy paths, error cases, and logging in that order.
3. **Follow patterns above exactly** — `jest.spyOn` for models, `jest.fn()` stubs in all providers.
4. **Ensure proper organization** — nested describes, clear test names.
5. **Run the tests** to verify they pass.
