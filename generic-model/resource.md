# Resource

**What it is.** A Resource is the Generic Model type for unstructured data: a file, an image, a document. The entity describes the data, and a Resource Source says where the bytes are.

**When you need it.** You need it for every upload, download, generated document and attached file. A property of type `Resource` is how a Model holds a file.

## What a Resource is

A file has two halves, and Generic Model keeps them apart.

| Half | Is | Behaves like |
|---|---|---|
| The `Resource` entity | Name, size, media type, checksum, who created it and when | Any other entity: queried, stored, marshalled, referenced |
| The payload | The bytes | A stream, fetched when somebody asks for it |

That split is why a query for a thousand documents does not move a gigabyte. The entities travel; the bytes stay until somebody opens a stream.

## The properties of a Resource

| Property | Holds |
|---|---|
| `name` | The file name |
| `mimeType` | The media type |
| `fileSize` | The size in bytes |
| `md5` | A checksum, when one was computed |
| `created`, `creator` | When it was stored, and by whom |
| `tags` | Free labels, for grouping |
| `specification` | A description of the content, see below |
| `resourceSource` | Where the payload is |

All of them may be absent. A Resource built from a stream knows nothing about itself until something fills it in.

## Resource Source

`ResourceSource` is an abstract entity, and each kind says where the payload is and how to reach it. Keeping it an entity rather than a URL string means the platform can carry a source it does not itself understand, and hand it to whoever does.

| Source | The payload is |
|---|---|
| `FileSystemSource` | A file on disk |
| `SqlSource` | A row in a database |
| `ClassPathSource` | A file inside an artifact |
| `TransientSource` | Behind a stream provider, in this process only |
| `StringSource` | The text of the source entity itself |
| `UploadSource`, `FileUploadSource`, `UrlUploadSource` | Being uploaded right now |
| `TemplateSource`, `ConversionServiceSource`, `PackagedSource`, `BlobSource`, `StaticSource` | Produced or held by a specific mechanism |

Two methods on the Resource answer what a caller usually needs to know:

- `isTransient()` — the payload exists in this process only, and disappears with it.
- `isStreamable()` — the bytes can be read without a Session.

## Creating a transient Resource

A transient Resource is the normal way to hand bytes to something: a service that takes a file, a Marshaller, an upload.

```java
Resource fromFile = Resources.createTransient(new File("addressbook.csv"));
Resource fromText = Resources.createTransient("name;city\nAlice;Vienna");
Resource fromStream = Resources.createTransient(() -> new ByteArrayInputStream(bytes));
```

It is not stored anywhere. It carries a stream provider, and it lives as long as the process does.

## Reading the payload

```java
try (InputStream in = resource.openStream()) {
	// read the bytes
}

resource.writeToStream(outputStream);
```

Both work in two ways, and the difference matters when they fail.

- If the source is streamable, the bytes come straight from it. No Session is needed.
- Otherwise the Resource must be attached to a Session that supports Resource access. Without one, the call fails and says so.

So a Resource that arrived detached from its Session cannot be opened. Read the payload while you still have the Session, or keep the Resource attached.

## The Resource API of a Session

A Session offers a builder per operation.

```java
Resource stored = session.resources().create()
		.name("addressbook.csv")
		.mimeType("text/csv")
		.store(inputStream);

try (InputStream in = session.resources().retrieve(stored).stream()) {
	// read it back
}

session.resources().delete(stored).delete();
```

| Builder | Purpose |
|---|---|
| `create()` | Store bytes and get a Resource back. Set `name`, `mimeType`, `tags`, `creator`, `specification`, `useCase` before `store(...)`. |
| `retrieve(resource)` | Read the payload. Supports a byte range and a condition. |
| `update(resource)` | Replace the payload of an existing Resource |
| `delete(resource)` | Remove it |
| `url(resource)` | Build a URL a client can fetch directly |

`retrieve(...).range(...)` is what serves a video or a resumed download, and `condition(...)` is what lets a client skip a fetch when nothing changed.

## Resource Specification

A `ResourceSpecification` describes the content rather than the file.

| Specification | Says |
|---|---|
| `RasterImageSpecification`, `VectorImageSpecification` | It is an image, of that kind |
| `PixelDimensionSpecification`, `PhysicalDimensionSpecification` | Its size, in pixels or in physical units |
| `PdfSpecification`, `PageCountSpecification` | It is a PDF, and how many pages |
| `OcrSpecification` | Text was recognized in it |

A specification is normally filled by whoever stored or analysed the Resource, not by hand. It is what lets a client show a thumbnail at the right ratio without opening the file.

## The Resource Service Requests

`resource-api-model` declares the same operations as Service Requests, so they can cross a process boundary. Use them when you are not on the side that holds the Session.

| Group | Requests |
|---|---|
| Read | `GetResource`, `StreamResource`, `DownloadResource`, and the `Binary` and `Source` variants |
| Write | `UploadResource`, `UploadResources`, `UpdateResource`, `StoreBinary` |
| Change part of a payload | `PutBinary`, `AppendBinary`, `InsertBinary`, `DeleteBinaryRange` |
| Delete | `DeleteResource`, `DeleteSource`, `DeleteBinary` |
| Enrich | `EnrichResource`, which fills in what a Resource does not know about itself |

Prefer the Session API when you have a Session. Use the requests when the caller is remote, or when the operation must be part of a request chain.

## A Resource in a Model

A property of type `Resource` is declared like any other.

```java
public interface Person extends GenericEntity {

	EntityType<Person> T = EntityTypes.T(Person.class);

	Resource getPhoto();
	void setPhoto(Resource photo);
}
```

The Access stores the `Resource` entity as it stores any other entity. Where the payload goes is a separate decision, made by the platform rather than by the Model. See [../reflex/resources.md](../reflex/resources.md).

## Marshalling a Resource

A marshalled graph carries the Resource entity, not its bytes. How the payload travels with it depends on the format and the transport. See [marshalling.md](marshalling.md).

## Cheat sheet

| Element | Purpose |
|---|---|
| `Resource` | The entity that describes a file |
| `ResourceSource` | Where the payload is |
| `resource.isTransient()`, `isStreamable()` | Whether it lives only in this process, and whether it can be read without a Session |
| `Resources.createTransient(...)` | A Resource over a file, a text or a stream provider |
| `resource.openStream()`, `writeToStream(...)` | Read the payload |
| `session.resources().create()` | Store bytes and get a Resource |
| `session.resources().retrieve(r)` | Read a payload, with range and condition |
| `session.resources().url(r)` | A URL for a client |
| `ResourceSpecification` | What the content is |
| `resource-api-model` | The same operations as Service Requests |

## See also

- [sessions.md](sessions.md) — the Session a Resource needs to open its payload
- [marshalling.md](marshalling.md) — a Resource in a marshalled graph
- [../reflex/resources.md](../reflex/resources.md) — where the payload is actually stored
