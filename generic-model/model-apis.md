# Model APIs: Metadata, Model Oracle and Cmd Resolver

**What it is.** Metadata is configuration attached to a Model element. The Model Oracle reads the structure of a Model, and the Cmd Resolver answers which Metadata is effective in a given situation.

**When you need it.** You need Metadata to configure a type without changing it, the Model Oracle to inspect a Model, and the Cmd Resolver to ask what applies here and now.

## Metadata is modeled data

A piece of Metadata is an entity. It is attached to a Model element — a Model, an Entity Type, a property, an enum type or an enum constant — and says something about it that the type system itself does not.

Metadata is not part of the modeling. The type says what the data is; Metadata configures how that type is treated, and can differ per deployment, per use case or per role while the type stays the same.

Because Metadata is data, it travels. A client that receives a Model receives its Metadata too, and can build a form from it without knowing the domain.

Every Metadata type extends `MetaData` and inherits four properties.

| Property | Purpose |
|---|---|
| `selector` | A condition. Without one, the Metadata always applies. |
| `conflictPriority` | Which of two candidates wins |
| `inherited` | Whether a subtype also gets it. True by default. |
| `important` | Marks it as significant to a generic user interface |

A **Predicate** is Metadata with no data of its own: it is either there or not, such as `Mandatory`. Each Predicate has a counterpart that switches it off again, such as `Optional`. That pair is how a subtype or a use case removes what a supertype declared.

## Declaring Metadata by annotation

The short way is an annotation on the interface. The platform turns it into the Metadata entity.

```java
public interface Person extends GenericEntity {

	EntityType<Person> T = EntityTypes.T(Person.class);

	@Name("Full name")
	@Description("The name as it appears on the passport")
	@Mandatory
	@MaxLength(200)
	String getName();
	void setName(String name);
}
```

The common ones, by group:

| Group | Annotations |
|---|---|
| Prompt | `@Name`, `@Description`, `@Hidden`, `@Confidential`, `@Deprecated`, `@Priority`, `@Placeholder` |
| Constraint | `@Mandatory`, `@Optional`, `@Unique`, `@Min`, `@Max`, `@MinLength`, `@MaxLength`, `@Pattern`, `@Unmodifiable`, `@Bidirectional` |
| Display | `@Color`, `@Emphasized`, `@SelectiveInformation` |
| Mapping | `@Alias`, `@PositionalArguments`, `@WordCasing`, `@WordSeparator` |

An annotation is right when the statement is true wherever the Model is used. It is part of the Model, so it ships with it.

## Declaring Metadata by configuration

The other way is to attach the Metadata entity to the Model at boot, in an initializer. Nothing is declared in the interface.

Use configuration when the same Model must behave differently in different deployments: a property that is mandatory for one customer, a type hidden for one role, a name that differs per language. See [../reflex/persistence.md](../reflex/persistence.md).

Both ways produce the same thing. A reader of Metadata cannot tell which was used.

## Custom Metadata annotations

You can give your own Metadata type an annotation. Declare the annotation and the Metadata Entity Type, then register the pair in a file named `META-INF/gmf.mda` in the artifact.

One line per annotation, comma separated:

```
com.example.model.annotation.Sensitive,com.example.model.meta.Sensitive,value,level
```

The first entry is the annotation, the second is the Metadata type. The rest are pairs: an annotation member, then the property of the Metadata entity it fills. A Predicate needs no pairs.

## Selectors

A selector makes Metadata conditional. It is an entity too, so the condition ships with the Metadata.

| Selector | Applies when |
|---|---|
| `UseCaseSelector` | The caller declared that use case |
| `RoleSelector` | The user has one of the roles |
| `AccessSelector`, `AccessTypeSelector` | The data comes from that Access |
| `PropertyNameSelector`, `PropertyRegexSelector`, `PropertyTypeSelector` | The property matches |
| `EntityTypeSelector`, `EntitySignatureRegexSelector` | The entity matches |
| A property discriminator | A property of the entity has a given value |
| `ConjunctionSelector`, `DisjunctionSelector`, `NegationSelector` | Combine the above |

A property discriminator makes Metadata depend on the entity in hand, so a property can be mandatory only when another property of the same entity has a given value. The other selectors depend on the type or the caller, not on the data.

When two pieces of Metadata of the same type apply, `conflictPriority` decides. Set it only where a conflict is real. A Model full of priorities is hard to reason about.

## Cmd Resolver

The Cmd Resolver answers the question a caller actually has: which Metadata applies here? It walks the Model, the supertypes, the selectors and the priorities, and returns one answer.

```java
ModelMdResolver md = cmdResolver.getMetaData();

boolean mandatory = md.entityType(Person.T)
		.property(Person.name)
		.useCase("web")
		.is(Mandatory.T);

Name name = md.entityType(Person.T)
		.property(Person.name)
		.meta(Name.T)
		.exclusive();
```

You narrow from the Model down to the element, add context, then ask.

| Step | Methods |
|---|---|
| Narrow | `entityType(...)`, `entity(...)`, `property(...)`, `enumType(...)`, `enumConstant(...)` |
| Add context | `useCase(...)`, `access(...)`, `with(Aspect.class, value)` |
| Ask | `is(Predicate.T)` for a Predicate, `meta(Type.T)` for Metadata with data |
| Loosen | `lenient(true)`, `ignoreSelectors()`, `ignoreSelectorsExcept(...)` |

Give the context, not the answer. `useCase("web")` is what makes the selectors work. Without it, Metadata guarded by a use case is invisible.

## Model Oracle

The Model Oracle reads the structure of a Model: which types exist, what they declare, and which Models they came from. It answers questions about a Model without a single instance.

```java
ModelOracle oracle = cmdResolver.getModelOracle();

Set<GmEntityType> allTypes = oracle.getEntityTypeOracle(GenericEntity.T)
		.getSubTypes()
		.includeSelf()
		.transitive()
		.asGmTypes();
```

| From | You get |
|---|---|
| `getTypes()` | Every type in the Model |
| `getDependencies()` | The Models this one depends on |
| `getEntityTypeOracle(...)` | One Entity Type: its properties, subtypes and supertypes |
| `getEnumTypeOracle(...)` | One enum type and its constants |

Each of those returns a small builder with `transitive()`, `includeSelf()`, a filter, and a terminal `asGmTypes()` or `asTypes()`. That is how a tool walks a Model it knows nothing about.

Use the Model Oracle for structure, and the Cmd Resolver for meaning. `CmdResolver` gives you both, over one Model.

## Cheat sheet

| Element | Purpose |
|---|---|
| `MetaData` | The base of every Metadata type |
| `Predicate` and its erasure | Metadata that is only on or off, and how to switch it off |
| `@Name`, `@Description`, `@Mandatory` and the rest | Declare Metadata in the interface |
| `META-INF/gmf.mda` | Register your own annotation for your own Metadata type |
| `MetaDataSelector` | Makes Metadata conditional |
| `conflictPriority` | Decides between two candidates |
| `CmdResolver` | The entry point to both Metadata and structure |
| `CmdResolver.getMetaData()` | The Metadata resolver |
| `MdResolver.is(...)` and `.meta(...)` | Ask for a Predicate, or for Metadata with data |
| `CmdResolver.getModelOracle()` | The structure of the Model |
| `ModelOracle.getEntityTypeOracle(...)` | One type: properties, subtypes, supertypes |

## See also

- [models.md](models.md) — what a Model is
- [entity-types.md](entity-types.md) — the annotations that shape a type, rather than describe it
- [access.md](access.md) — where Metadata decides how data is stored and queried
