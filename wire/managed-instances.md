# Managed Instances

**What it is.** A Managed Instance is an instance that Wire keeps for you. `@Managed` marks a method of a Space, and a repeated call then returns the instance the scope already holds.

**When you need it.** You need it for every instance that must be shared between consumers, keyed by a parameter, or closed when the assembly ends.

## @Managed

The annotation sits in two places, and means a different thing in each.

| Where | Meaning |
|---|---|
| On the class | This class is a Space. Wire prepares it, and the annotation value sets the default scope for its methods. |
| On a method | Wire keeps the result of this method. |

Both are needed. A Space class without `@Managed` is not prepared, and the `@Managed` methods inside it have no effect. A method without `@Managed` runs as written, on every call.

```java
@Managed
public class PersistenceSpace implements PersistenceContract {

	@Managed
	@Override
	public PersonStore personStore() {
		PersonStore bean = new PersonStore();
		bean.setDatabase(database());
		return bean;
	}

	@Managed
	private Database database() {
		Database bean = new Database();
		bean.setUrl("jdbc:h2:mem:addressbook");
		return bean;
	}
}
```

`personStore()` and `database()` each return the same instance on every call, so every consumer of `database()` shares one database.

## Scopes

A scope decides how long an instance lives, and how many of them exist.

| Scope | The instance lives |
|---|---|
| `singleton` | Once per Wire Context. This is the default. |
| `prototype` | Not at all. Every call runs the method and returns a new instance. |
| `aggregate` | Once per top instance that started the current build, and is closed together with it. |
| `caller` | In the scope of the instance that asked for it. With no caller it behaves as `singleton`. |
| `inherit` | In the default scope. This is the value `@Managed` uses when you write none. |

Write the scope as the annotation value: `@Managed(Scope.prototype)`.

The default is resolved in two steps. `@Managed` on the class sets the default for that Space. The Wire Context builder sets the default for the whole assembly, and that default is `singleton`.

Use `prototype` for an instance that carries per call state. Use `aggregate` when a group of instances belongs to one owner and must be closed with it.

## Cyclic dependencies

Two Managed Instances may point at each other. Whether that works depends on how you write the method.

Wire finds the local variable that the method returns, and publishes its value to the holder at every assignment. The instance is therefore already available while the rest of the method still runs, and a call back into the first method returns it.

```java
@Managed
private Person alice() {
	Person bean = new Person();   // published here, before the next line runs
	bean.setPartner(bob());
	return bean;
}

@Managed
private Person bob() {
	Person bean = new Person();
	bean.setPartner(alice());     // gets the Person that alice() already published
	return bean;
}
```

This is why a Managed Instance is written in four steps: create, assign to a local variable, configure, return the variable. A method that creates and configures in one expression publishes nothing before it ends, so a cycle through it cannot close.

```java
// A cycle through this method does not terminate. Nothing is published.
@Managed
private Person alice() {
	return new Person().withPartner(bob());
}
```

## Instances with parameters

A `@Managed` method may take parameters. The parameter list decides how many instances exist.

| Parameters | How many instances |
|---|---|
| None | One per scope |
| One `ScopeContext` | One per `ScopeContext` value |
| Anything else | One per distinct argument list, kept in a map keyed by the arguments |

The `ScopeContext` form is how an assembly holds per session or per request state. `WireContext.close(scopeContext)` destroys everything held for that one context and leaves the rest of the Wire Context running.

A parameterized method is easy to misuse. See [patterns.md](patterns.md).

## InstanceConfiguration and the lifecycle

A Managed Instance often owns something that must be released. `InstanceConfiguration.currentInstance()` inside a `@Managed` method gives the configuration of the instance being built.

```java
@Managed
private Database database() {
	Database bean = new Database();
	bean.setUrl("jdbc:h2:mem:addressbook");
	InstanceConfiguration.currentInstance().closeOnDestroy(bean);
	return bean;
}
```

| Method | Purpose |
|---|---|
| `onDestroy(Runnable)` | Runs the given code when the instance is destroyed |
| `closeOnDestroy(AutoCloseable)` | Closes the given object when the instance is destroyed |
| `qualification()` | Which Space, which name, which scope |

Destruction happens when the scope ends: `WireContext.shutdown()` for the default scope, `close(scopeContext)` for one `ScopeContext`, and the owner's destruction for an `aggregate` instance.

## Identifying an instance

Three types name an instance. They appear in error messages and in listeners.

| Type | Answers |
|---|---|
| `InstanceQualification` | Which Space, which method name, which scope |
| `InstanceHolder` | The qualification, plus the instance itself and its configuration |
| `InstancePath` | The chain of instances currently being built, from the first to the one in construction |

`WireContext.currentInstancePath()` gives the path. It is the fastest way to see why an instance is being built at all.

## LifecycleListener and CreationListener

Both are registered on the Wire Context builder, and both see every instance in the assembly.

| Interface | Called |
|---|---|
| `CreationListener` | `onBeforeCreate` and `onAfterCreate`, around the construction of an instance |
| `LifecycleListener` | `onPostConstruct` after an instance is complete, `onPreDestroy` before it is destroyed |

Use them for cross cutting work: timing the start of an assembly, or registering every instance of a kind.

## Under the hood

Wire prepares a Space at runtime, while the Wire Context is built. It loads the class through its own class loader and rewrites each `@Managed` method, so that the method first asks its holder and runs the original body only when no instance exists yet.

The rewriting explains why both `@Managed` annotations are needed. The one on the class tells Wire that this class is worth rewriting.

A Space is only rewritten if its class name is covered by the Wire Context. If a Space with `@Managed` is loaded unchanged, Wire reports that the Space "was not instrumented" and names the package that is not covered. `@Enriched` on a class states that the class is already prepared, so Wire leaves it alone.

## Cheat sheet

| Annotation or type | Purpose |
|---|---|
| `@Managed` on a class | Marks a Space, and sets its default scope |
| `@Managed` on a method | Wire keeps the result |
| `Scope` | `singleton`, `prototype`, `aggregate`, `caller`, `inherit` |
| `InstanceConfiguration.currentInstance()` | The configuration of the instance being built |
| `onDestroy` and `closeOnDestroy` | Work to run when the instance is destroyed |
| `InstanceQualification` | Space, name and scope of an instance |
| `InstanceHolder` | The instance, its qualification and its configuration |
| `InstancePath` | The chain of instances currently being built |
| `CreationListener` | Sees construction start and end |
| `LifecycleListener` | Sees an instance become complete, and its destruction |
| `@Enriched` | The class is already prepared; Wire does not rewrite it |

## See also

- [wire.md](wire.md) — the four elements, and a minimal example
- [contracts-and-spaces.md](contracts-and-spaces.md) — what a Space may expose
- [wire-context.md](wire-context.md) — where scopes end
- [patterns.md](patterns.md) — the mistakes that follow from this page
