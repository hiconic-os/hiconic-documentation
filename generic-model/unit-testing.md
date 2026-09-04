# Unit testing

**What it is.** A set of artifacts that give a test a Model, an Access, a Session and assertions that understand entities, without a running application.

**When you need it.** You need it for every test that touches Generic Model data: a processor, a query, a Marshaller, or logic that reads and writes entities.

## The test artifact

Tests live in their own artifact, named after the tested artifact with a `-test` suffix, and declared with the `test` archetype. The artifact under test never depends on its test artifact.

One dependency brings the whole test tool set:

```xml
<dependency>
	<groupId>com.braintribe.gm</groupId>
	<artifactId>gm-unit-test-deps</artifactId>
	<version>${V.com.braintribe.gm}</version>
</dependency>
```

| It brings | For |
|---|---|
| JUnit 4 and AssertJ | The test framework and the base assertions |
| `gm-assertj-assertions` | Assertions for entities and for `Maybe` |
| `test-model-test-tools` | A ready made test Model, and through it `gm-test-tools` |
| `gm-core4-jvm` | The Generic Model runtime for the JVM |

Tests use JUnit 4: `org.junit.Test` and `org.junit.Before`.

## A Model for a test

`NewMetaModelGeneration` creates a `GmMetaModel` for the Entity Types you give it. There is no Model artifact and no build step, so a test declares exactly the types it needs.

```java
NewMetaModelGeneration mmg = new NewMetaModelGeneration();

GmMetaModel model = mmg.buildMetaModel("test:AddressBookModel", asList(Person.T, Company.T));
```

The two argument form makes the new Model depend on the root Model. Pass the dependencies yourself to layer Models on each other:

```java
GmMetaModel base = mmg.buildMetaModel("test:PersonModel", asList(Person.T), asList(mmg.rootMetaModel()));
GmMetaModel full = mmg.buildMetaModel("test:AddressBookModel", asList(Company.T), asList(base));
```

`withValidation()` on the generator checks that every type reachable from the Model is either listed in the build call or comes from a stated dependency. Use it while writing the test. It is a debugging aid: after a failed validation the generator is no longer consistent and must not be reused.

The result holds the types and nothing else. Add Metadata to it in the test when the behaviour under test depends on Metadata. See [model-apis.md](model-apis.md).

When the test needs types, but not particular ones, use the shared test Model instead of declaring your own:

```java
GmMetaModel model = TestModelTestTools.createTestModelMetaModel();
```

It covers simple types, primitives, collections, enums, inheritance and multiple inheritance, which is what a Marshaller or a type system test needs.

## An Access and a Session

`GmTestTools` builds a Smood Access in memory and a Session on it.

```java
SmoodAccess access = GmTestTools.newSmoodAccessMemoryOnly("test.access", model);
PersistenceGmSession session = GmTestTools.newSession(access);
```

| Call | Gives |
|---|---|
| `newSmoodAccessMemoryOnly(accessId, model)` | An Access that keeps everything in memory |
| `newSmoodAccessMemoryOnly()` | The same, with the default id `test.access` and no Model |
| `newSmoodAccessWithTemporaryFile(accessId)` | An Access that writes to a temporary file, so a test can check what was persisted |
| `newSession(access)` | A Session on that Access. It also sets the Model Accessory from the Access Model, so Metadata resolution works. |
| `newSessionWithSmoodAccessMemoryOnly()` | Both at once, when the test does not need the Access itself |
| `newSmood(model)` | The bare in memory store, without an Access around it |

Take the Access separately whenever the test needs two Sessions on the same data. That is how you check that one Session sees what another committed.

```java
PersistenceGmSession writer = GmTestTools.newSession(access);
PersistenceGmSession reader = GmTestTools.newSession(access);
```

Two points that cost time when missed:

- A memory only Access ignores partitions you assign yourself. If the test assigns them, call `getDatabase().setIgnorePartitions(false)` on the Access first. Otherwise the assignment is dropped, and induced Manipulations are generated instead.
- `disableSynchronization(access)` removes the read write lock. Use it only in a single threaded test where the lock affects the measurement.

## Resources in a test

A Session does not resolve a Resource payload on its own. Give it a storage that keeps the bytes in memory.

```java
InMemoryResourceAccessFactory resources = GmTestTools.newInMemoryResourceAccessFactory();
PersistenceGmSession session = GmTestTools.newSessionWithInMemoryResources(access, resources);
```

One factory stands for the storage of one Access. Pass the same factory to every Session on that Access, or one Session cannot read what another wrote. The Model of the Access must contain `Resource` and `BlobSource`. See [resource.md](resource.md).

## Asserting on an entity

