# Access

**What it is.** An Access is a persistence domain behind one interface. It answers queries and applies Manipulations, and it hides which store is underneath.

**When you need it.** You need it whenever data must be stored. You also want it whenever the same code should work over a database and over memory.

The word is easy to misread. An Access is a *place data lives*, not a permission.

## IncrementalAccess

An Access implements one interface, and the interface is small on purpose.

| Method | Purpose |
|---|---|
| `query(SelectQuery)` | Query across types, with joins and projections |
| `queryEntities(EntityQuery)` | Query entities of one type |
| `queryProperty(PropertyQuery)` | Load one property of one entity |
| `applyManipulation(...)` | Apply recorded changes |
| `getReferences(...)` | Find what points at an entity |
| `getPartitions()` | The partitions this Access holds |
| `getMetaModel()` | The Model it stores |

The name says the important part. An Access takes **incremental** change: a set of Manipulations describing what happened, not a new version of the data. That is what a Session sends on `commit()`. See [sessions.md](sessions.md).

Because the interface is this narrow, an Access can be a relational database, a store in memory, a file, or a facade over another Access.

## Access implementations

| Implementation | Use it for |
|---|---|
| Hibernate Access | The normal case: a relational database |
| Smood Access | Everything in memory. Tests, small reference data, and anything that is rebuilt at boot. |

**Smood** is the in memory store: it holds an object graph, indexes it, and answers the same queries as a database. It is not a cut down version for tests only — an application uses one wherever data is small and read often.

Configuring an Access is a platform job, not a Model job. See [../reflex/persistence.md](../reflex/persistence.md).

## Queries

There are three query types. They differ in what they return, not only in how they are written.

| Query | Returns | Use it when |
|---|---|---|
| `EntityQuery` | Entities of one type | You want objects of one type, with a condition |
| `SelectQuery` | Rows of selected values | You need joins, projections, grouping or aggregates |
| `PropertyQuery` | One property of one entity | You have the entity and need a collection it did not carry |

`PropertyQuery` is the one people forget. When a query returned entities with an absent collection, this is how you load that collection, without fetching the whole entity again.

A query is an entity, like everything else. It can be built, stored, sent to another process and evaluated there.

## Building a query

Use the fluent builder rather than assembling the entities by hand.

```java
EntityQuery query = EntityQueryBuilder.from(Person.T)
		.where()
			.property(Person.name).like("A*")
		.orderBy(Person.name)
		.limit(50)
		.done();
```

```java
SelectQuery query = new SelectQueryBuilder()
		.from(Person.T, "p")
		.join("p", Person.company, "c")
		.select("p", Person.name)
		.select("c", Company.name)
		.where()
			.property("c", Company.name).eq("Example Ltd")
		.done();
```

There is also a query language, and the Session accepts it as a string. It is convenient in a tool or a console, and less so in code, because a string is not checked by the compiler.

## Running a query

A Session runs a query against the Access it belongs to.

```java
List<Person> people = session.query().entities(query).list();

Person person = session.query().entity(Person.T, id).require();
Person byGlobalId = session.query().findEntity(globalId);
```

| Session method | Purpose |
|---|---|
| `query()` | Query the Access |
| `queryCache()` | Query what the Session already holds, without the Access |
| `queryDetached()` | Query the Access without attaching the result to the Session |
| `entity(...)` | One entity by id, or by reference |
| `findEntity(globalId)` | One entity by `globalId` |

`queryDetached()` is the right choice for a read that must not become part of the Session's transaction, such as a large export.

## Partitions

One Access can hold data from several sources, and `partition` on every entity says which. An entity is identified by its type, its `id` and its `partition` together.

Most applications have one partition and never think about it. It matters as soon as an Access federates two stores, because an `id` alone is then no longer unique.

## Aspects

An Access can be wrapped by another Access that adds behaviour and delegates the rest. Because the interface is small, the wrapper is small too.

That is how cross cutting concerns are added without touching the store or the callers:

- Row level security: narrow every query before it reaches the store.
- Auditing: record every Manipulation that goes through.
- Identifier generation: fill in ids before entities are written.
- Full text: maintain an index beside the store.

An aspect sees exactly what the Access sees — queries and Manipulations — so it cannot be bypassed by a caller.

## Cheat sheet

| Element | Purpose |
|---|---|
| `IncrementalAccess` | The interface every Access implements |
| `applyManipulation(...)` | How changes reach a store |
| Hibernate Access | A relational database |
| Smood Access | A store in memory |
| `EntityQuery` | Entities of one type |
| `SelectQuery` | Rows, joins, projections, aggregates |
| `PropertyQuery` | One property of one entity |
| `EntityQueryBuilder`, `SelectQueryBuilder` | Build a query fluently |
| `session.query()` | Run a query through a Session |
| `session.queryCache()`, `queryDetached()` | Query the Session only, or without attaching |
| `partition` | Which store inside an Access an entity came from |

## See also

- [sessions.md](sessions.md) — how changes are recorded and committed
- [model-apis.md](model-apis.md) — the Metadata that shapes how an Access stores a Model
- [../reflex/persistence.md](../reflex/persistence.md) — configuring an Access in an application
- [unit-testing.md](unit-testing.md) — an Access in memory, for a test
