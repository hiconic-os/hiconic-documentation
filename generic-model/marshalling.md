# Marshalling

**What it is.** A Marshaller writes an entity graph to a stream and reads it back. Because the type system is known, the output can carry its own type information.

**When you need it.** You need it for every transport and every file: a REST response, a configuration file, a message payload, a stored request.

## The Marshaller interface

One interface serves every format.

```java
Marshaller marshaller = new YamlMarshaller();

marshaller.marshall(outputStream, person);

Person read = (Person) marshaller.unmarshall(inputStream);
```

| Method | Purpose |
|---|---|
| `marshall(out, value, options)` | Write a value |
| `unmarshall(in, options)` | Read one back |
| `unmarshallReasoned(in, options)` | Read one back, with a Reason instead of an exception on bad input |

A Marshaller handles any value the type system knows: an entity, a collection, a simple value, or a whole graph. It is not written for one Model, so the same Marshaller serves a Model it has never seen.

`MarshallerRegistry` picks the Marshaller for a media type, which is how a web layer honors what a client asked for.

## The formats

| Format | Use it for |
|---|---|
| YAML | Configuration, and anything a person reads or edits |
| JSON | The web, and anything a browser consumes |

Both are complete: anything one writes, it can read back.

## Type information

A graph is polymorphic. A property declared as `Person` may hold an `Employee`, and a property declared as `Object` may hold anything. So the output has to say what a value is, or the reader cannot rebuild it.

`TypeExplicitness` decides how much is written.

| Value | Writes the type |
|---|---|
| `auto` | Where the reader would otherwise not know. The default. |
| `polymorphic` | Where the declared type has subtypes |
| `entities` | For every entity |
| `always` | Everywhere |
| `never` | Nowhere |

`never` produces the plain JSON an external consumer expects, and gives up reading it back into the right types. Use it at the edge of the system, not inside it.

## Options that change the result

| Option | Effect |
|---|---|
| `outputPrettiness()` | Indented output, or compact |
| `writeEmptyProperties()` | Whether a null or an empty collection appears at all |
| `writeAbsenceInformation()` | Whether an absent property is written as absent |
| `stabilizeOrder()` | A deterministic order, so two runs give the same bytes |
| `useDirectPropertyAccess()` | Bypass the interceptor while reading properties |

On the way back in:

| Option | Effect |
|---|---|
| `absentifyMissingProperties()` | A property missing in the input becomes absent, not null |
| `session(...)` | Entities are created in that Session, and attached |
| `decodingLenience(...)` | Unknown types or properties are tolerated instead of failing |
| `identityManagement(...)` | How a repeated entity in the input is recognized as the same one |

`stabilizeOrder()` is what makes a marshalled file comparable in a version control system. `decodingLenience` is what lets an old reader accept a newer Model.

Options are built by deriving from the defaults:

```java
GmSerializationOptions options = GmSerializationOptions.deriveDefaults()
		.outputPrettiness(OutputPrettiness.high)
		.stabilizeOrder(true)
		.writeAbsenceInformation(true)
		.build();
```

## Absence and identity

Two properties of the type system survive a round trip, and both matter.

**Absence.** A property that was never loaded is not the same as a property that is empty. With `writeAbsenceInformation()` the difference is written, and with `absentifyMissingProperties()` it is restored. Without them, a partly loaded graph silently becomes a graph full of empty values. See [cloning-and-traversing.md](cloning-and-traversing.md).

**Identity.** One entity referenced from three places is one entity, not three copies. Identity management writes it once and refers back to it, and rebuilds the same shape on reading. Turning it off inflates the output and breaks cycles.

## A Resource in a marshalled graph

A marshalled graph carries the `Resource` entity, not its bytes. The entity says what the file is and where its payload lives; the payload travels separately, or not at all.

That is what keeps a response with a hundred documents small. See [resource.md](resource.md).

## Cheat sheet

| Element | Purpose |
|---|---|
| `Marshaller` | Write and read a value of any type |
| `unmarshallReasoned(...)` | Read with a Reason instead of an exception |
| `MarshallerRegistry` | The Marshaller for a media type |
| YAML Marshaller | Configuration, and output a person reads or edits |
| JSON Marshaller | The web |
| `GmSerializationOptions` | Options for writing |
| `GmDeserializationOptions` | Options for reading |
| `TypeExplicitness` | How much type information is written |
| `writeAbsenceInformation`, `absentifyMissingProperties` | Keep absence across a round trip |
| `IdentityManagementMode` | How a repeated entity is recognized |
| `stabilizeOrder` | Deterministic output |

## See also

- [cloning-and-traversing.md](cloning-and-traversing.md) — the same walk over a graph, without a stream
- [resource.md](resource.md) — why the bytes of a file do not travel with the entity
- [configuration.md](configuration.md) — YAML marshalling as the basis of modeled configuration
