# Resource storage

> Status: skeleton. Each section states the question it must answer. Answers are written in Phase 3, from code.

**Scope.** Where a Resource payload is stored, and how a storage is configured and written. The Resource type itself is described in [../generic-model/resource.md](../generic-model/resource.md).

## The ResourceStorage interface

What must a storage implement, and what does the base class already do?

## The payload API

Which Service Requests exist for store, get, download, pipe and delete, and what does each return?

## Configuring a storage

What does the configuration model declare, and how is a storage selected?

## File system storage

What does the file system storage do, and where does it put its files?

## Database storage

What does the JDBC storage do, and when is it the better choice?

## Packaged Resources

What is a packaged Resource, and how does a Module reach one?

## Writing your own storage

Which steps are needed, and which shared test proves the result is correct?

## Cheat sheet

| Type or Module | Purpose |
|---|---|

## See also

- [../generic-model/resource.md](../generic-model/resource.md)
- [web-api.md](web-api.md)