`GmAssertions` extends the AssertJ assertions, so one static import covers both the general and the entity assertions.

```java
import static com.braintribe.testing.junit.assertions.gm.assertj.core.api.GmAssertions.assertThat;
```

| Assertion | Checks |
|---|---|
| `isExactly(Type.T)` | The entity is of exactly that type |
| `isInstanceOf(Type.T)` | The entity is of that type or a subtype |
| `hasId()`, `hasGlobalId()`, `hasPartition()` | The identity property is set |
| `hasNoId()`, `hasNoGlobalId()`, `hasNoPartition()` | It is not set |
| `hasIdMatchingIdOf(other)` | Two entities carry the same id |
| `hasPropertyValue(name, value)` | One property holds that value |
| `hasAbsentProperty(name)`, `hasPresentProperty(name)` | Whether the property was loaded |
| `hasSessionAttached()`, `hasNoSessionAttached()` | Whether the entity belongs to a Session |

`isInstanceOf(Type.T)` takes an Entity Type, not a Java class, so it works on a value the test received as `Object`.

```java
assertThat(manipulation).isInstanceOf(InstantiationManipulation.T);
```

The absence assertions are the reason this library exists. A partly loaded entity cannot be checked with an equality assertion, because an absent property and an empty one look the same from outside. See [cloning-and-traversing.md](cloning-and-traversing.md).

## Asserting on a Maybe

A method that returns `Maybe` is checked without unpacking it.

```java
assertThat(maybe)
		.isSatisfied()
		.hasNonNullValue();

assertThat(maybe)
		.isUnsatisfied()
		.isUnsatisfiedBy(NotFound.T);

assertThat(maybe).hasReasonWhich()
		.hasPropertyValue(PersonNotFound.personId, "p-1");
```

| Assertion | Checks |
|---|---|
| `isSatisfied()`, `isUnsatisfied()` | Which case the `Maybe` is in |
| `hasValue()`, `hasNonNullValue()` | A value is present, and is not null |
| `isIncomplete()` | It carries a value and a Reason |
| `hasReason()`, `hasReasonOfType(Type.T)`, `isUnsatisfiedBy(Type.T)` | The Reason, and its type |
| `hasReasonWhich()` | Continues as an entity assertion on the Reason itself |

Assert on the Reason type, not on its text. The type is the contract; the text is not. See [reasons.md](reasons.md).

## A complete test

```java
public class AddressBookTest {

	private SmoodAccess access;
	private PersistenceGmSession session;

	@Before
	public void setup() {
		GmMetaModel model = new NewMetaModelGeneration() //
				.buildMetaModel("test:AddressBookModel", asList(Person.T, Company.T));

		access = GmTestTools.newSmoodAccessMemoryOnly("test.access", model);
		session = GmTestTools.newSession(access);
	}

	@Test
	public void storesPerson() {
		Person person = session.create(Person.T);
		person.setName("Alice");
		session.commit();

		PersistenceGmSession reader = GmTestTools.newSession(access);
		Person found = reader.query().entities(EntityQueryBuilder.from(Person.T).done()).unique();

		assertThat(found)
				.hasId()
				.hasPropertyValue(Person.name, "Alice");
	}
}
```

The second Session is what makes the test meaningful. Reading back through the Session that wrote the data would pass even if nothing reached the Access.

## Cheat sheet

| Element | Purpose |
|---|---|
| `-test` artifact, `test` archetype | Where tests live |
| `gm-unit-test-deps` | One dependency for JUnit, AssertJ, GM assertions and test tools |
| `new NewMetaModelGeneration().buildMetaModel(name, types)` | A Model from Entity Types, with no artifact |
| `withValidation()` | Check while building that every reachable type is declared or comes from a dependency |
| `TestModelTestTools.createTestModelMetaModel()` | The shared test Model |
| `GmTestTools.newSmoodAccessMemoryOnly(id, model)` | An Access in memory |
| `GmTestTools.newSmoodAccessWithTemporaryFile(id)` | An Access backed by a temporary file |
| `GmTestTools.newSession(access)` | A Session on an Access, with the Model Accessory set |
| `GmTestTools.newInMemoryResourceAccessFactory()` | Resource payloads in memory |
| `GmAssertions.assertThat(entity)` | Entity assertions |
| `GmAssertions.assertThat(maybe)` | `Maybe` assertions |

## See also

- [sessions.md](sessions.md) — what a Session records, and what `commit()` sends
- [access.md](access.md) — the Smood Access these tools build
- [reasons.md](reasons.md) — the Reasons a `Maybe` assertion checks for
- [../reflex/testing.md](../reflex/testing.md) — testing a Module against a running platform
