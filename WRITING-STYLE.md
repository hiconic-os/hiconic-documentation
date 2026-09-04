# Writing style

How every page in this repository is written. Read this before you add or change a page.

The documentation has two readers: a developer building a Hiconic application, and an LLM answering questions about Hiconic. Both are served by the same thing — short, exact, predictable pages.

## Page shape

Every page follows the same order.

1. `# Title`
2. **What it is** — two sentences.
3. **When you need it** — two sentences.
4. Concept sections. One concept per `##` heading. The heading is the term itself, so it can be linked and searched.
5. **Under the hood** — optional. Implementation detail that a reader does not need in order to use the thing correctly.
6. **Cheat sheet** — a table: type, annotation or method, and its purpose.
7. **See also** — links to related pages.

A concept is defined on exactly one page. Every other page links to it.

## Accuracy

**No marketing.** Every sentence must be checkable. Write what a thing does and what follows from it. Do not write that something is powerful, elegant, flexible, clean or seamless, and do not write a benefit that cannot be traced to a mechanism.

Two tests before a sentence stays:

- *Could a reader disagree with it, and be settled by the code?* If not, it says nothing.
- *Does it name the mechanism, or only the effect?* An effect without its cause is a claim, not documentation.

Vague nouns hide missing facts. "Better", "easier", "modern", "enterprise ready", and anything about what a value "means to a user" are not statements about the software.

When you cannot say precisely why something exists, do not fill the gap with prose. Ask, or leave the section marked open.

**Every statement comes from code read while writing, not from memory.**

**Name the evidence to yourself, never to the reader.** Before you write a sentence about a mechanism, find the class that implements it. If you cannot name that class, you have not verified the mechanism, and you must not write the sentence. The class name stays out of the page.

**Existing documentation is a lead, never a source.** Other pages, in this project or outside it, tell you where to look. They do not tell you what is true. Verify against the code before you write it down.

**Check overloaded words against the reader's meaning.** The code base uses some words in a narrower sense than a reader assumes. Known cases:

| Word | What the code base means | What a reader assumes |
|---|---|---|
| compile | byte code generation at runtime | a build step |
| weaving | the same | a build step |
| space | a Wire class | a namespace |
| access | a persistence domain | a permission |

When such a word appears, either avoid it or define it on the spot.

## What a page may reference

**Reference nothing outside this repository.** No source file paths, no implementation class names, no links to other documentation. All of it changes without notice, and none of it helps the reader.

A page names only what a user of Hiconic writes or calls:

- public API types, interfaces and annotations
- methods a user calls
- artifact names
- system properties and configuration keys

A page never names a class that only exists inside the platform.

## Examples

**Write the examples, do not quote them.** An example is written for the documentation and stays as short as the point allows. Every type, annotation and method in it is verified against the code first. An example is never a citation of a test or of an artifact, and it carries no source reference.

**Use one example domain everywhere.** `Person`, `Company`, `Address` and `Gender`. A reader who has seen one example recognizes the next one immediately.

## Naming

**Capitalize a named concept.** Written with a capital, the word is the Hiconic concept. Written in lower case, it is the ordinary English word. This lets a sentence such as "a Space is not a namespace" read correctly. Keep the capital in the middle of a sentence, and in the plural: Contracts, Spaces.

| Layer | Concepts |
|---|---|
| Wire | Contract, Space, Managed Instance, Wire Context, Module |
| Generic Model | Entity Type, Model, Metadata, Session, Manipulation, Access, Service Request, Reason, Marshaller, Resource |
| Reflex | Module, Application, Service Domain, Worker |

Inside a page the short form is enough, because the layer is clear. Where the layer is not clear — in the glossary, in the top level README, and when a page crosses layers — write the qualified form: Wire Contract, Wire Space, Wire Module, Gm Session, Rx Module.

Do not use *contract* in the ordinary sense of "the interface a class implements". Write *interface*. The word Contract belongs to Wire.

## Structure of the prose

**Implementation detail comes last, never first.** A reader opens a page to learn what a thing is and how to use it. How it works inside is a different question. A larger block of implementation detail goes into an **Under the hood** section at the end. A small point that makes a concept easier to understand may stay inline, as one or two sentences where it helps. Never open a page, and never open a concept section, with implementation detail.

This settles the tension with the accuracy rule above. A property that the reader needs early does not have to carry its mechanism in the same sentence — the mechanism may stand in **Under the hood**, where the reader has the context for it. What the accuracy rule forbids is a property whose cause appears nowhere on the page. A `because` clause that names a mechanism the reader has not met yet explains nothing.

**Introduce a thing by what it is, not by one use of it.** "An Access needs a Model" tells a reader what one caller wants, names something the page has not introduced, and hides that there are other reasons. Write what the thing is and what it produces, then give the uses.

**Do not reference a concept before the page introduces it.** Either introduce it in the sentence, or link to the page that defines it, or reorder the sections.

**Prefer a table over prose** when the content is a list of items with one property each.

**Enumerated cases become a list, not a sentence.** When a statement covers two or more cases, write one sentence that gives the context, then the cases as a vertical list. A reader finds a case in a list at a glance, while a sentence that carries three cases has to be read to the end first. The same holds in Javadoc, where the list is a `<ul>`.

Not this:

> The configuration is checked at startup: a missing name, a duplicate id, or a property that names an unknown type stops the application.

This:

> The configuration is checked at startup. The application stops when it finds one of these:
> - a missing name
> - a duplicate id
> - a property that names an unknown type

**Say it once.** If two pages need the same explanation, one page owns it and the other links.

## Language

Short sentences, one idea each. Active voice. Simple present.

Use the same word for the same thing every time. Do not vary the wording for style.

Explain a term the first time it appears on a page, or link to the page that defines it.

## Markdown

One line per paragraph and per bullet. Never hard wrap a paragraph.

Link between pages with a relative path. Within a folder use the file name; across folders use `../<folder>/<file>.md`. The link text is the file name.

Code identifiers, annotations, artifact names, paths and configuration keys go in backticks.
