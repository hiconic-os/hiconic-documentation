# Wire patterns and pitfalls

**What it is.** Two mechanisms that a Wire assembly usually needs — property lookups and test support — and the mistakes that Wire's design makes easy to fall into.

**When you need it.** Read this after you have written your first Space, and again when a wiring problem is hard to explain.

## Property lookups

An assembly usually reads configuration values. Wire turns a plain interface into a typed reader of those values, so a Space never parses a string.

Declare an interface. One method is one property, and the method name is the property name.

```java
public interface AddressBookProperties {
	@Default("addressbook")
	String databaseName();

	@Default("50")
	int poolSize();

	@Name("ADDRESSBOOK_ADMINS")
	Set<String> admins();

	@Required
	@Decrypt
	String databasePassword();
}
```

Create a reader by giving the interface and a source of raw values.

```java
AddressBookProperties properties = PropertyLookups.create(AddressBookProperties.class, System::getenv);
```

The source is any `Function<String, String>`: the environment, system properties, or a map you built.

### What a property method may declare

| Element | Effect |
|---|---|
| Method name | The property name |
| `@Name("OTHER")` | Uses that name instead |
| `@Default("...")` | The value when the property is missing |
| One parameter of the return type | The same, given at the call instead of in the annotation. Cannot be combined with `@Default`. |
| `@Required` | An error when the value is missing |
| `@Decrypt` | The value is decrypted before it is converted |

Without a value and without a default, an object type gives `null` and a primitive type gives its zero value. That is the trap `@Required` exists for.

### Supported types

| Kind | Types |
|---|---|
| Text and number | `String`, `boolean`, `char`, `byte`, `short`, `int`, `long`, `float`, `double`, their wrappers, `BigDecimal` |
| Time | `Date`, `Duration` |
| System | `File`, `Path`, `Class` |
| Enum | Any enum, by constant name |
| Collection | `List<E>`, `Set<E>`, `Map<K, V>` |

A collection value is a comma separated list, and each element is URL decoded. A map value is a comma separated list of `key=value` entries. Use URL encoding when an element contains a comma or an equals sign.

### Secrets

`@Decrypt` decrypts the value before conversion. `SecretResolution.VALUE` treats the `secret` of the annotation as the key itself; `SecretResolution.REFERENCE` treats it as the name of another property that holds the key. Keep a real key out of the annotation, and use `REFERENCE`.

## Wire in tests

Extend `AbstractWiredTest` and name the Module. The Wire Context is built before each test and closed after it.

```java
public class AddressBookTest extends AbstractWiredTest<AddressBookContract> {

	@Override
	protected WireTerminalModule<AddressBookContract> wireModule() {
		return new TestModule();
	}

	@Test
	public void addsAPerson() {
		contract().addressBook().add(person);
		...
	}
}
```

The Module under test binds the real Spaces, and replaces the ones a test must not use. Binding an instance is the shortest way to do that, because it needs no Space at all:

```java
builder.bindContract(PersistenceContract.class, new InMemoryPersistenceSpace());
```

See [wire-context.md](wire-context.md) for the override rule this relies on.

## Collection helpers

A Space often needs a small literal collection. `Lists`, `Maps` and `Sets` build one in a single expression, so a bean method needs no local variable for it.

```java
Lists.list("home", "work");
Sets.linkedSet("home", "work");
Maps.map(Maps.entry("home", homeAddress()), Maps.entry("work", workAddress()));
```

`linkedSet` and `linkedMap` keep the order you wrote. Use them when the order is part of the meaning.

## Pitfall: a @Managed method with arguments

Wire keeps one instance per distinct argument list, in a map that is never cleaned. A method that takes an argument from outside the assembly therefore grows without limit.

```java
// Wrong. One Managed Instance per person id, kept until the Wire Context ends.
@Managed
private PersonView personView(String personId) { ... }
```

Use a parameterized Managed Instance only for a small, closed set of values that the assembly itself decides. For anything per request or per user, take a `ScopeContext` parameter, and end it with `WireContext.close(scopeContext)`.

## Pitfall: state in a Space

A Space is built once per Wire Context, so a field on a Space lives as long as the assembly. It is not per call, and it is not thread safe.

A Space builds instances. It does not hold application state. If you need shared state, make it a Managed Instance and give it a type of its own.

## Pitfall: creating and configuring in one expression

Wire publishes an instance when the returned local variable is assigned. A method that returns an expression publishes nothing before it ends, and a cycle through it does not terminate. Write create, assign, configure, return. See [managed-instances.md](managed-instances.md).

## Pitfall: importing a Space instead of a Contract

`@Import` accepts a Space class. The Space then sees every public method of that Space, and cannot be rebound, so a test can no longer replace it. Import a Contract unless the Space you need has no Contract.

## Pitfall: using a Contract after shutdown

`shutdown()` destroys the Managed Instances. A reference you kept to an instance still points at a destroyed object, and a Contract call after shutdown gives a broken assembly rather than a clear error. Keep the lifetime of a Wire Context wider than the lifetime of anything that uses it, and prefer a try with resources block for a short lived one.

## Pitfall: overriding configureContext without super

A Module that overrides `configureContext` and does not call `WireTerminalModule.super.configureContext(builder)` loses the binding of its own package. The symptom is a Contract that is suddenly unbound, in a Module that did not change.

## Cheat sheet

| Symptom | Cause |
|---|---|
| A Contract is unbound in a Module that did not change | `configureContext` overridden without calling `super` |
| Memory grows over time | A `@Managed` method keyed by an argument from outside the assembly |
| A value is `0` or `false` and nobody set it | A missing property with a primitive type and no `@Required` |
| Two tests interfere | State in a Space, or a Wire Context that is not closed |
| A cycle does not terminate | A Managed Instance built and configured in one expression |
| The Space "was not instrumented" | The Space is in a package the Wire Context does not cover |

## See also

- [managed-instances.md](managed-instances.md) — scopes, cycles and parameterized instances
- [wire-context.md](wire-context.md) — overriding a binding, and shutting down
- [contracts-and-spaces.md](contracts-and-spaces.md) — `@Import` and binding
