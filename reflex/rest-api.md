# REST

> Status: skeleton. Each section states the question it must answer. Answers are written in Phase 3, from code.

**Scope.** The two REST servers of Reflex, and how to choose between them.

## Which server to use

What is the difference between the two servers, and which one fits which task?

## Service API: a Service Request as an endpoint

How does a Service Request become an HTTP endpoint?

## Mapping a Service Request to a path

How is a mapping declared, and what does the mapping Metadata control?

## Endpoint parameters

How do query parameters and headers reach the Service Request?

## Request and response bodies

How are bodies marshalled, how is the format negotiated, and how does a Resource property travel?

## CRUD API: entities as resources

Which operations does the generic CRUD API offer over an Access?

## Reasons and HTTP status

How does an unsatisfied Reason become a status code and a body?

## OpenAPI

How is the API contract generated, and which Metadata feeds it?

## Worked example

Which demo endpoints exist, and what do the `curl` calls and their output look like?

## Cheat sheet

| Module or type | Purpose |
|---|---|

## See also

- [web-api.md](web-api.md)
- [../generic-model/reasons.md](../generic-model/reasons.md)
