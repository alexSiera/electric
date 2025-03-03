---
title: Shapes
description: >-
  The core primitive for controlling sync in the ElectricSQL system.
sidebar_position: 30
---

Shapes are the core primitive for controlling sync in the ElectricSQL system.

Local apps establish shape subscriptions. This syncs data from the cloud onto the local device using the [Satellite replication protocol](../../api/satellite.md), into the local embedded SQLite database. Once the initial data has synced, [queries](./queries.md) can run against it.

The [Electric sync service](../installation/service.md) maintains shape subscriptions and streams any new data and data changes onto the local device. In this way, local devices can sync a sub-set of a larger database for interactive offline use.

:::caution Work in progress
Shapes are currently limited to whole-table sync. See [Roadmap -> Shapes](../../reference/roadmap.md#shapes) for more&nbsp;information.
:::

## What is a shape?

A shape is a set of related data that's synced onto the local device. It is defined by:

- a **table**, in your [electrified DDL schema](../data-modelling/electrification.md), such as `projects`
- a **query**, with where clauses used to filter the rows in that table
- an **include tree**, a directed acyclic graph of related data

For example, this `sync` call causes a project and all its issues, their comments and comment authors to sync atomically onto the local device:

```js
await db.projects.sync({
  where: {
    id: 'abcd'
  },
  include: {
    issues: {
      include: {
        comments: {
          include: {
            author: true
          }
        }
      }
    }
  }
})
```

Once the data has synced onto the local device, it's kept in sync using a **shape subscription**. This monitors the replication stream and syncs any new data, updates and deletions onto the local device for as long as the shape's [subscription and retention semantics](#subscription-and-retention-semantics) define.

## Syncing shapes

ElectricSQL syncs shapes using the [`sync`](../../api/clients/typescript.md#sync) client function. You can sync individual rows:

```ts
await db.projects.sync({
  where: {
    id: 'abcd'
  }
})
```

You can sync filtered sets of rows:

```ts
await db.projects.sync({
  where: {
    status: 'active'
  }
})
```

You can sync wide, shallow shapes, such as a small set of columns from all rows in a table:

```ts
await db.projects.sync({
  where: true,
  select: {
    id,
    title
  }
})
```

You can sync deep nested shapes, such as an individual project with all its related content:

```ts
await db.projects.sync({
  where: {
    id: 'abcd'
  },
  include: {
    issues: {
      include: {
        comments: {
          include: {
            author: true
          }
        }
      }
    }
  }
})
```

### Filter clauses

Shape query clauses support:

- `equality`
- `not`
- `in` / `notIn`
- `gt` / `gte`
- `lt` / `lte`

They can be combined as multiple parts of the same clause, and explicitly using:

- `AND`
- `OR`

See the <DocPageLink path="api/clients/typescript" /> docs for more details.

### Permissions

When a user subscribes to a shape, they will only see data that they have permission to read. Any data that belongs to a shape that a user does not have permission to read is dropped from the replication stream.

As a result, the data that syncs onto a users' local device for a given shape may be a **partial result** and will often be different for different users.

See <DocPageLink path="usage/data-modelling/permissions" /> for more information.

### Promise workflow

The [`sync`](../../api/clients/typescript.md#sync) function resolves to an object containing a promise:

1. the first `sync()` promise resolves when the shape subscription has been confirmed by the server (the sync service)
2. the second `synced` promise resolves when the data in the shape has fully synced onto the local device

```tsx
// Resolves once the shape subscription
// is confirmed by the server.
const shape = await db.projects.sync()

// Resolves once the initial data load
// for the shape is complete.
await shape.synced
```

If the shape subscription is invalid, the first promise will be rejected. If the data load fails for some reason, the second promise will be rejected.

### Data loading

Data synced onto the local device via a shape subscription appears atomically in the local database. I.e.: it all loads within a single transaction.

You can query the local database at any time, for example establishing a [Live query](./queries.md#live-queries) at the same time as initiating the shape sync. The query results will initially be empty (unless data is already in the local database) and then will update once with the full set of data loaded by the shape subscription.

For example, this is OK:

```tsx
const MyComponent = () => {
  const { db } = useElectric()!
  const { results } = useLiveQuery(db.projects.liveMany())

  // console.log('MyComponent rendering')
  // console.log('results', results)

  const syncProjects = async () => {
    // console.log('syncProjects')

    const shape = await db.projects.sync()
    // console.log('shape subscription confirmed')

    await shape.synced
    // console.log('shape data synced')
  }

  useEffect(() => {
    syncProjects()
  }, [])

  return (
    <h1>{ results.length }</h1>
  )
}
```

Or you can explicitly wait for the sync, for example, by conditionally rendering a child component once `shape.synced` has resolved:

```tsx
const MyContainer = () => {
  const { db } = useElectric()!
  const [ ready, setReady ] = useState(false)

  // console.log('MyContainer rendering')
  // console.log('ready', ready)

  const syncProjects = async () => {
    // console.log('syncProjects')

    const shape = await db.projects.sync()
    // console.log('shape subscription confirmed')

    await shape.synced
    // console.log('shape data synced')

    setReady(true)
  }

  useEffect(() => {
    syncProjects()
  }, [])

  if (!ready) {
    return null
  }

  return (
    <MyComponent />
  )
}

const MyComponent = () => {
  const { db } = useElectric()!
  const { results } = useLiveQuery(db.projects.liveMany())

  // console.log('MyComponent rendering')
  // console.log('results', results)

  return (
    <h1>{ results.length }</h1>
  )
}
```

For many applications you can simply define the data you want to sync up-front, for example at app load time and then just code against the local database once the data has synced in. For others, you can craft more dynamic partial replication, for example syncing data in as the user navigates through different routes or parts of the app.

## Future capabilities

Shape-based sync is under active development. We aim soon to provide additional capabilities and primitives, such as the ones outlined below.

### Segmentation indexes

When a shape subscription is established, the initial data load (the rows in the shape) are fetched by a Postgres query. However, ongoing changes from the replication stream are mapped into shapes by the Electric sync service.

This limits the expressiveness of shape filter clauses to the matching capabilities of the sync service (as opposed to the full query capabilities of Postgres). Segmentation indexes are a mechanism to pre-define virtual columns as user-defined functions in the Postgres database, in order to support abitrary query logic in shape definitions.

### Subscription and retention semantics

Currently all shapes are always live. However, in some cases, you may want to make ephemeral queries and keep results available for offline use without always keeping them live and up-to-date. Subscription semantics will allow you to configure whether a shape subscription is maintained to keep the data synced in a shape live or not.

Currently all synced data is retained forever until explicitly deleted. Retention semantics will provide a declarative API to control data retention  and deterministic behaviour when there's contention for storage resources.

### Discovered shapes

Many applications need to provide listings and search capabilities. You don't always want to sync the whole search index or table onto the local device to support this.

Discovery queries allow you to "discover" the results of a query run server-side on Postgres and to then subscribe to the query results. This allows you to keep a shape subscription live for a search or listing query.

### Derived shapes

Derived shapes are shapes that are derived from a component hierarchy. They are analogous to the way that fragments are aggregated into a top-level query by the Relay compiler.
