---
outline: deep
---

# SQL Databases

<Badges>

[![npm version](https://img.shields.io/npm/v/@feathersjs/knex.svg?style=flat-square)](https://www.npmjs.com/package/@feathersjs/knex)
[![Changelog](https://img.shields.io/badge/changelog-.md-blue.svg?style=flat-square)](https://github.com/feathersjs/feathers/blob/dove/packages/knex/CHANGELOG.md)

</Badges>

Support for SQL databases like PostgreSQL, MySQL, MariaDB, SQLite or MSSQL is provided in Feathers via the `@feathersjs/knex` database adapter which uses [KnexJS](https://knexjs.org/). Knex is a fast and flexible query builder for SQL and supports many databases without the overhead of a full blown ORM like Sequelize. It still provides an intuitive syntax and more advanced tooling like migration support.

```bash
$ npm install --save @feathersjs/knex
```

<BlockQuote>

The Knex adapter implements the [common database adapter API](./common) and [querying syntax](./querying).

</BlockQuote>

## API

### KnexService(options)

`new KnexService(options)` returns a new service instance initialized with the given options. The following example extends the `KnexService` and then uses the `sqliteClient` (or relevant client for your SQL database type) from the app configuration and provides it to the `Model` option, which is passed to the new `MessagesService`.

```ts
import type { Params } from '@feathersjs/feathers'
import { KnexService } from '@feathersjs/knex'
import type { KnexAdapterParams, KnexAdapterOptions } from '@feathersjs/knex'

import type { Application } from '../../declarations'
import type { Messages, MessagesData, MessagesQuery } from './messages.schema'

export interface MessagesParams extends KnexAdapterParams<MessagesQuery> {}

export class MessagesService<ServiceParams extends Params = MessagesParams> extends KnexService<
  Messages,
  MessagesData,
  ServiceParams
> {}

export const messages = (app: Application) => {
  const options: KnexAdapterOptions = {
    paginate: app.get('paginate'),
    Model: app.get('sqliteClient'),
    name: 'messages'
  }
  app.use('messages', new MessagesService(options))
}
```

### Options

The Knex specific adapter options are:

- `Model {Knex}` (**required**) - The KnexJS database instance
- `name {string}` (**required**) - The name of the table
- `schema {string}` (_optional_) - The name of the schema table prefix (example: `schema.table`)

The [common API options](./common.md#options) are:

- `id {string}` (_optional_, default: `'id'`) - The name of the id field property. By design, Knex will always add an `id` property.
- `paginate {Object}` (_optional_) - A [pagination object](#pagination) containing a `default` and `max` page size
- `multi {string[]|boolean}` (_optional_, default: `false`) - Allow `create` with arrays and `patch` and `remove` with id `null` to change multiple items. Can be `true` for all methods or an array of allowed methods (e.g. `[ 'remove', 'create' ]`)

There are additionally several legacy options in the [common API options](./common.md#options)

### getModel([params])

`service.getModel([params])` returns the [Knex](https://knexjs.org/guide/query-builder.html) client for this table.

### db(params)

`service.db([params])` returns the Knex database instance for a request. This will include the `schema` table prefix and use a transaction if passed in `params`.

### createQuery(params)

`service.createQuery(params)` returns a query builder for a service request, including all conditions matching the query syntax. This method can be overriden to e.g. [include associations](#associations) or used in a hook customize the query and then passing it to the service call as [params.knex](#paramsknex).

```ts
app.service('messages').hooks({
  before: {
    find: [
      async (context: HookContext) => {
        const query = context.service.createQuery(context.params)

        // do something with query here
        query.orderBy('name', 'desc')

        context.params.knex = query
      }
    ]
  }
})
```

### params.knex

When making a [service method](https://docs.feathersjs.com/api/services.html) call, `params` can contain an `knex` property which allows to modify the options used to run the KnexJS query. See [createQuery](#createqueryparams) for an example.

## Querying

In addition to the [common querying mechanism](./querying.md), this adapter also supports the following operators. Note that these operators need to be added for each query-able property to the [TypeBox query schema](../schema/typebox.md#query-schemas) or [JSON query schema](../schema/schema.md#querysyntax) like this:

```ts
const messageQuerySchema = Type.Intersect(
  [
    // This will additionally allow querying for `{ name: { $ilike: 'Dav%' } }`
    querySyntax(messageQueryProperties, {
      name: {
        $ilike: Type.String()
      }
    }),
    // Add additional query properties here
    Type.Object({})
  ],
  { additionalProperties: false }
)
```

### $like

Find all records where the value matches the given string pattern. The following query retrieves all messages that start with `Hello`:

```ts
app.service('messages').find({
  query: {
    text: {
      $like: 'Hello%'
    }
  }
})
```

Through the REST API:

```
/messages?text[$like]=Hello%
```

### $notlike

The opposite of `$like`; resulting in an SQL condition similar to this: `WHERE some_field NOT LIKE 'X'`

```ts
app.service('messages').find({
  query: {
    text: {
      $notlike: '%bar'
    }
  }
})
```

Through the REST API:

```
/messages?text[$notlike]=%bar
```

### $ilike

For PostgreSQL only, the keywork $ilike can be used instead of $like to make the match case insensitive. The following query retrieves all messages that start with `hello` (case insensitive):

```ts
app.service('messages').find({
  query: {
    text: {
      $ilike: 'hello%'
    }
  }
})
```

Through the REST API:

```
/messages?text[$ilike]=hello%
```

## Search

Basic search can be implemented with the [query operators](#querying).

## Associations

While [resolvers](../schema/resolvers.md) offer a reasonably performant way to fetch associated entities, it is also possible to join tables to populate and query related data. This can be done by overriding the [createQuery](#createqueryparams) method and using the [Knex join methods](https://knexjs.org/guide/query-builder.html#join) to join the tables of related services.

### Querying

Considering a table like this:

```ts
await db.schema.createTable('todos', (table) => {
  table.increments('id')
  table.string('text')
  table.bigInteger('personId').references('id').inTable('people').notNullable()
  return table
})
```

To query based on properties from the `people` table, join the tables you need in `createQuery` like this:

```ts
class TodoService<ServiceParams = KnexAdapterParams<TodoQuery>> extends KnexService<Todo> {
  createQuery(params: KnexAdapterParams<AdapterQuery>) {
    const query = super.createQuery(params)

    query.join('people as person', 'todos.personId', 'person.id')

    return query
  }
}
```

This will alias the table name from `people` to `person` (since our Todo only has a single person) and then allow to query all related properties as dot separated properties like `person.name`, including the [Feathers query syntax](./querying.md):

```ts
// Find the Todos for all Daves older than 100
app.service('todos').find({
  query: {
    'person.name': 'Dave',
    'person.age': { $gt: 100 }
  }
})
```

Note that in most applications, the query-able properties have to explicitly be added to the [TypeBox query schema](../schema/typebox.md#query-schemas) or [JSON query schema](../schema/schema.md#querysyntax). Support for the query syntax for a single property can be added with the `queryProperty` helper:

```ts
import { queryProperty } from '@feathersjs/typebox'

export const todoQueryProperties = Type.Pick(userSchema, ['text'])
export const todoQuerySchema = Type.Intersect(
  [
    querySyntax(userQueryProperties),
    // Add additional query properties here
    Type.Object(
      {
        // Only query the name for strings
        'person.name': Type.String(),
        // Support the query syntax for the age
        'person.age': queryProperty(Type.Number())
      },
      { additionalProperties: false }
    )
  ],
  { additionalProperties: false }
)
```

### Populating

Related properties from the joined table can be added as aliased properties with [query.select](https://knexjs.org/guide/query-builder.html#select):

```ts
class TodoService<ServiceParams = KnexAdapterParams<TodoQuery>> extends KnexService<Todo> {
  createQuery(params: KnexAdapterParams<AdapterQuery>) {
    const query = super.createQuery(params)

    query
      .join('people as person', 'todos.personId', 'person.id')
      // This will add a `personName` property
      .select('person.name as personName')
      // This will add a `person.age' property
      .select('person.age')

    return query
  }
}
```

<BlockQuote type="warning" label="important">

Since SQL does not have a concept of nested objects, joined properties will be dot separated strings, **not nested objects**. Conversion can be done by e.g. using Lodash `_.set` in a [resolver converter](../schema/resolvers.md#options).

</BlockQuote>

This works well for individual properties, however if you require the complete (and safe) representation of the entire related data, use a [resolver](../schema/resolvers.md) instead.

## Transactions

Transactions are essential for maintaining data consistency when performing multiple database operations that should succeed or fail as a single unit. The Knex adapter provides powerful transaction support through three specialized hooks that work together to manage transaction lifecycle.

### How Transactions Work

The Knex adapter transaction system consists of three hooks that manage the complete lifecycle of a database transaction:

1. **`transaction.start()`** - Creates a new transaction or reuses an existing one from the parent context
2. **`transaction.end()`** - Commits the transaction if all operations were successful
3. **`transaction.rollback()`** - Rolls back all changes if an error occurs

When a request begins:
- A new Knex transaction is created and attached to `params.transaction`
- All subsequent database operations within that request context use this transaction
- If the request completes successfully, all changes are committed atomically
- If any error occurs, all changes are rolled back, ensuring data consistency

The transaction object stored in `params.transaction` contains:
- The Knex transaction instance
- A `committed` promise that resolves when the transaction completes
- Nested transaction tracking for complex service interactions

### Basic Transaction Setup

Transactions can be configured at the application level (affecting all services) or at individual service levels:

#### Application-wide Transactions

```ts
// src/app.ts
import { transaction } from '@feathersjs/knex'

app.hooks({
  before: {
    all: [transaction.start()]
  },
  after: {
    all: [transaction.end()]
  },
  error: {
    all: [transaction.rollback()]
  }
})
```

#### Service-level Transactions

```ts
// src/services/orders/orders.ts
import { transaction } from '@feathersjs/knex'

export const orders = (app: Application) => {
  app.use('orders', new OrderService(options), {
    methods: ['find', 'get', 'create', 'patch', 'remove'],
    events: []
  })
  
  app.service('orders').hooks({
    before: {
      all: [transaction.start()]
    },
    after: {
      all: [transaction.end()]
    },
    error: {
      all: [transaction.rollback()]
    }
  })
}
```

### Using the Around Hook

The `around` hook provides a cleaner way to manage transactions by wrapping the entire operation:

```ts
import { transaction } from '@feathersjs/knex'

app.service('orders').hooks({
  around: {
    all: [
      async (context, next) => {
        // Start transaction
        await transaction.start()(context)
        
        try {
          // Execute the service method
          await next()
          
          // Commit on success
          await transaction.end()(context)
        } catch (error) {
          // Rollback on error
          await transaction.rollback()(context)
          throw error
        }
      }
    ]
  }
})
```

### Comprehensive Example: Order and Shipping System

Here's a real-world example demonstrating transactions with nested service calls. When creating an order, we automatically create a shipping order. If either operation fails, everything is rolled back:

#### Schema Definitions

```ts
// src/services/orders/orders.schema.ts
import { Type } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'

export const orderSchema = Type.Object(
  {
    id: Type.Number(),
    customerId: Type.Number(),
    productId: Type.Number(),
    quantity: Type.Number(),
    totalAmount: Type.Number(),
    status: Type.String(),
    createdAt: Type.String({ format: 'date-time' }),
    updatedAt: Type.String({ format: 'date-time' })
  },
  { $id: 'Order', additionalProperties: false }
)

export type Order = Static<typeof orderSchema>
export type OrderData = Omit<Order, 'id' | 'createdAt' | 'updatedAt'>

// src/services/shipping-orders/shipping-orders.schema.ts
import { Type } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'

export const shippingOrderSchema = Type.Object(
  {
    id: Type.Number(),
    orderId: Type.Number(),
    shippingAddress: Type.String(),
    shippingMethod: Type.String(),
    estimatedDelivery: Type.String({ format: 'date-time' }),
    trackingNumber: Type.Optional(Type.String()),
    status: Type.String(),
    createdAt: Type.String({ format: 'date-time' }),
    updatedAt: Type.String({ format: 'date-time' })
  },
  { $id: 'ShippingOrder', additionalProperties: false }
)

export type ShippingOrder = Static<typeof shippingOrderSchema>
export type ShippingOrderData = Omit<ShippingOrder, 'id' | 'createdAt' | 'updatedAt'>
```

#### Service Implementation with Transactions

```ts
// src/services/orders/orders.class.ts
import { KnexService } from '@feathersjs/knex'
import type { KnexAdapterParams } from '@feathersjs/knex'
import type { Application } from '../../declarations'
import type { Order, OrderData } from './orders.schema'

export class OrderService extends KnexService<Order, OrderData> {
  constructor(options: any) {
    super(options)
  }
}

// src/services/orders/orders.ts
import { transaction } from '@feathersjs/knex'
import { hooks as schemaHooks } from '@feathersjs/schema'
import type { Application } from '../../declarations'
import { OrderService } from './orders.class'

export const orders = (app: Application) => {
  const options = {
    paginate: app.get('paginate'),
    Model: app.get('knexClient'),
    name: 'orders'
  }

  app.use('orders', new OrderService(options), {
    methods: ['find', 'get', 'create', 'patch', 'remove'],
    events: []
  })

  app.service('orders').hooks({
    around: {
      all: [schemaHooks.resolveExternal(), schemaHooks.resolveResult()]
    },
    before: {
      all: [
        transaction.start(),
        schemaHooks.validateQuery(),
        schemaHooks.resolveQuery()
      ],
      create: [
        schemaHooks.validateData(),
        schemaHooks.resolveData(),
        async (context) => {
          // This hook automatically creates a shipping order after order creation
          context.params.createShipping = true
          return context
        }
      ]
    },
    after: {
      all: [transaction.end()],
      create: [
        async (context) => {
          if (context.params.createShipping) {
            const { transaction } = context.params
            
            try {
              // Create shipping order using the same transaction
              const shippingOrder = await app.service('shipping-orders').create(
                {
                  orderId: context.result.id,
                  shippingAddress: context.data.shippingAddress || 'Default Address',
                  shippingMethod: context.data.shippingMethod || 'Standard',
                  estimatedDelivery: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString(),
                  status: 'pending'
                },
                {
                  ...context.params,
                  transaction // Pass the transaction to nested service call
                }
              )
              
              // Attach shipping order to result
              context.result.shippingOrder = shippingOrder
            } catch (error) {
              // If shipping order creation fails, the transaction will be rolled back
              throw new Error(`Failed to create shipping order: ${error.message}`)
            }
          }
          return context
        }
      ]
    },
    error: {
      all: [transaction.rollback()]
    }
  })
}
```

#### Nested Transactions Example

```ts
// src/services/shipping-orders/shipping-orders.ts
import { transaction } from '@feathersjs/knex'
import type { Application } from '../../declarations'
import { ShippingOrderService } from './shipping-orders.class'

export const shippingOrders = (app: Application) => {
  const options = {
    paginate: app.get('paginate'),
    Model: app.get('knexClient'),
    name: 'shipping_orders'
  }

  app.use('shipping-orders', new ShippingOrderService(options), {
    methods: ['find', 'get', 'create', 'patch', 'remove'],
    events: []
  })

  app.service('shipping-orders').hooks({
    before: {
      all: [
        transaction.start() // Will reuse parent transaction if exists
      ],
      create: [
        async (context) => {
          // Validate that the order exists
          const order = await app.service('orders').get(context.data.orderId, {
            transaction: context.params.transaction // Use same transaction
          })
          
          if (!order) {
            throw new Error('Order not found')
          }
          
          // Additional validation
          if (order.status === 'cancelled') {
            throw new Error('Cannot create shipping for cancelled order')
          }
          
          return context
        }
      ]
    },
    after: {
      all: [transaction.end()]
    },
    error: {
      all: [transaction.rollback()]
    }
  })
}
```

### Error Handling in Transactions

Proper error handling is crucial for transaction integrity. The transaction system automatically handles rollbacks when errors occur, but you can also implement custom error handling:

#### Basic Error Handling

```ts
app.service('orders').hooks({
  error: {
    all: [
      transaction.rollback(),
      async (context) => {
        // Log transaction failure
        console.error('Transaction failed:', {
          service: context.path,
          method: context.method,
          error: context.error.message,
          data: context.data
        })
        
        // You can modify the error message for the client
        if (context.error.message.includes('shipping')) {
          context.error.message = 'Order creation failed: Unable to setup shipping'
        }
        
        return context
      }
    ]
  }
})
```

#### Advanced Error Handling with Retry Logic

```ts
async function withTransactionRetry(fn, maxRetries = 3) {
  let lastError
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error
      
      // Check if error is retryable (e.g., deadlock)
      if (error.code === 'ER_LOCK_DEADLOCK' && attempt < maxRetries) {
        console.log(`Transaction deadlock detected, retrying (${attempt}/${maxRetries})...`)
        await new Promise(resolve => setTimeout(resolve, 100 * attempt))
        continue
      }
      
      throw error
    }
  }
  
  throw lastError
}

// Usage in a hook
app.service('orders').hooks({
  before: {
    create: [
      async (context) => {
        context.result = await withTransactionRetry(async () => {
          // Your transactional logic here
          return await context.service._create(context.data, context.params)
        })
        
        return context
      }
    ]
  }
})
```

### Waiting for Transaction Completion

When dealing with real-time events or external systems, you may need to wait for transaction completion:

```ts
app.service('orders').publish(async (data, context) => {
  const { transaction } = context.params

  if (transaction) {
    // Wait for transaction to complete
    const success = await transaction.committed

    if (!success) {
      // Transaction was rolled back, don't send events
      return []
    }
  }

  // Transaction committed successfully, send real-time updates
  return app.channel(`customers/${data.customerId}`)
})
```

### Testing Transactions

Here's an example of testing transaction behavior:

```ts
// test/services/orders.test.ts
import assert from 'assert'
import { app } from '../../src/app'

describe('Order Transactions', () => {
  it('rolls back order when shipping creation fails', async () => {
    // Mock shipping service to fail
    const originalCreate = app.service('shipping-orders')._create
    app.service('shipping-orders')._create = async () => {
      throw new Error('Shipping service unavailable')
    }

    try {
      // Attempt to create order (should fail and rollback)
      await app.service('orders').create({
        customerId: 1,
        productId: 1,
        quantity: 2,
        totalAmount: 100,
        status: 'pending'
      })
      
      assert.fail('Should have thrown an error')
    } catch (error) {
      assert.equal(error.message.includes('shipping'), true)
    }

    // Verify order was not created
    const orders = await app.service('orders').find({
      query: { customerId: 1 }
    })
    assert.equal(orders.total, 0)

    // Restore original method
    app.service('shipping-orders')._create = originalCreate
  })

  it('successfully creates order with shipping in transaction', async () => {
    const order = await app.service('orders').create({
      customerId: 1,
      productId: 1,
      quantity: 2,
      totalAmount: 100,
      status: 'pending',
      shippingAddress: '123 Main St',
      shippingMethod: 'Express'
    })

    assert.ok(order.id)
    assert.ok(order.shippingOrder)
    assert.equal(order.shippingOrder.orderId, order.id)
  })
})
```

### Important Considerations

<BlockQuote type="warning" label="Important">

When calling another Knex service within a hook and you want to share the transaction, you must pass `context.params.transaction` in the parameters of the service call. Failing to do so will result in the nested service call executing outside the transaction context.

</BlockQuote>

<BlockQuote type="info" label="Best Practices">

1. Always use all three transaction hooks (start, end, rollback) together
2. Pass the transaction parameter to all nested service calls
3. Handle errors appropriately to ensure proper rollback
4. Use the `committed` promise when coordinating with external systems
5. Test transaction rollback scenarios thoroughly
6. Consider using the `around` hook for cleaner transaction management
7. Be aware that long-running transactions can cause database locks

</BlockQuote>

This also works with nested service calls and nested transactions. For example, if a service calls `transaction.start()` and passes the transaction param to a nested service call, which also calls `transaction.start()` in its own hooks, they will share the top-most `committed` promise that will resolve once all of the transactions have successfully committed.

## Error handling

The adapter only throws [Feathers Errors](https://docs.feathersjs.com/api/errors.html) with the message to not leak sensitive information to a client. On the server, the original error can be retrieved through a secure symbol via `import { ERROR } from '@feathersjs/knex'`

```ts
import { ERROR } from 'feathers-knex'

try {
  await knexService.doSomething()
} catch (error: any) {
  // error is a FeathersError with just the message
  // Safely retrieve the Knex error
  const knexError = error[ERROR]
}
```

## Migrations

In a generated application, migrations are already set up. See the [CLI guide](../../guides/cli/knexfile.md) and the [KnexJS migrations documentation](https://knexjs.org/guide/migrations.html) for more information.
