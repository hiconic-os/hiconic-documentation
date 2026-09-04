# Modeled configuration

**What it is.** Configuration is an entity, not a property file. A configuration Model declares the shape, and a YAML file supplies the values.

**When you need it.** You need it whenever something must be configurable, because a modeled configuration is typed, documented and validated before the application starts.

This page covers configuration at Generic Model level. How a Reflex application assembles its configuration from several sources is in [../reflex/configuration.md](../reflex/configuration.md).

## Configuration as a Model

A configuration type is an ordinary Entity Type in a `-configuration-model` artifact.

```java
public interface AddressBookConfiguration extends GenericEntity {

	EntityType<AddressBookConfiguration> T = EntityTypes.T(AddressBookConfiguration.class);

	@Mandatory
	@Description("JDBC URL of the address book database")
	String getDatabaseUrl();
	void setDatabaseUrl(String databaseUrl);

	@Initializer("50")
	int getPageSize();
	void setPageSize(int pageSize);
}
```

Compare that with a map of strings.

| A map of strings | A configuration entity |
|---|---|
| Every value is text, and every reader parses it | Values are typed, and parsed once |
| A missing value is found at first use | `@Mandatory` is checked when the configuration is read |
| The default is repeated in every reader | `@Initializer` states it once |
| Nobody can list what is configurable | The Model is the list |
| Documentation lives elsewhere | `@Description` travels with the type |

The last two rows follow from the configuration being a Model: it can be read with the Model Oracle, and rendered as documentation or as a form. See [model-apis.md](model-apis.md).

## Reading a configuration

`ModeledConfiguration` reads the configuration for a type.

```java
AddressBookConfiguration config = modeledConfiguration.config(AddressBookConfiguration.T);
```

| Method | Returns |
|---|---|
| `config(Type.T)` | The configuration, or throws when it cannot be produced |
| `configReasoned(Type.T)` | A `Maybe`, with a Reason when it cannot |
| `config(Type.T, useCase)` | The configuration for one use case |

Prefer `configReasoned` at boot. A bad configuration is an expected failure, and the Reason says which property was wrong. See [reasons.md](reasons.md).

## From YAML to an entity

The values come from a YAML file, read by the YAML Marshaller. The file mirrors the type.

```yaml
databaseUrl: "jdbc:postgresql://localhost/addressbook"
pageSize: 100
```

Reading directly, without the platform around it:

```java
Maybe<AddressBookConfiguration> maybe = YamlConfigurations.read(AddressBookConfiguration.T)
		.placeholders()
		.from(configFile);
```

| Builder method | Effect |
|---|---|
| `from(file)`, `from(url)`, `from(stream)` | Where the YAML comes from |
| `placeholders()` | Resolve placeholders in the values |
| `placeholders(resolver)` | The same, with your own resolver |
| `noDefaulting()` | Do not apply the `@Initializer` defaults |
| `absentifyMissingProperties()` | A missing property becomes absent, not its default |
| `options(...)` | Any deserialization option |

`absentifyMissingProperties()` is what makes layering work: a value that a file does not mention stays absent, so a later layer can supply it, rather than overwriting it with a default.

## Placeholders

A value in the file may be a placeholder rather than a literal, and be resolved later.

```yaml
databaseUrl: "${DATABASE_URL}"
```

A placeholder is a Value Descriptor: an entity that stands for a value that is not known yet. Resolution walks the configuration and replaces each one. It can also be partial, which leaves the unresolved placeholders in place for a later stage.

This is why a configuration entity can be assembled before everything about the environment is known, and completed afterwards.

## Cheat sheet

| Element | Purpose |
|---|---|
| A `-configuration-model` artifact | Where a configuration type lives |
| `@Mandatory`, `@Initializer`, `@Description` | Required, default, documented |
| `ModeledConfiguration.config(Type.T)` | Read the configuration |
| `configReasoned(Type.T)` | Read it, with a Reason on failure |
| `YamlConfigurations.read(Type.T)` | Read one YAML file directly |
| `placeholders()` | Resolve placeholders in the values |
| `absentifyMissingProperties()` | Keep a missing value absent, for layering |
| Value Descriptor | An entity standing for a value not yet known |

## See also

- [marshalling.md](marshalling.md) — the YAML Marshaller underneath
- [model-apis.md](model-apis.md) — reading a configuration Model as documentation
- [../reflex/configuration.md](../reflex/configuration.md) — how an application assembles its configuration
