# Sessions

**What it is.** A Session is the working area for entities. It creates them, records every change as a Manipulation, and can send those changes to an Access.

**When you need it.** You need a Session whenever you read or change stored data. You also want one whenever changes must be recorded rather than applied silently.

## Three levels

There are three interfaces, and each adds to the one before it. Ask for the least you need.

| Interface | Adds |
|---|---|
| `GmSession` | Creating entities, attaching and deleting them, and the interceptor that makes changes observable |
| `ManagedGmSession` | Identity management, queries, Resource access, and applying Manipulations |
| `PersistenceGmSession` | An Access behind it: `commit()`, a transaction, and evaluation of Service Requests |

`ManagedGmSession` alone is useful. It keeps one instance per identity and lets you query what it holds, without any store behind it. That is what a client uses to hold data it received.

## Creating a Session

A `PersistenceGmSessionFactory` gives a Session for one Access, named by its id.

```java
PersistenceGmSession session = sessionFactory.newSession("access.addressbook");
```

A Session is short lived and belongs to one unit of work. Do not share one across threads, and do not keep one open across requests. In Reflex you rarely build one by hand; the platform hands you the Session for the Access you are working in.

## Working with entities

Create through the Session, not through the type. An entity created through the Session is attached from the first property.

```java
Person person = session.create(Person.T);
person.setName("Alice");

session.commit();
```

| Method | Purpose |
|---|---|
| `create(Type.T)` | A new entity, attached, with the `@Initializer` defaults |
| `createRaw(Type.T)` | The same, without the defaults |
| `acquire(Type.T, globalId)` | The entity with that `globalId`, created if it does not exist yet |
| `attach(entity)` | Bring an entity that was built outside under the Session |
| `deleteEntity(entity)` | Mark it for deletion |
| `query()` | Query the Access, or what the Session already holds |
| `commit()` | Send everything recorded so far to the Access |

`acquire` makes an initializer or an import idempotent: running it twice does not create a second entity, because the `globalId` identifies the one that already exists.

## Manipulations

Every change through a Session becomes a Manipulation: an entity that describes what happened.

| Manipulation | Records |
|---|---|
| `InstantiationManipulation`, `DeleteManipulation` | An entity came into existence or was removed |
| `ChangeValueManipulation` | A property got a new value |
| `AddManipulation`, `RemoveManipulation`, `ClearCollectionManipulation` | A collection property changed |
| `CompoundManipulation` | Several of the above, as one |
| `AbsentingManipulation` | A property was turned back into an absent one |

This is the reason an entity from a Session is an enhanced entity: the property access goes through an interceptor, and the interceptor records the Manipulation. See [entity-types.md](entity-types.md).

Because a Manipulation is an entity, the record is data like everything else. It can be sent to another process, stored as an audit trail, or applied to a second Session to reproduce the same change.

```java
session.manipulate().mode(ManipulationMode.REMOTE).apply(manipulation);
```

## Transaction

The recorded Manipulations form a transaction, and the transaction can be walked backwards.

| Method | Purpose |
|---|---|
| `hasManipulations()` | Whether anything changed since the last commit |
| `undo(n)`, `redo(n)`, `canUndo()`, `canRedo()` | Move through the recorded changes |
| `beginNestedTransaction()` | A frame you can roll back on its own |
| `getManipulatedProperties()` | Exactly which properties changed |

A nested transaction is how a set of changes is attempted and then kept or dropped: begin one, make the changes, and either commit it into its parent frame or roll it back.

## Commit

`commit()` sends the recorded Manipulations to the Access and returns what came back, including the ids the store assigned to new entities.

Two rules follow from the recording.

- Nothing reaches the store before `commit()`. A Session that is dropped without a commit changes nothing.
- A commit sends the recorded changes, not the current state. Two Sessions that changed the same entity do not overwrite each other blindly; each sends what it did.

## Transient and persistent

Not every Session has a store behind it.

| Situation | Session |
|---|---|
| Holding data received from a service, on a client | `ManagedGmSession` |
| Building a graph in memory to marshal or send | `ManagedGmSession` |
| Reading and writing an Access | `PersistenceGmSession` |

A `PersistenceGmSession` is also an `Evaluator<ServiceRequest>`, so a Service Request can be evaluated against the Access the Session belongs to. See [services.md](services.md).

## Cheat sheet

| Element | Purpose |
|---|---|
| `GmSession` | Create, attach and delete entities |
| `ManagedGmSession` | Identity management, queries, Resources, applying Manipulations |
| `PersistenceGmSession` | An Access behind it, with `commit()` and a transaction |
| `PersistenceGmSessionFactory.newSession(accessId)` | Open a Session on an Access |
| `session.create(Type.T)` | A new attached entity |
| `session.acquire(Type.T, globalId)` | Find or create, by `globalId` |
| `session.query()` | Query the Access or the Session |
| `session.commit()` | Send the recorded changes |
| `session.getTransaction()` | Undo, redo and nested frames |
| `Manipulation` | One recorded change, as an entity |

## See also

- [entity-types.md](entity-types.md) — why an enhanced entity is what makes recording possible
- [access.md](access.md) — what sits behind a `PersistenceGmSession`
- [resource.md](resource.md) — reading a Resource payload through a Session
- [unit-testing.md](unit-testing.md) — a Session for a test, without an application
