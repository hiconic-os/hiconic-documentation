# Project layout

> Status: skeleton. Each section states the question it must answer. Answers are written in Phase 3, from code.

**Scope.** Which artifact a class belongs in, and which artifact may depend on which.

## Artifact suffixes

What belongs in each suffix, and what does not?

## Group and package conventions

How does a group map to a package, and why does that matter?

## Dependency directions

Which dependencies must never be created, and what is the reason for each?

## A project, laid out

What does a complete project look like, artifact by artifact?

## Reserved classpath folders

An artifact publishes files on the classpath by declaring them, as described in [../generic-model/classpath-resources.md](../generic-model/classpath-resources.md). The names it declares are otherwise free, but Reflex reserves two of them and reads them itself.

| Folder | Read by |
|---|---|
| `HICONIC-CONF/` | the platform, as the classpath layer of the Application configuration |
| `HICONIC-APP-RESOURCES/` | the assembler, which copies the content into the root of the assembled Application |

Both are top level folders of the artifact, so a file lands at `src/HICONIC-CONF/logback.xml`. Every artifact on the classpath may contribute to either one, and the contributions of all artifacts are taken together.

### HICONIC-CONF

What the Application reads as configuration:

- modeled configuration, one file per configuration type and use case
- `logback.xml`, the logging configuration
- `log-levels.properties`
- `properties.yaml`

Two more files in the same folder are read when the Application is assembled rather than when it runs:

- `jvm.options`, collected into the launch configuration
- `webapp-dependencies.properties`, which names the web applications to package

How the contributions of several artifacts are ordered and merged is described in [configuration.md](configuration.md).

### HICONIC-APP-RESOURCES

Files an artifact contributes to the assembled Application directory instead of to its classpath. The prefix is stripped, so `HICONIC-APP-RESOURCES/local/compose.yaml` becomes `local/compose.yaml` beside the Application.

The assembly fails when two artifacts contribute the same path, and when a contribution targets a path the Application keeps for itself.

## Cheat sheet

| Suffix | Contains |
|---|---|

## See also

- [modules.md](modules.md)
- [../generic-model/models.md](../generic-model/models.md)
- [../generic-model/classpath-resources.md](../generic-model/classpath-resources.md) — how an artifact declares what it publishes
