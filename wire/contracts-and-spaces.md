# Contracts and Spaces

**What it is.** A Contract is an interface that declares what one part of an application offers. A Space is the class that implements a Contract and builds the instances behind it.

**When you need it.** You write a Space whenever you add a part to an application, and a Contract whenever another part must use it.

## WireSpace

Every Contract and every Space stands on `WireSpace`. A Contract extends it; a Space implements the Contract, and through it `WireSpace`.

`WireSpace` is not an empty marker. It gives two methods, both with a default, so a Space overrides one only when it needs it.

| Method | Purpose |
|---|---|
| `onLoaded(WireContextConfiguration)` | Runs when the Space is loaded into a Wire Context, before any instance is built |
| `reflect()` | Returns a `WireSpaceReflection`, which lists the Managed Instances of the Space |

## A Contract is the public API

A Contract states what a consumer may ask for. It must expose API types only. An implementation type on a Contract makes every consumer depend on a decision that should stay inside the Space.

```java
// Wrong. Every consumer now depends on the storage technology.
public interface PersistenceContract extends WireSpace {
	HibernatePersonStore personStore();
}

// Right. The consumer sees the interface, and the Space chooses the implementation.
public interface PersistenceContract extends WireSpace {
	PersonStore personStore();
}
```

The same rule applies to method names. A Contract method names what the instance is, not how it is built.

A Space may implement more than one Contract. That is the normal way to offer two separate views of one part, for example a public Contract and a Contract meant only for tests.

## Naming convention

Wire finds a Space from its Contract by package and name.

| Role | Package | Name |
|---|---|---|
| Contract | `<base>.contract` | ends with `Contract` |
| Space | `<base>.space` | ends with `Space` |

So `com.example.addressbook.contract.PersistenceContract` resolves to `com.example.addressbook.space.PersistenceSpace`.

Both the two packages and the two suffixes can be set to other values. The convention is a default, not a rule of the library.

## @Import

`@Import` on a field of a Space gives that Space access to another part of the assembly. Wire fills the field before the first instance is built. The field may be private.

```java
package com.example.addressbook.space;

@Managed
public class AddressBookSpace implements AddressBookContract {

	@Import
	private PersistenceContract persistence;

	@Managed
	@Override
	public AddressBook addressBook() {
		AddressBook bean = new AddressBook();
		bean.setStore(persistence.personStore());
		return bean;
	}
}
```

The field type may be either of two things, and the choice matters.

| Field type | Effect |
|---|---|
| A Contract interface | The Space sees only what the Contract declares. The implementing Space can be replaced. |
| Another Space class | The Space sees every public method of that Space, including the ones outside its Contract. |

Import a Contract unless you have a reason not to. A Space that imports another Space is bound to it and cannot be rebound.

## Binding a Contract to a Space

The naming convention is one way to bind. The Wire Context builder offers several, and they can be combined.

| Method | Binds |
|---|---|
| `bindContract(contract, spaceClass)` | One Contract to one Space class |
| `bindContract(contract, spaceInstance)` | One Contract to an instance you already have |
| `bindContract(contract, className)` | One Contract to a Space named by its class name |
| `bindContracts(basePackage)` | Everything under `<basePackage>.contract` and `<basePackage>.space`, by the convention |
| `bindContracts(contractPackage, spacePackage)` | The same, with the two packages given |
| `bindContracts(contractPackage, contractSuffix, spacePackage, spaceSuffix)` | The same, with the two suffixes given as well |
| `bindContracts(spaceClass)` | Like `bindContracts(basePackage)`, with the base package taken from the class |
| `bindContracts(resolver)` | Everything a `ContractSpaceResolver` you write can resolve |

A `ContractSpaceResolver` answers one question: given a Contract, which Space implements it? It returns a `ContractResolution`, or `null` when it does not know the Contract. A `null` lets the next resolver try, so several strategies can live in one Wire Context.

Binding an instance is what makes a Wire Context testable. You bind a Contract to a stub you built by hand, and every consumer of that Contract gets the stub.

## Aggregating Contracts

`@ContractAggregation` marks an interface that extends several Contracts and adds nothing of its own. No Space implements it. Wire implements it for you, and sends each method to the Contract that declares it.

```java
@ContractAggregation
public interface AddressBookAppContract extends AddressBookContract, PersistenceContract {
	// no methods of its own
}
```

Use it to give a consumer, or a Wire Context, one entry point instead of several. Do not declare a method on the aggregating interface: only the methods of the extended Contracts are served.

## Cheat sheet

| Type or annotation | Purpose |
|---|---|
| `WireSpace` | The interface under every Contract and every Space |
| `@Import` | On a field of a Space: Wire fills it with another Contract or Space |
| `@ContractAggregation` | On an interface that extends several Contracts: Wire implements it |
| `WireContextConfiguration` | What `onLoaded` receives |
| `WireSpaceReflection` | The Managed Instances of a Space, from `reflect()` |
| `ContractSpaceResolver` | Answers which Space implements a Contract |
| `ContractResolution` | The answer, or `null` when the resolver does not know the Contract |

## See also

- [wire.md](wire.md) — the four elements, and a minimal example
- [managed-instances.md](managed-instances.md) — what `@Managed` does to a method of a Space
- [wire-context.md](wire-context.md) — how Modules bind Contracts, and how a binding is overridden
