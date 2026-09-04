# Modules

> Status: skeleton. Each section states the question it must answer. Answers are written in Phase 3, from code.

**Scope.** The unit of extension in Reflex: how a Module is declared, when its code runs, and how Modules reach each other.

## RxModule and RxModuleContract

How does a Module name its Contract, and what does the platform do with it?

## The hooks

What belongs in each hook, and in which order are they called?

## The Module artifact

What goes into a `-rx-module` artifact, and what goes into its `-module-api` companion?

## Module discovery and order

How does the platform find the Modules, and what decides their order?

## Exports

How does one Module offer instances to another, and how does the other consume them?

## Platform Contracts

Which Contracts may a Module import, and which page covers each?

## A worked example

What does a complete Module look like, read line by line?

## Cheat sheet

| Hook | Use it for |
|---|---|

## See also

- [../wire/wire-context.md](../wire/wire-context.md)
- [service-processing.md](service-processing.md)
