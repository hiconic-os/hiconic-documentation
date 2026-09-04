# Hiconic documentation

This repository holds the documentation for Hiconic. It contains no source code.

Start at [README.md](README.md). It names the three layers, the reading order, and the page that answers each task.

## Before you write

Read [WRITING-STYLE.md](WRITING-STYLE.md) first. It is binding for every page. The two rules that are broken most often:

- A page references nothing outside this repository. No source paths, no implementation class names, no links to other documentation.
- Every statement comes from code read while writing. Find the class that implements a mechanism before you describe it, and keep that class name out of the page.

[GLOSSARY.md](GLOSSARY.md) is the term list. A concept is defined on exactly one page, and the glossary points at it.

## Layout

| Folder | Layer |
|---|---|
| [wire/](wire/) | Wire — dependency injection in which the wiring is Java code |
| [generic-model/](generic-model/) | Generic Model — a type system for entities, and the APIs on it |
| [reflex/](reflex/) | Reflex — a modular application platform on Wire and Generic Model |

## Work in progress

[OUTLINE.md](OUTLINE.md) is the working plan for the first complete version: the page list, the order of work, and the findings collected so far. It is deleted when the documentation is complete. [WRITING-STYLE.md](WRITING-STYLE.md) stays.
