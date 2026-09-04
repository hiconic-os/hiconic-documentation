# Cloning and traversing

**What it is.** Every value in Generic Model is a graph of entities and collections. Traversing walks that graph, and cloning copies it, with control over where the copy stops.

**When you need it.** You need it whenever data crosses a boundary: out of a Session, into a response, into a Marshaller, or between two Sessions.

## Why generic traversal exists

Copy a graph, write it to a stream, cut it at a boundary, find every entity of a kind: these look like four problems. They are one walk over the same graph, with a different action at each step.

Generic Model writes that walk once. Every type object can walk a value of its type, and the caller supplies what happens at each element. Marshalling, copying, cutting and searching are then variations of one mechanism.

## Cloning

`clone` is on the type object, so it works for a single entity and for a whole graph.

```java
Person copy = Person.T.clone(person, null, StrategyOnCriterionMatch.skip);
```

Three things decide the result:

| Argument | Decides |
|---|---|
| The value | What is copied |
| The Matcher, or a Traversing Criterion | Where the walk stops |
| The `StrategyOnCriterionMatch` | What happens at the place it stops |

Cloning follows the graph, so a cycle is safe: an entity already copied is not copied twice, and the copy points at the copy.

## CloningContext

The three argument form is the short one. The full form takes a `CloningContext`, which decides every step of the copy.

| Method | Decides |
|---|---|
| `supplyRawClone` | How a copy is created. This is where a copy is created in a Session instead of free floating. |
| `canTransferPropertyValue` | Whether one property is copied at all |
| `postProcessCloneValue` | A last change to each copied value |
| `createAbsenceInformation` | What a property holds when it is not copied |
| `preProcessInstanceToBeCloned` | A substitute for the entity about to be copied |

`ConfigurableCloningContext` builds one without a subclass:

```java
CloningContext cc = ConfigurableCloningContext.build()
		.supplyRawCloneWith(session)
		.skipIndentifying(true)
		.withMatcher(matcher)
		.done();
```

`supplyRawCloneWith(session)` is the usual reason to reach for it: the copy is created in a Session, so it is attached and tracked from the first property. `skipIndentifying` leaves the identity properties out, which turns a copy into a new entity rather than a second entity with the same id.

## What cloning is used for

| Task | Why cloning |
|---|---|
| Detach data from a Session | The copy has no Session, so nothing is tracked and nothing is written back |
| Move data between Sessions | The copy is created in the target Session |
| Return data from a service | The response must not carry the whole reachable graph |
| Turn an entity into a template for a new one | Copy without the identity properties |

## Traversing

Traversing walks without producing anything. You give a `TraversingVisitor`, and it is called at every element.

```java
Person.T.traverse(person, matcher, tc -> {
	// called at every element of the graph
});
```

A `TraversingContext` is passed to the visitor and tells it where the walk currently is:

| Method | Answers |
|---|---|
| `getTraversingStack()` | The path from the root to here, as criteria |
| `getObjectStack()` | The values along that path |
| `getCurrentCriterionType()` | Whether this step is a root, an entity, a property or a collection element |
| `isVisited(entity)` | Whether the walk has already seen this entity |
| `registerAsVisited` and `getAssociated` | Attach your own object to an entity, and read it back |

The visited set is what keeps a walk finite in a graph with cycles.

## Traversing Criteria

A Traversing Criterion selects a position in the graph, not a value. It matches against the path the walk has taken.

The `TC` builder assembles one:

```java
TraversingCriterion tc = TC.create()
		.negation()
			.disjunction()
				.property(GenericEntity.id)
				.property(GenericEntity.partition)
			.close()
		.done();
```

| Element | Matches |
|---|---|
| `property()`, `property("name")` | Any property, or one by name |
| `propertyType(...)`, `propertyWithType(...)` | A property by its type, or by name and type |
| `entity()`, `entity(Type.T)` | Any entity, or one type |
| `typeCondition(...)` | A type that satisfies a condition |
| `root()`, `listElement()`, `setElement()`, `mapKey()`, `mapValue()` | A position in the graph |
| `pattern()` | A sequence of steps, rather than a single one |
| `conjunction()`, `disjunction()`, `negation()` | Combine the above |
| `joker()`, `recursion(min, max)` | Any step, or a repeated one, inside a pattern |

A criterion built with `pattern()` matches a path: "a `HasAcl` entity, then its `owner` property". A criterion without a pattern matches one step wherever it occurs.

## StrategyOnCriterionMatch

When the walk matches, the strategy decides what the copy holds at that place.

| Strategy | The result holds |
|---|---|
| `skip` | Nothing. The property keeps its default value, and nobody can tell it was left out. |
| `partialize` | An Absence Information, so the property is marked as not loaded |
| `reference` | A reference to the entity instead of the entity itself |

Use `partialize` when data leaves a Session. With `skip`, a cut collection arrives as an empty collection, and the reader cannot tell it apart from one that is genuinely empty. With `partialize`, the reader can.

## Absence Information

An `AbsenceInformation` sits in a property in place of its value and says: this was not loaded. `Property.isAbsent(entity)` reports it, and `AbsenceInformation.getSize()` may carry how many elements a collection would have had.

A partly loaded entity stays a valid entity. It can be shown, sent on, and stored again, as long as the code respects absence rather than reading through it. This is what makes it possible to load a large graph in parts.

## The traversing engine

The type object covers one walk with one visitor. `GMT` covers the configured cases, and is the entry point when a plain `clone` is not enough.

```java
Person copy = GMT.clone(person);

GMT.doClone()
		.visitor(myVisitor)
		.doFor(person);

GMT.traverse(myVisitor, person);
```

Use `GMT` when the walk needs several visitors, a skipping rule, or a custom property transfer. Use `EntityType.clone` for everything simpler.

## Model paths

A `ModelPath` names a position inside a graph, as a list of elements: a root, then a property, then a collection element, and so on. Where a Traversing Criterion describes positions in general, a Model Path is one concrete position, and it can be stored and passed around.

## Cheat sheet

| Type | Purpose |
|---|---|
| `EntityType.clone(...)` | Copy a value, with a matcher and a strategy |
| `CloningContext` | Full control over how a copy is made |
| `ConfigurableCloningContext.build()` | Build one without a subclass |
| `TraversingVisitor` | Called at every element of a walk |
| `TraversingContext` | Where the walk is, and what it has seen |
| `TraversingCriterion` and `TC.create()` | Which positions in the graph are selected |
| `StrategyOnCriterionMatch` | `skip`, `partialize`, `reference` |
| `AbsenceInformation` | Marks a property as not loaded |
| `GMT` | The configured clone and traverse engine |
| `ModelPath` | One concrete position in a graph |

## See also

- [entity-types.md](entity-types.md) — the properties a walk visits
- [sessions.md](sessions.md) — where a detached copy comes from
- [marshalling.md](marshalling.md) — the same walk, writing to a stream
