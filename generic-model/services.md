# Services

**What it is.** A Service Request is an entity that describes a call. An Evaluator evaluates it, and a processor registered for its type handles it.

**When you need it.** Every callable operation in Hiconic is a Service Request, so you need this for anything an application offers.

## A Service Request is data

A Service Request is an Entity Type like any other. The request is data, and so is the response.

```java
public interface FindPersons extends ServiceRequest {

	EntityType<FindPersons> T = EntityTypes.T(FindPersons.class);

	String namePattern = "namePattern";

	@Mandatory
	String getNamePattern();
	void setNamePattern(String namePattern);

	@Override
	EvalContext<List<Person>> eval(Evaluator<ServiceRequest> evaluator);
}
```

The overridden `eval` is what makes the response type visible. Without it the request still works, but the caller gets an untyped result.

Everything follows from the request being data:

| Because a request is data | You get |
|---|---|
| It can be marshalled | The same request works locally, over HTTP and over a message queue |
| It can be stored | A call can be queued, retried, or kept as an audit record |
| It has a Model | A REST layer and an OpenAPI document can be generated from it |
| It carries Metadata | Roles, descriptions and validation are declared on the request itself |

## Evaluating a Service Request

```java
FindPersons request = FindPersons.T.create();
request.setNamePattern("A*");

List<Person> people = request.eval(evaluator).get();
```

The Evaluator decides where the request runs. The same line evaluates locally or against a remote server, depending only on which Evaluator you hold. A `PersistenceGmSession` is one, for the Access it belongs to.

Two ways to take the result, and the choice matters:

| Call | Returns | On failure |
|---|---|---|
| `get()` | The response | Throws |
| `getReasoned()` | `Maybe<R>` | Returns the Reason |

Use `getReasoned()` whenever the failure is expected and you can act on it. See [reasons.md](reasons.md).

`eval` returns an `EvalContext`, so you can add context before asking for the result, and you can ask asynchronously with a callback.

## The request hierarchy

A request declares what it needs by extending the right type.

| Type | Says |
|---|---|
| `ServiceRequest` | The base. Everything else extends it. |
| `AuthorizedRequest` | The caller must be authenticated |
| `DomainRequest` | The request is evaluated in a named Service Domain |
| `DispatchableRequest` | The request carries the id of the component that should handle it |
| `AccessRequest` | The request runs against an Access, and the processor gets a Session for it |

`ServiceRequest` also answers a few questions the platform asks, and a request can override them: `system()`, `requiresAuthentication()`, `interceptable()`, `domainId()`.

## Processors

A processor is plain Java. It takes a context and the request, and returns the response.

```java
public class FindPersonsProcessor implements ReasonedServiceProcessor<FindPersons, List<Person>> {

	@Override
	public Maybe<List<Person>> processReasoned(ServiceRequestContext context, FindPersons request) {
		...
	}
}
```

| Interface | Use it when |
|---|---|
| `ServiceProcessor<P, R>` | The operation either succeeds or throws |
| `ReasonedServiceProcessor<P, R>` | Failure is expected and modeled. Prefer this one. |
| `AccessRequestProcessor<P, R>` | The operation works on an Access, and wants a Session |

`ReasonedServiceProcessor` also implements `process`, by throwing when the `Maybe` is unsatisfied. So a reasoned processor works everywhere a plain one does.

`AccessRequestProcessor` takes a single `AccessRequestContext`, which carries the request and the Session together. That is what an operation over stored data normally wants.

## ServiceRequestContext

The context is the processor's connection to everything around the call.

| Method | Gives |
|---|---|
| `eval(...)` | The Evaluator, so a processor can call other requests |
| `getRequestorUserName()`, `getRequestorSessionId()`, `isAuthorized()` | Who is calling |
| `getDomainId()`, `getRequestedEndpoint()` | Where the call arrived |
| `findAspect(...)` | Anything else the platform put there |
| `derive()` | A modified context for a nested call |

The context is itself an Evaluator. A processor that needs another operation evaluates a request through the context, and the call keeps the caller's identity and context.

## Dispatching

One processor can serve a family of requests. Extend `AbstractDispatchingServiceProcessor` and register one method per request type.

```java
public class AddressBookProcessor extends AbstractDispatchingServiceProcessor<AddressBookRequest, Object> {

	@Override
	protected void configureDispatching(DispatchConfiguration<AddressBookRequest, Object> dispatching) {
		dispatching.register(FindPersons.T, this::findPersons);
		dispatching.register(AddPerson.T, this::addPerson);
	}

	private List<Person> findPersons(ServiceRequestContext context, FindPersons request) { ... }

	private Person addPerson(ServiceRequestContext context, AddPerson request) { ... }
}
```

The dispatch is by entity type and respects the type hierarchy: a registration for a supertype catches every subtype that has none of its own.

For requests against an Access there is the same pattern with `AccessRequestProcessor`, plus `registerStateful(...)` for a handler that needs per call state. A stateful handler is created fresh for each request, so it may keep fields. A plain processor is shared, and must not.

## Interceptors

An interceptor runs around a request without being its processor.

| Kind | Signature | Can |
|---|---|---|
| Pre | `process(context, request)` returns the request | Change or replace the request before it is handled |
| Post | `process(context, response)` returns the response | Change the response after it is handled |
| Around | `process(context, request, proceedContext)` | Both, and decide whether to proceed at all |

An around processor receives a `ProceedContext` and calls `proceed(request)` or `proceedReasoned(request)` to continue the chain. Not calling it is how a request is answered from a cache, or refused.

Interceptors are where cross cutting work belongs: authorization, logging, validation, caching, auditing. A request that must not be intercepted says so with `interceptable()`.

## Service Domains

A request is evaluated in a Service Domain, which decides which Models are visible and which processors are registered. `domainId()` on the request names it, and `DomainRequest` makes it an explicit property.

Domains are configured by the platform, not by the Model. See [../reflex/service-processing.md](../reflex/service-processing.md).

## Cheat sheet

| Element | Purpose |
|---|---|
| `ServiceRequest` | The base of every request |
| The `eval` override | Declares the response type |
| `request.eval(evaluator).get()` | Evaluate, and throw on failure |
| `.getReasoned()` | Evaluate, and get a `Maybe` |
| `AuthorizedRequest`, `DomainRequest`, `DispatchableRequest`, `AccessRequest` | Declare what a request needs |
| `ServiceProcessor` | A handler that throws on failure |
| `ReasonedServiceProcessor` | A handler that returns a `Maybe` |
| `AccessRequestProcessor` | A handler that gets a Session |
| `AbstractDispatchingServiceProcessor` | One processor for a family of requests |
| `ServiceRequestContext` | The caller, the domain, and an Evaluator for nested calls |
| `ServicePreProcessor`, `ServicePostProcessor`, `ServiceAroundProcessor` | Interception |
| `ProceedContext.proceed(...)` | Continue the chain from an around processor |

## See also

- [reasons.md](reasons.md) — what a processor returns when it fails
- [sessions.md](sessions.md) — the Session an Access request works with
- [../reflex/service-processing.md](../reflex/service-processing.md) — where domains and processors are configured
- [../reflex/rest-api.md](../reflex/rest-api.md) — how a request becomes an HTTP endpoint
