# Web layer

> Status: skeleton. Each section states the question it must answer. Answers are written in Phase 3, from code.

**Scope.** Everything below REST: the HTTP server, calls between processes, websockets and streaming. REST itself is in `rest-api.md`.

## The web server

How is the server configured, and how does a Module add a servlet?

## RPC

How is a Service Request evaluated in another process, with the same request types?

## Calling a remote web API

How is a remote API used as if it were local?

## Websockets

What does the websocket server offer, and when is it the right choice?

## Streaming

How are binary payloads served, and how does that relate to a Resource?

## Cheat sheet

| Module | Purpose |
|---|---|

## See also

- [rest-api.md](rest-api.md)
- [resources.md](resources.md)
