# Generic Model

**What it is.** Generic Model is a type system for data. A type is declared as a Java interface, and the same type is also available at runtime as data, so that generic code can work with it.

**When you need it.** You need it for data that has to exist in more than one form: in memory, in a database, in a file a person can edit, and on the wire.

## The problem it solves

An application holds the same data in several forms: objects in a Java process, rows in a database, a file a person can read and edit, JSON on the wire.

A programming language describes data for one of those forms only, the one inside its own process. Java offers classes, and records, which are immutable and therefore cannot be built step by step, loaded in part, or edited. Everything beyond the process is left to the application: a mapping per type per database, a converter per type per format, a schema kept in step with both.

Generic Model is that missing layer. It fixes one type system for data, describes every type as data, and keeps the set of possible value kinds closed. A component that handles one direction — to a database, to YAML, to JSON, to a form — is written once against that type system, and then works for every Model, including Models written later.

## The type system is deliberately small

A value is of one of these kinds, and of no other.

| Kind | Members |
|---|---|
| Simple | `String`, `Boolean`, `Integer`, `Long`, `Float`, `Double`, `Decimal`, `Date` |
| Enum | Any Java enum that implements `EnumBase` |
| Entity | Any interface that extends `GenericEntity` |
| Collection | `List<E>`, `Set<E>`, `Map<K, V>` |
| Base | The type of `Object`. It accepts any value. |

There are no arrays, no arbitrary Java classes and no user defined containers.

The reason is that every kind has to be implemented on every platform Generic Model integrates with: an SQL database, values in a YAML file, the JavaScript type system. A larger set makes that work tedious, and in places impossible. Mapping Java's `float` and `double` onto JavaScript numbers is already awkward with the eight types that exist.

A larger set would also buy no modeling power. Further calendar or numeric types, or a `char` beside `String`, can already be expressed with what is there. The cost would be real and the gain none.

## Where the flexibility comes from

The small type system is not a restricted one. Three mechanisms carry the modeling.

| Mechanism | What it allows |
|---|---|
| Multiple inheritance | An Entity Type is an interface and may extend several others, so a shared trait is mixed into unrelated types instead of forcing one hierarchy |
| Entities as values, and in collections | A property may hold an entity, or a collection of entities, so a Model is a graph and not a tree of records |
| The base type | A property typed as the base type accepts any value, so a Model can stay open where the type is not known in advance |

## A type is available as data

A Model is not only compiled interfaces. The same Model exists as an entity graph, and can be stored, marshalled and sent like any other data. A component can receive a Model it was never compiled against and work with its types. See [models.md](models.md).

The type system is also implemented for JavaScript, so a Model can be used in a browser. This documentation describes the Java side.

## What an entity has that a Java object does not

| Feature | What it is for |
|---|---|
| Three identity properties | `id`, `partition` and `globalId` on every entity, so persistence and references need no per type convention |
| Interceptable property access | Reading and writing a property goes through an interceptor, which is what makes recorded changes and partly loaded entities possible |
| Mutable, and buildable in steps | A value can be created empty and filled in, which a record cannot |

Why a type is declared as an interface, and not as a class, is answered in [entity-types.md](entity-types.md).

## Asking a type what it is

Every type object implements `GenericModelType`, and every kind has a code in `TypeCode`.

A type answers what kind it is, without a cast: `isSimple()`, `isEnum()`, `isEntity()`, `isCollection()`, `isBase()`, `isScalar()`. Generic code branches on those, or on `getTypeCode()`. Because the set of kinds is closed, such a branch is complete once it covers them.

Every type object can also walk and copy a value of its type. That is what makes marshalling, copying and cutting one mechanism instead of many. See [cloning-and-traversing.md](cloning-and-traversing.md).

## GenericModelTypeReflection

`GenericModelTypeReflection` is the entry point that resolves a type. You reach it through `GMF.getTypeReflection()`.

```java
GenericModelTypeReflection typeReflection = GMF.getTypeReflection();

EntityType<Person> personType = typeReflection.getEntityType(Person.class);
GenericModelType byName = typeReflection.getType("com.example.model.Person");
GenericModelType ofValue = typeReflection.getType(somePerson);
```

| Lookup | By |
|---|---|
| `getType(Class)`, `getType(String)`, `getType(Object)` | A Java class, a type signature, or a value |
| `getEntityType(...)`, `getEnumType(...)`, `getSimpleType(...)` | The same, when you already know the kind |
| `getListType`, `getSetType`, `getMapType` | The element or key and value types |
| `findType`, `findEntityType`, `findEnumType` | The same as `getType`, but `null` instead of an error |
| `getModel(String)`, `getModelForType(String)` | A Model by name, or the Model that declares a type |

In application code you rarely need it. Every Entity Type carries its own type object in a constant, so `Person.T` is the same object as `typeReflection.getEntityType(Person.class)`. Use the reflection when the type is only known as a string or as a value.

## A Model is a set of types

Types are grouped into a Model, and a Model is one build artifact with its own dependencies. The Model at the bottom of every other is `com.braintribe.gm:root-model`, which declares `GenericEntity`.

See [models.md](models.md).

## What is built on top

Everything else in Hiconic takes this type system as its base.

| Layer | Adds | Page |
|---|---|---|
| Entity Types | How a type is declared and instantiated | [entity-types.md](entity-types.md) |
| Metadata | Configuration attached to the elements of a Model | [model-apis.md](model-apis.md) |
| Sessions | Working with entities, and recording every change | [sessions.md](sessions.md) |
| Access | Storing and querying entities | [access.md](access.md) |
| Services | Calls declared as entities | [services.md](services.md) |
| Reasons | Failures declared as entities | [reasons.md](reasons.md) |
| Resource | Files and other unstructured data | [resource.md](resource.md) |
| Marshalling | Reading and writing a graph | [marshalling.md](marshalling.md) |
| Unit testing | A Model, an Access and assertions for a test | [unit-testing.md](unit-testing.md) |

## Cheat sheet

| Type | Purpose |
|---|---|
| `GenericModelType` | The type object of any value |
| `TypeCode` | Which kind of type it is |
| `GenericModelTypeReflection` | Resolves a type from a class, a signature or a value |
| `GMF.getTypeReflection()` | How you reach it |
| `EssentialTypes` | The constants of the simple, collection and base types |
| `GenericEntity` | The root of every Entity Type |
| `EnumBase` | The interface of an enum in a Model |

## See also

- [entity-types.md](entity-types.md) — how you declare a type
- [models.md](models.md) — how types are grouped and shipped
