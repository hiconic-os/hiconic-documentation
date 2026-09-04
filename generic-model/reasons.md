# Reasons and Maybe

**What it is.** A Reason is a modeled description of why something did not succeed. `Maybe<T>` carries either a value or a Reason, so a failure is a value and not an exception.

**When you need it.** You need it for every failure a caller can expect: not found, not allowed, invalid input. Exceptions stay for defects.

## Why value based error handling

An expected failure is part of what a method does, so it belongs in the return type.

| With an exception | With a Reason |
|---|---|
| The failure is invisible in the signature | The failure is in the return type |
| The cause is a Java class, and cannot leave the process | The cause is an entity, and can be marshalled, stored and shown |
| A caller must know which exceptions to catch | A caller must handle the unsatisfied case to reach the value |
| A message is a string | A Reason has typed properties, so a client can act on it |

The last two matter most. A Reason crosses a network boundary as data, so a REST client sees the same failure the server produced, rather than a status code and a sentence. See [../reflex/rest-api.md](../reflex/rest-api.md).

## Reason

`Reason` is an Entity Type, so a Reason is an entity like any other.

| Property | Holds |
|---|---|
| `text` | The description, for a human |
| `reasons` | The Reasons that caused this one |

Because `reasons` is a list of Reasons, a failure is a tree and not a single message. Each level says what failed at that level, and the level below says why.

A Reason type is declared like any other Entity Type, and adds the properties a caller can act on.

```java
public interface PersonNotFound extends NotFound {

	EntityType<PersonNotFound> T = EntityTypes.T(PersonNotFound.class);

	String getPersonId();
	void setPersonId(String personId);
}
```

## Maybe

`Maybe<T>` is the return type. It holds a value, a Reason, or both.

| State | Means |
|---|---|
| Satisfied | The value is there, and complete |
| Empty | No value. A Reason says why. |
| Incomplete | A value **and** a Reason. The value is usable, but something was left out or degraded. |

The third state is how a method returns a partial result together with the reason it is partial, instead of choosing between a value and a failure.

```java
Maybe<Person> maybe = personService.find(personId);

if (maybe.isUnsatisfied())
	return maybe.propagateReason();

Person person = maybe.get();
```

| Method | Purpose |
|---|---|
| `isSatisfied()`, `isUnsatisfied()` | Which case you are in |
| `hasValue()`, `isEmpty()`, `isIncomplete()` | Whether a value exists, including the incomplete case |
| `get()` | The value. Throws `ReasonException` when there is none. |
| `value()` | The value without the check |
| `whyUnsatisfied()` | The Reason |
| `isUnsatisfiedBy(Type.T)`, `isUnsatisfiedAny(...)` | Whether the Reason is of a given type |
| `propagateReason()` | The same failure, typed for your own return type |
| `map`, `flatMap` | Continue with the value, and carry the failure through untouched |
| `ifSatisfied`, `ifValue`, `ifUnsatisfied` | Act on one case without branching |

`propagateReason()` is what keeps a chain of calls short. It converts the type parameter and leaves the Reason alone, so a method can pass a failure up without unpacking it.

## Building a Reason

`Reasons.build(...)` builds a Reason and, in one step, the `Maybe` around it.

```java
return Reasons.build(PersonNotFound.T)
		.text("No person with id " + personId)
		.assign(PersonNotFound::setPersonId, personId)
		.toMaybe();
```

| Method | Purpose |
|---|---|
| `text(...)` | The description |
| `assign(Setter, value)` | One typed property of the Reason |
| `cause(...)`, `causes(...)` | The Reasons underneath this one |
| `enrich(...)` | Anything else, with the Reason in hand |
| `toReason()` | The Reason itself |
| `toMaybe()` | An empty `Maybe` carrying it |
| `toMaybe(value)` | An incomplete `Maybe`: a value **and** the Reason |

`Reasons.create(Type.T, text)` is the short form when there is nothing but a text.

## Standard Reasons

Two Models carry the Reasons that any application can use. Extend one of them instead of inventing a parallel vocabulary, so that generic code — a REST layer, a log, a client — can recognize the kind of failure.

| Model | Reasons |
|---|---|
| `essential-reason-model` | `NotFound`, `AlreadyExists`, `InvalidArgument`, `UnsupportedOperation`, `Canceled`, `Timeout`, `CommunicationError`, `ConfigurationError`, `IoError`, `FilesystemError`, `ParseError`, `InternalError` |
| `security-reason-model` | `Forbidden`, `AuthenticationFailure`, `InvalidCredentials`, `MissingCredentials`, `InvalidSession`, `MissingSession`, `SessionExpired`, `SessionNotFound`, `SecondFactorRequired` |

`InternalError` is the one that means a defect rather than an expected failure. It is what an exception becomes when it must cross a boundary.

## Aggregation and nesting

A Reason keeps its causes, so the chain is preserved from the place that failed to the place that reports it.

```java
return Reasons.build(ImportFailed.T)
		.text("Could not import the address book")
		.cause(maybe.whyUnsatisfied())
		.toMaybe();
```

`stringify()` on a Reason renders the whole tree as indented text, which is what belongs in a log. `asString()` gives a single line.

Add a level when your level knows something the level below does not. Do not add one that only repeats the cause in other words.

## Exception or Reason

| Situation | Use |
|---|---|
| A caller can reasonably expect this, and could act on it | A Reason |
| The failure crosses a service boundary | A Reason |
| A programming error: a broken invariant, a null that cannot be null | An exception |
| A library you call throws, and you cannot continue | Catch it, and turn it into a Reason at the boundary |

Two helpers bridge the two worlds when a call sits inside code that cannot return a `Maybe`.

- `maybe.get()` throws a `ReasonException` when the `Maybe` is unsatisfied.
- `UnsatisfiedMaybeTunneling.getOrTunnel(maybe)` throws an exception that carries the whole `Maybe`, so an outer layer can unwrap it and return the original Reason unchanged.

Use tunneling inside one implementation, never across an API. An API that throws where it could return a `Maybe` forces its callers back to exceptions.

## Cheat sheet

| Element | Purpose |
|---|---|
| `Reason` | A modeled failure, with `text` and nested `reasons` |
| `Maybe<T>` | A value, a Reason, or both |
| `Maybe.get()` | The value, or a `ReasonException` |
| `Maybe.whyUnsatisfied()` | The Reason |
| `Maybe.propagateReason()` | Pass a failure up, retyped |
| `Reasons.build(Type.T)` | Build a Reason and its `Maybe` |
| `Reasons.create(Type.T, text)` | The short form |
| `Reason.stringify()` | The whole tree, as indented text |
| `essential-reason-model` | The standard failures |
| `security-reason-model` | The standard security failures |
| `UnsatisfiedMaybeTunneling` | Carry an unsatisfied `Maybe` through code that cannot return one |

## See also

- [services.md](services.md) — where a Reason is returned from
- [entity-types.md](entity-types.md) — how a Reason type is declared
- [../reflex/rest-api.md](../reflex/rest-api.md) — how a Reason reaches an HTTP client
- [unit-testing.md](unit-testing.md) — asserting on a `Maybe` and its Reason
