# Wire

**What it is.** Wire is a dependency injection library for Java in which the wiring itself is Java code. A Contract is an interface, a Space is a class that implements it, and one instance is one method.

**When you need it.** You need Wire as soon as you assemble an application from parts. Every Hiconic application and every Reflex Module is wired.

## The problem Wire solves

An application is a graph of objects. Something must create the objects, connect them, and close them again. There are three usual answers, and each one has a cost.

| Answer | Cost |
|---|---|
| Each component creates what it needs | The components are coupled. You cannot replace one part in a test. |
| An external file, for example XML | The compiler cannot check it. An error appears at start, or later. |
| Annotations on the components, plus a container that scans the classpath | The components depend on the container. The assembly is implicit, so you cannot read it. |

Wire gives a fourth answer. The assembly is Java code, it lives in one place, and the components know nothing about Wire.

## The wiring is Java code

A Space is a normal class. An instance is a normal method. A dependency is a normal method call. Everything else follows from that.

| You get | Because |
|---|---|
| The compiler checks the assembly | A missing or wrong dependency is a compile error, not a start error |
| The IDE navigates and renames | An instance is a method, so "go to definition" and "find usages" work |
| The debugger steps into the assembly | A breakpoint in a method that builds an instance stops there |
| Loops, conditions and generics | The assembly is Java, so there is no separate expression language to learn |
| Components have no dependency on Wire | The component classes carry no Wire annotation, and are constructed by ordinary Java code |

## The four elements

| Element | What it is |
|---|---|
| Contract | An interface that extends `WireSpace` and declares what is available |
| Space | A class that implements a Contract and builds the instances |
| Managed Instance | A method marked `@Managed`, whose result Wire keeps |
| Wire Context | The running assembly, which owns every Managed Instance |

## A minimal example

The Contract declares what a caller may ask for. It extends `WireSpace`.

```java
package com.example.addressbook.contract;

public interface AddressBookContract extends WireSpace {
	AddressBook addressBook();
}
```

The Space implements the Contract. Each method builds one instance, in plain Java.

```java
package com.example.addressbook.space;

@Managed
public class AddressBookSpace implements AddressBookContract {

	@Managed
	@Override
	public AddressBook addressBook() {
		AddressBook bean = new AddressBook();
		bean.setStore(personStore());
		return bean;
	}

	@Managed
	private PersonStore personStore() {
		return new PersonStore();
	}
}
```

The caller builds a Wire Context and asks the Contract for the instance.

```java
WireContext<AddressBookContract> context =
		Wire.contextWithStandardContractBinding(AddressBookContract.class).build();

AddressBook addressBook = context.contract().addressBook();

context.shutdown();
```

Three points from this example:

- `addressBook()` returns the same `AddressBook` on every call, because the method is `@Managed`. Without `@Managed`, the method runs again and returns a new instance each time.
- `personStore()` is private. Only a method on the Contract is reachable from outside, so a Space can build as many internal instances as it needs.
- `AddressBook` and `PersonStore` are plain classes. They carry no annotation and know nothing about Wire.

### How Wire found the Space

`contextWithStandardContractBinding` uses a package and name convention. It takes the package of the Contract, removes the last segment, and binds from there.

| Role | Package | Name |
|---|---|---|
| Contract | `<base>.contract` | ends with `Contract` |
| Space | `<base>.space` | ends with `Space` |

So `com.example.addressbook.contract.AddressBookContract` resolves to `com.example.addressbook.space.AddressBookSpace`. Both the packages and the two suffixes can be set to other values. See [contracts-and-spaces.md](contracts-and-spaces.md).

## Comparison with annotation based injection

Read this if you know Spring or CDI. The elements are similar, but three answers are different.

| Question | Annotation based container | Wire |
|---|---|---|
| Where is the assembly declared? | On the component classes, plus a component scan | In a Space |
| What checks the assembly? | The container, when the application starts | The Java compiler, when you build |
| How is one instance identified? | By type, plus a qualifier when the type is not enough | By the method that returns it |
| Who builds instance X? | Search for annotations | Find usages of the method |
| May the assembly use a loop or a condition? | No | Yes |
| What must the component class know? | The container's annotations | Nothing |

## Under the hood

You do not need this section to use Wire. Read it if you want to know what happens to a Space, or if you meet an error that comes from it.

Wire changes the Space classes **at runtime**, while the Wire Context is built. It is not a build step, and there is no compiler plugin. Wire loads each Space through its own class loader and rewrites the `@Managed` methods, so that a repeated call returns the instance that the scope already holds. The system property `com.braintribe.wire.enrichment` selects how the rewriting is done. It is set to `asm` unless you change it.

A Space is only rewritten if its class name is covered by the Wire Context. If a Space with `@Managed` methods is loaded unchanged, Wire reports that the Space "was not instrumented", and names the package that is not covered.

Wire is not free of Java reflection. It resolves classes by name, it calls constructors through method handles, and it builds a JDK dynamic proxy for an aggregated Contract.

## Cheat sheet

| Element | Purpose |
|---|---|
| `WireSpace` | The interface that every Contract and every Space extends |
| `@Managed` | On a method: Wire keeps the result. On a class: sets the default scope for its methods |
| `@Import` | On a field of a Space: gives the Space access to another Space or Contract |
| `Wire.context(Class)` | Starts a Wire Context builder for a Contract |
| `Wire.contextWithStandardContractBinding(Class)` | The same, with the `contract` and `space` package convention already set |
| `Wire.context(WireTerminalModule)` | Builds a Wire Context from a Module and its dependencies |
| `WireContext.contract()` | Returns the main Contract of the running assembly |
| `WireContext.shutdown()` | Ends the assembly and destroys its Managed Instances |

## See also

- [contracts-and-spaces.md](contracts-and-spaces.md) — how a Contract is bound to a Space
- [managed-instances.md](managed-instances.md) — scopes, lifecycle and cycles
- [wire-context.md](wire-context.md) — Modules, and how a Wire Context is assembled from them
- [patterns.md](patterns.md) — practices and pitfalls
