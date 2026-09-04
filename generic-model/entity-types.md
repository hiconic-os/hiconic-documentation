# Entity Types

**What it is.** An Entity Type is declared as a Java interface that extends `GenericEntity` and holds a constant of its own type object. You write only the interface; the platform provides the implementation.

**When you need it.** You declare an Entity Type for every data structure the platform must handle, from a stored record to a Service Request.

## Why an interface

A type declaration should contain the data and nothing else: the property names, their types, and Metadata such as `@Mandatory`. The implementation is the platform's business, and from the declaring side it is simply there.

Java offers one way to declare that and nothing else, and that is an interface. The alternative would be a separate syntax for entities plus a build step that generates Java from it, which means a language and a build stage to maintain.

An interface buys three more things.

- **More than one implementation of the same type.** A plain instance, an enhanced instance and a proxy all satisfy the same interface. A proxy is far easier to produce for an interface than for a class.
- **Utility methods stay out of sight.** What an implementation needs internally is not declared on the interface, so a caller never sees it.
- **A type may extend several others**, which a Java class cannot. A shared trait is mixed in rather than duplicated.

## GenericEntity

Every Entity Type extends `GenericEntity`, directly or through another Entity Type. It brings three properties and nothing else.

| Property | Purpose |
|---|---|
| `id` | The identifier inside one Access. Its type is decided by the Access, so the getter is generic. |
| `partition` | Which store the entity came from, when an Access holds more than one |
| `globalId` | An identifier that stays the same everywhere. Unique. |

An entity is complete with none of them set. They are filled when the entity is stored, or when you assign a `globalId` yourself for something that must be found again by name.

## The T constant

Every Entity Type declares one constant, by convention called `T`.

```java
public interface Person extends GenericEntity {

	EntityType<Person> T = EntityTypes.T(Person.class);

	String name = "name";
	String company = "company";

	String getName();
	void setName(String name);

	Company getCompany();
	void setCompany(Company company);
}
```

`Person.T` is the type object. It is how you create an instance, read the properties, and name the type in an API.

```java
Person person = Person.T.create();
person.setName("Alice");
```

The `String` constants next to it are the property names. They cost nothing and let you name a property without a string literal, which is what queries and Metadata need.

## Creating an instance

`EntityType` offers four ways to create an entity, and the difference matters.

| Call | Gives you |
|---|---|
| `create()` | An enhanced entity with the `@Initializer` defaults applied. Use this one. |
| `createRaw()` | An enhanced entity, without the defaults |
| `createPlain()` | A plain entity with the defaults applied |
| `createPlainRaw()` | A plain entity, without the defaults |

An **enhanced** entity routes every property access through an interceptor. That is what lets a Session record a Manipulation for each change, and what lets a partly loaded entity report an absent property. A **plain** entity is a bare data holder, and is faster.

Use `create()` unless you are building a large number of throwaway objects and you know that nothing must observe them.

There is also a flow style form, and a form that creates through a Session and attaches the entity in one step:

```java
Person person = Person.T.create(p -> p.setName("Alice"));
Person attached = Person.T.create(session, p -> p.setName("Alice"));
```

## Properties

A property is a getter and setter pair. The platform reads the pair and makes a `Property` out of it.

`Property` is how generic code works with a value it does not know:

| Method | Answers |
|---|---|
| `get(entity)`, `set(entity, value)` | The value, through the interceptor |
| `getType()` | The type of the value |
| `isIdentifier()`, `isPartition()`, `isGlobalId()` | Whether this is one of the three identity properties |
| `isAbsent(entity)` | Whether the value was left out when the entity was loaded |
| `getInitializer()` | The declared default, if any |

An Entity Type lists them with `getProperties()`, or `getDeclaredProperties()` for the ones it adds itself.

A `@Transient` pair is not a property of the Model. The value is held in a plain field, it is never stored and never marshalled, and it does not appear in `getProperties()`. Use it for something that belongs to one run of the program, such as a stack trace kept beside a Reason.

## Enums

An enum in a Model is a normal Java enum that implements `EnumBase`.

```java
public enum Gender implements EnumBase<Gender> {

	MALE, FEMALE;

	public static final EnumType<Gender> T = EnumTypes.T(Gender.class);

	@Override
	public EnumType<Gender> type() {
		return T;
	}
}
```

The pattern mirrors an Entity Type: a constant holds the type object, and `type()` gives it for any constant.

## Structural annotations

These annotations change the type itself. Metadata, which describes what a type means, is a separate subject: see [model-apis.md](model-apis.md).

| Annotation | On | Effect |
|---|---|---|
| `@Abstract` | Type | The type cannot be instantiated |
| `@Initializer("...")` | Property | The default value applied by `create()` |
| `@Transient` | Property | Not a property of the Model; a plain field instead |
| `@TypeRestriction` | Property | Narrows which types a property may actually hold, when its declared type is a common supertype |
| `@SelectiveInformation("...")` | Type | A template that renders an entity as text, for display |
| `@ToStringInformation("...")` | Type | A template for `toString()`, for logs rather than for users |
| `@ForwardDeclaration("group:model")` | Type | The type belongs to another Model than the artifact it sits in |

`@Initializer` takes a string, and the string is encoded by type: text in single quotes, a `l`, `f`, `d` or `b` suffix for a non integer number, and `now()` for the current date.

```java
@Initializer("'unknown'")
String getName();

@Initializer("now()")
Date getCreated();
```

## Under the hood

You write an interface, so something must provide the class behind it. The platform generates that class **at runtime**, the first time the type is used. Reading `Person.T` analyses the interface and creates the implementation classes for it, one enhanced and one plain. There is no generated source in your project and no build step to run.

The same machinery can go one step further. A Model that exists only as data, and has no compiled interfaces at all, can be deployed at runtime with `deploy()`. Its types then behave like any other, which is how a platform can serve a Model it was never compiled against.

## Cheat sheet

| Element | Purpose |
|---|---|
| `GenericEntity` | The root of every Entity Type; brings `id`, `partition`, `globalId` |
| `EntityType<T> T = EntityTypes.T(X.class)` | The type object constant of an Entity Type |
| `EntityType.create()` | An enhanced instance with defaults applied |
| `EntityType.createPlain()` | A plain instance |
| `EntityType.getProperties()` | The properties of the type |
| `Property` | One property, and how to read or write it generically |
| `EnumBase` and `EnumTypes.T(...)` | The same pattern for an enum |
| `@Abstract`, `@Initializer`, `@Transient`, `@TypeRestriction` | Change the type itself |
| `@SelectiveInformation`, `@ToStringInformation` | How an entity is shown |
| `@ForwardDeclaration` | The type belongs to another Model |

## See also

- [generic-model.md](generic-model.md) — the type system this fits into
- [models.md](models.md) — how types are grouped into a Model
- [model-apis.md](model-apis.md) — Metadata, which describes a type rather than shaping it
- [sessions.md](sessions.md) — why an enhanced entity matters
