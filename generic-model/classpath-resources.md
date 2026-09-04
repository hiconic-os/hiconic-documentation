# Classpath resources

**What it is.** A Classpath Resource is a file on classpath an artifact explicitly declares as a resource, by referencing it, or on of its parent dirs, in its `META-INF/classpath-resources.txt`. There resources are indexed so they can be enumerated, with the information which artifact they came form.

**When you need it.** You need it when code looks for files it cannot name in advance, such as all the configuration files, or when it has to know which artifact a file came from. It also blocks resources on the classpath that would match a given criteria by accident, e.g. those in third party libs.

## Why declare a resource

`ClassLoader.getResource` reaches any file on the classpath, and keeps working. It answers one question: give me the file at this exact name.

A declaration adds three things it cannot give.

- **Intent.** The declaration separates the files an artifact publishes as resources from everything else it happens to ship.
- **Enumeration.** Declared files can be listed, all of them or by prefix, without knowing a single name.
- **Origin.** Every listed file names the artifact it came from.

## Reading them

`ClasspathIndex` is the index over the declared resources of the whole classpath.

```java
ClasspathIndex index = new ClasspathIndex();

for (ClasspathEntry entry : index.forPrefix("CONF/"))
	read(entry.url, entry.origin);
```

| Member | What it gives |
|---|---|
| `all()` | every declared resource on the classpath, ordered by path |
| `forPrefix(prefix)` | those whose path starts with the prefix |
| `entry.path` | the path as declared, relative to the artifact |
| `entry.url` | where the file is now: inside a jar, in an output folder, or in an assembled application |
| `entry.origin` | the artifact the file came from |

Two artifacts may declare the same path, and then both entries are in the list. `origin` is what tells them apart, so a component can treat the contributions of several artifacts as a set rather than as one file.

## The declaration

An artifact declares its Classpath Resources in one authored file, kept in the source tree beside the resources it names.

```text
src/META-INF/classpath-resources.txt
```

Each line is a file or a folder, relative to the artifact root, and a folder contributes every file below it. Blank lines and lines starting with `#` are ignored. A path uses `/` and may not leave the artifact.

```text
# address-book-configuration
CONF
logo.svg
```

Declare a file only when something has to find it without knowing its name. A file that one caller loads by a fixed name is found directly, so declaring it adds nothing. The one exception is a Pure Classpath Resource Artifact, which declares everything it ships.

## How a classpath root is read

A classpath root is a jar, or a folder that holds compiled output. What is read depends on which of the two files it carries.

| The root carries | What is read |
|---|---|
| `META-INF/classpath-index.txt` | the index — a jar, or a folder produced by the build |
| only `META-INF/classpath-resources.txt` | the declaration, expanded against the folder — a folder produced by an IDE |
| neither | nothing |

`META-INF/classpath-index.txt` is the declaration expanded into one resource path per line. The build generates it, and it is what a packaged artifact carries.

An IDE copies the declaration into its output folder like any other resource, and a folder can be listed. An IDE launch therefore needs no build step, no builder and no plugin to see the Classpath Resources of a project.

Both views expand the same declaration, so an application sees the same set of resources from a workspace and from a jar.

## The build

`common-ant-script` generates the index, and every `library-ant-script` build runs it. An artifact without a declaration is unaffected, so any artifact may publish Classpath Resources and no separate artifact type is needed.

The build fails when it finds one of these:

- a declared entry that does not exist
- a `.java` file below a declared folder, because it would be compiled into the output folder and indexed there, but never packaged

## Pure Classpath Resource Artifact

A Pure Classpath Resource Artifact holds nothing in its sources besides the declaration and the entries the declaration names. It has no classes, because none may be declared. The build detects this and adds a marker.

```text
META-INF/classpath-resource-only
```

The marker is a packaging signal. It lets an application image drop the jar from its library folder once the resources have been copied out of it.

## Under the hood

An assembled application does not read Classpath Resources from the classpath. The assembler copies them into one folder, with a subfolder per artifact, and writes `index.properties` beside them. The application reads that instead.

Every resource keeps its artifact-relative path and the id of the artifact it came from, so `path` and `origin` are the same as on a classpath and only `url` differs.

### index.properties

A UTF-8 properties file at the root of that folder.

```properties
formatVersion=1
artifact.count=2
artifact.0.folder=address-book-configuration-1.0
artifact.0.origin=address-book-configuration
artifact.0.sourceName=address-book-configuration-1.0.jar
artifact.0.resource.count=2
artifact.0.resource.0.path=CONF/address-book.yaml
artifact.0.resource.1.path=logo.svg
artifact.1.folder=...
```

| Key | Meaning |
|---|---|
| `formatVersion` | Always `1`. Any other value is rejected. |
| `artifact.count` | Number of artifact blocks |
| `artifact.<a>.folder` | The subfolder holding this artifact's resources, relative to the index |
| `artifact.<a>.origin` | The artifact id every resource of the block reports as its origin |
| `artifact.<a>.sourceName` | The jar file name, kept for diagnostics and otherwise unused |
| `artifact.<a>.resource.count` | Number of resources in the block |
| `artifact.<a>.resource.<r>.path` | A resource path, relative to the artifact folder |

`<a>` and `<r>` start at `0` and leave no gaps. Every key except `sourceName` is mandatory. A folder or a path that leaves its parent, or that names something missing, is rejected. Values escape `\`, newline and `=`.

Blocks are written in `folder` order, so the same input always gives the same file.

## Cheat sheet

| Element | Purpose |
|---|---|
| `new ClasspathIndex()` | The index over the declared resources of the current classpath |
| `all()`, `forPrefix(prefix)` | List them, all or by prefix |
| `ClasspathEntry` | One resource: `path`, `url` and `origin` |
| `META-INF/classpath-resources.txt` | Declares what the artifact publishes; authored |
| `META-INF/classpath-index.txt` | The declaration expanded; generated by the build |
| `META-INF/classpath-resource-only` | The artifact ships resources and nothing else |
| `index.properties` | The index of an assembled application's resource folder |
| `common-ant-script` | Generates the index and the marker |

## See also

- [configuration.md](configuration.md) — modeled configuration, read from Classpath Resources
- [../reflex/configuration.md](../reflex/configuration.md) — how an application assembles its configuration
