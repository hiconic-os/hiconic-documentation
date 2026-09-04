# Models

**What it is.** A Model is a named set of types together with its dependencies on other Models. One Model is one build artifact.

**When you need it.** You need a Model whenever you declare types, and you need to know where the Model boundaries run to keep the dependencies of an application in the right direction.

## A Model is an artifact

There is no way to declare a Model inside a larger artifact. The artifact **is** the Model: its types are the Model's types, and its dependencies on other Model artifacts are the Model's dependencies.

Three consequences follow, and they decide how a project is laid out.

- A type belongs to exactly one Model, so the decision where a type lives is a decision about who may use it.
- A Model can only see types of the Models it depends on, and the compiler enforces that, because the dependency is an ordinary artifact dependency.
- Moving a type between Models is a breaking change for everyone who depended on the old Model to get it.

A Model artifact holds interfaces and enums, and nothing else. No processing code, no implementation. That is what lets a Model be shared by a server, a client and a tool without dragging anything behind it.

## The base Models

| Model | Brings |
|---|---|
| `root-model` | `GenericEntity`, and the base of the type system. Every Model depends on it, directly or not. |
| `meta-model` | The types that describe a Model itself, so a Model can be handled as data |
| `resource-model` | `Resource` and its sources, for unstructured data. See [resource.md](resource.md). |

Beyond those, a Model depends on whichever Models declare the types it uses. A Model that declares Service Requests depends on `service-api-model`; one that reports failures depends on the Reason Models. See [services.md](services.md) and [reasons.md](reasons.md).

## Naming convention

The artifact suffix says what kind of types are inside, and a reader relies on it.

| Suffix | Contains |
|---|---|
| `-model` | Data types: the entities an application stores and exchanges |
| `-api-model` | Service Requests and their responses: the callable API |
| `-configuration-model` | The configuration entities of one component |

Keep them apart even when a project is small. The data types outlive the API, and the API is what a client depends on. A single artifact holding both forces every client to depend on the storage model.

## Building a Model

The build compiles the interfaces, and adds two things to the artifact.

- A declaration of the Model, so that the platform can find the Model and its types at runtime without scanning the classpath.
- A generated class named after the Model, with the Model name in a constant. It gives you the name without a string literal.

```java
Model model = GMF.getTypeReflection().getModel(_HelloWorldModel_.name);
```

No implementation classes are generated. You write interfaces, and the platform provides the classes behind them at runtime. See [entity-types.md](entity-types.md).

## Model and GmMetaModel

A Model exists in two forms, and both are useful.

| Form | Is | Use it to |
|---|---|---|
| `Model` | The runtime handle | Find the Model, its dependencies, and its declared Java types |
| `GmMetaModel` | The Model as data, an entity graph like any other | Read the Model, ship it, store it, attach Metadata to it |

`Model.getMetaModel()` moves from the first to the second.

```java
Model model = GMF.getTypeReflection().getModel(_HelloWorldModel_.name);
GmMetaModel metaModel = model.getMetaModel();
```

The data form is what a generic component reads. A Model can be sent to a client that was never compiled against it, and Metadata is attached to the `GmMetaModel` elements rather than to Java classes. Reading a `GmMetaModel` directly is rarely the shortest way to answer a question about a Model; the Model Oracle is. See [model-apis.md](model-apis.md).

## Cheat sheet

| Element | Purpose |
|---|---|
| `root-model` | The Model every other Model builds on |
| `meta-model` | The types that describe a Model |
| `-model`, `-api-model`, `-configuration-model` | Data types, API types, configuration types |
| `Model` | The runtime handle of a Model |
| `Model.getMetaModel()` | The Model as data |
| `GmMetaModel` | The Model in its data form |
| `_XyzModel_.name` | The generated constant holding the Model name |
| `GMF.getTypeReflection().getModel(name)` | Look a Model up by name |

## See also

- [entity-types.md](entity-types.md) — what goes into a Model
- [model-apis.md](model-apis.md) — reading a Model, and describing it with Metadata
- [../reflex/project-layout.md](../reflex/project-layout.md) — how Model artifacts sit in a project
