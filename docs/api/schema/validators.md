---
outline: deep
---

# Validators

[Ajv](https://ajv.js.org/) is the default JSON Schema validator used by `@feathersjs/schema`. We chose it because it's fully compliant with the JSON Schema spec and it's the fastest JSON Schema validator because it has its own compiler. It pre-compiles code for each validator, instead of dynamically creating validators from schemas during runtime.

<BlockQuote type="warning" label="Important">

Ajv and most other validation libraries are only used for ensuring data is valid and are not designed to convert data to different types. Type conversions and populating data can be done using [resolvers](./resolvers.md). This ensures a clean separation of concern between validating and populating data.

</BlockQuote>

## Usage

The following is the standard `validators.ts` file that sets up a validator for data and queries (for which string types will be coerced automatically). It also sets up a collection of additional formats using [ajv-formats](https://ajv.js.org/packages/ajv-formats.html). The validators in this file can be customized according to the [Ajv documentation](https://ajv.js.org/) and [its plugins](https://ajv.js.org/packages/). You can find the available Ajv options in the [Ajv class API docs](https://ajv.js.org/options.html).

```ts
import { Ajv, addFormats } from '@feathersjs/schema'
import type { FormatsPluginOptions } from '@feathersjs/schema'

const formats: FormatsPluginOptions = [
  'date-time',
  'time',
  'date',
  'email',
  'hostname',
  'ipv4',
  'ipv6',
  'uri',
  'uri-reference',
  'uuid',
  'uri-template',
  'json-pointer',
  'relative-json-pointer',
  'regex'
]

export const dataValidator = addFormats(new Ajv({}), formats)

export const queryValidator = addFormats(
  new Ajv({
    coerceTypes: true
  }),
  formats
)
```

## Validation functions

A validation function takes data and validates them against a schema using a validator. They can be used with any validation library. Currently the `getValidator` functions are available for:

- [TypeBox schema](./typebox.md#validators) to validate a TypeBox definition using an Ajv validator instance
- [JSON schema](./schema.md#validators) to validate a JSON schema object using an Ajv validator instance

## Hooks

The following hooks take a [validation function](#validation-functions) and validate parts of the [hook context](../hooks.md#hook-context).

### validateData

`schemaHooks.validateData` takes a [validation function](#validation-functions) and allows to validate the `data` in a `create`, `update` and `patch` request as well as [custom service methods](../services.md#custom-methods). It can be used as an `around` or `before` hook.

```ts
import { Ajv, schemaHooks } from '@feathersjs/schema'
import { Type, getValidator } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'
import { dataValidator } from '../validators'

const userSchema = Type.Object(
  {
    id: Type.Number(),
    email: Type.String(),
    password: Type.String(),
    avatar: Type.Optional(Type.String())
  },
  { $id: 'User', additionalProperties: false }
)
type User = Static<typeof userSchema>

const userDataSchema = Type.Pick(userSchema, ['email', 'password'])

// Returns validation functions for `create`, `update` and `patch`
const userDataValidator = getValidator(userDataSchema, dataValidator)

app.service('users').hooks({
  before: {
    all: [schemaHooks.validateData(userDataValidator)]
  }
})
```

### validateQuery

`schemaHooks.validateQuery` takes a [validation function](#validation-functions) and validates the `query` of a request. It can be used as an `around` or `before` hook. When using the `queryValidator` from the [usage](#usage) section, strings will automatically be converted to the right type using [Ajv's type coercion rules](https://ajv.js.org/coercion.html).

```ts
import { Ajv, schemaHooks } from '@feathersjs/schema'
import { Type, getValidator } from '@feathersjs/typebox'
import { queryValidator } from '../validators'

// Schema for allowed query properties
const messageQueryProperties = Type.Pick(messageSchema, ['id', 'text', 'createdAt', 'userId'], {
  additionalProperties: false
})
const messageQuerySchema = querySyntax(messageQueryProperties)
type MessageQuery = Static<typeof messageQuerySchema>

const messageQueryValidator = getValidator(messageQuerySchema, queryValidator)

app.service('messages').hooks({
  around: {
    all: [schemaHooks.validateQuery(messageQueryValidator)]
  }
})
```

## Validating Custom Methods

Custom service methods can also benefit from validation to ensure that the data they receive and return matches the expected schema. This section demonstrates how to apply validation to [custom service methods](../services.md#custom-methods).

### Example: Validating a Custom Method

The following example shows how to:

1. Define TypeBox schemas for the request and response data of a custom method
2. Create validators from these schemas using `getValidator`
3. Implement the custom method in a service class
4. Apply the validator to the custom method using `schemaHooks.validateData`

```ts
import { Ajv, schemaHooks } from '@feathersjs/schema'
import { Type, getValidator } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'
import { dataValidator } from '../validators'
import type { HookContext, Params } from '@feathersjs/feathers'

// Define the schema for the custom method request data
const processPaymentSchema = Type.Object(
  {
    amount: Type.Number({ minimum: 0.01 }),
    currency: Type.String({ enum: ['USD', 'EUR', 'GBP'] }),
    description: Type.Optional(Type.String()),
    customerId: Type.String({ format: 'uuid' })
  },
  { $id: 'ProcessPayment', additionalProperties: false }
)
type ProcessPaymentData = Static<typeof processPaymentSchema>

// Define the schema for the custom method response data
const paymentResultSchema = Type.Object(
  {
    transactionId: Type.String(),
    status: Type.String({ enum: ['succeeded', 'pending', 'failed'] }),
    amount: Type.Number(),
    currency: Type.String(),
    timestamp: Type.String({ format: 'date-time' })
  },
  { $id: 'PaymentResult', additionalProperties: false }
)
type PaymentResult = Static<typeof paymentResultSchema>

// Create validators for the request and response data
const processPaymentValidator = getValidator(processPaymentSchema, dataValidator)
const paymentResultValidator = getValidator(paymentResultSchema, dataValidator)

// Implement the service with the custom method
class PaymentService {
  async processPayment(data: ProcessPaymentData, params: Params): Promise<PaymentResult> {
    // Process the payment using a payment gateway or other logic
    // This is just an example implementation
    return {
      transactionId: crypto.randomUUID(),
      status: 'succeeded',
      amount: data.amount,
      currency: data.currency,
      timestamp: new Date().toISOString()
    }
  }
}

// Register the service with the custom method
const app = feathers<{ 'payments': PaymentService }>()
app.use('payments', new PaymentService(), {
  methods: ['processPayment']
})

// Apply validation to the custom method
app.service('payments').hooks({
  around: {
    processPayment: [
      // Validate the incoming data
      schemaHooks.validateData(processPaymentValidator),
      // You could also validate the result if needed
      async (context: HookContext, next) => {
        await next()
        // Validate the response data
        const result = context.result
        const validated = paymentResultValidator(result)
        if (validated !== result) {
          throw new Error('Invalid payment result format')
        }
      }
    ]
  }
})
```

In this example:

1. We define two TypeBox schemas:
   - `processPaymentSchema` for validating the incoming request data
   - `paymentResultSchema` for validating the response data

2. We create validators from these schemas using `getValidator` and our `dataValidator` instance.

3. We implement a `PaymentService` class with a custom `processPayment` method.

4. We register the service with the custom method using the `methods` option.

5. We apply validation to the custom method using:
   - `schemaHooks.validateData(processPaymentValidator)` to validate the incoming data
   - A custom hook to validate the response data (optional but recommended for consistency)

This approach ensures that both the data sent to your custom method and the data it returns conform to your defined schemas, providing type safety and validation throughout your application.
