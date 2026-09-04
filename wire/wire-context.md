# Wire Context and Modules

**What it is.** A Module declares which Contracts belong to an assembly, and which other Modules it needs. A Wire Context is the running assembly, and it owns every Managed Instance in it.

**When you need it.** You need a Module to make a set of Spaces reusable, and a Wire Context to start the assembly and to stop it again.

## WireModule

A Module is an interface with two default methods and nothing you must implement.

| Method | Default |
|---|---|
| `dependencies()` | An empty list |
| `configureContext(WireContextBuilder)` | Binds the Contracts of the Module's own package |

The default binding is the reason a Module is usually this short:

```java
public class PersistenceModule implements WireModule {
	// nothing to declare
}
```

The Module class sits in the base package, so `configureContext` binds `<base>.contract` to `<base>.space` by the naming convention. See [contracts-and-spaces.md](contracts-and-spaces.md).

Override `dependencies()` to name the Modules this one needs.

```java
public class AddressBookModule implements WireModule {
	@Override
	public List<WireModule> dependencies() {
		return list(new PersistenceModule());
	}
}
```

## WireTerminalModule

A terminal Module is the one that starts an assembly. It names the main Contract of the Wire Context as its type argument.

```java
public class AddressBookAppModule implements WireTerminalModule<AddressBookContract> {
	@Override
	public List<WireModule> dependencies() {
		return list(new PersistenceModule());
	}
}
```

`contract()` reads the main Contract from the type argument, so you do not write it twice. A Module that is only a building block stays a `WireModule`; only the entry point of an assembly is terminal.

## Building a Wire Context

```java
WireContext<AddressBookContract> context = Wire.context(new AddressBookAppModule());
```

| Call | Use it for |
|---|---|
| `Wire.context(terminalModule)` | The normal case: build the Wire Context of a Module and its dependencies |
| `Wire.context(terminalModule, otherModules...)` | The same, plus Modules that no dependency names |
| `Wire.contextBuilder(terminalModule)` | The same, but keep the builder open for more configuration |
| `Wire.context(contractClass)` | A builder for one Contract, with no Module and no binding yet |
| `Wire.contextWithStandardContractBinding(contractClass)` | The same, with the package convention already applied |

`Wire.context(Class)` accepts an interface only. A Space class is refused, because the top of a Wire Context is a Contract.

## Module order and overriding a Contract

Wire collects the Modules depth first: every dependency of a Module comes before the Module itself, and each Module appears once. It then calls `configureContext` on each, in that order.

That order is what makes overriding work. A Module configures **after** the Modules it depends on, so its binding replaces theirs. An explicit binding of a Contract wins over an earlier binding of the same Contract.

```java
public class TestModule implements WireTerminalModule<AddressBookContract> {

	@Override
	public List<WireModule> dependencies() {
		return list(new AddressBookAppModule());
	}

	@Override
	public void configureContext(WireContextBuilder<?> builder) {
		WireTerminalModule.super.configureContext(builder);
		builder.bindContract(PersistenceContract.class, new InMemoryPersistenceSpace());
	}
}
```

Call the `super` method first when you override `configureContext`. Without it the Module loses its own package binding.

## WireContext

The Wire Context is the running assembly.

| Method | Purpose |
|---|---|
| `contract()` | The main Contract, typed |
| `contract(Class)` | Any other Contract in the assembly |
| `findContract(Class)` | The same, but `null` instead of an error when it is not bound |
| `findModuleFor(Class)` | Which Module brought a given Contract |
| `currentInstancePath()` | The chain of instances currently being built |
| `shutdown()` | Ends the assembly and destroys its Managed Instances |
| `close(ScopeContext)` | Destroys only what is held for one `ScopeContext` |

`WireContext` extends `AutoCloseable`, and `close()` calls `shutdown()`. So a short lived assembly fits a try with resources block.

```java
try (WireContext<AddressBookContract> context = Wire.context(new AddressBookAppModule())) {
	context.contract().addressBook().add(person);
}
```

Shut a Wire Context down exactly once, and do not reach a Contract afterwards. See [patterns.md](patterns.md).

## WireContextConfiguration

A Space that overrides `onLoaded(WireContextConfiguration)` receives the configuration of the Wire Context it was loaded into. Use it when a Space must know something about the assembly around it, rather than about its own instances.

## Cheat sheet

| Type or method | Purpose |
|---|---|
| `WireModule` | A set of Contract bindings, plus the Modules it depends on |
| `WireTerminalModule<C>` | The Module that starts an assembly, with `C` as its main Contract |
| `dependencies()` | The Modules this one needs |
| `configureContext(builder)` | Where a Module binds Contracts; call `super` first when overriding |
| `Wire.context(module)` | Builds the Wire Context |
| `WireContext.contract()` | The main Contract |
| `WireContext.shutdown()` | Ends the assembly |
| `WireContext.close(scopeContext)` | Ends one `ScopeContext` |

## See also

- [contracts-and-spaces.md](contracts-and-spaces.md) — the binding methods a Module calls
- [managed-instances.md](managed-instances.md) — what `shutdown()` destroys
- [../reflex/modules.md](../reflex/modules.md) — how Reflex builds on terminal Modules
