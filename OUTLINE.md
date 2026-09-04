# Hiconic documentation — outline (working document)

This file is the plan for the documentation. It is not part of the published documentation. Delete it when the documentation is complete.

## Goal

A concise, information dense reference for the open source project Hiconic. Two audiences: a developer as well as LLM must build a Hiconic application, and quickly make sense of key concepts and how to use them.

Three layers, bottom up:

1. **Wire** (`com.braintribe.wire`) — dependency injection in which the wiring is ordinary Java code, so the compiler checks it. No dependency on the other two.
2. **Generic Model** (`com.braintribe.gm`) — a type system for entities, plus the APIs built on it: model reflection, metadata, sessions, access, services, reasons, value descriptors, marshalling. Also some primitives useful for app development such as distributed locking or messaging. Special type is Resource, to support binary data.
3. **Reflex** (`hiconic.platform.reflex`) — a modular application platform built on Wire and Generic Model.

## Scope decisions

- Examples come **only** from open source artifacts: `com.braintribe.wire`, `com.braintribe.gm`, `hiconic.platform.reflex`. See examples anywhere in the code. Tests related to JDBC are `initializer-manager-jdbc-test` or `jdbc-messaging-test`.
- No customer code and no `dt1.proventem` code appears in any example.
- Generic Model is documented at **application author** depth. Platform internals stay out: query planner and evaluator, instant type weaving internals, traversing internals, codec and marshaller internals, GWT support.
- Existing documentation stays where it is. We copy the content here and rewrite it. We do not modify or delete the originals.

## Source material to reuse

| Source | Lines | Target |
|---|---|---|
| `com.braintribe.wire/wire-doc/src/wire.md` | 70 | `wire/wire.md` |
| `com.braintribe.wire/wire-doc/src/wire_in_detail.md` | 450 | `wire/contracts-and-spaces.md`, `wire/managed-instances.md`, `wire/wire-context.md` |
| `com.braintribe.wire/wire-doc/src/property_lookups.md` | 131 | `wire/patterns.md` |
| `com.braintribe.gm/error-handling-doc/src/error-handling.md`, `reason.md` | 181 | `generic-model/reasons.md` |
| `hiconic.platform.reflex/docs/design-concepts/`, `platform-components/`, `applications/` | 90 | `reflex/reflex.md`, `reflex/application.md` |
| `hiconic.platform.reflex/docs/modules/modules.md` | 136 | `reflex/modules.md` |
| `hiconic.platform.reflex/docs/configuration/configuration-assembly.md` | 382 | `reflex/configuration.md` |
| `hiconic.platform.reflex/docs/modeling/modeling.md` | 138 | `generic-model/models.md`, `reflex/project-layout.md` |
| `hiconic.platform.reflex/docs/getting-started/getting-started.md` | 211 | `reflex/getting-started.md` |
| `hiconic.platform.reflex/docs/notes/*` | 916 | Raw notes. Mine for facts, do not copy the text. |
| `dt1.proventem/docs/architecture.md`, section "Core platform concepts" | 20 | `README.md` mental model |

## Page conventions

The permanent rules live in [WRITING-STYLE.md](WRITING-STYLE.md). Read that file before writing any page. It is not deleted when this outline is.

Rules that belong to this piece of work only, and disappear with it:

- **A skeleton states questions, not answers.** In Phase 2 a section carries the question it must answer. A mechanism claim first appears in Phase 3, after the implementing class is read.
- **Findings go in this file.** A verified finding that names an implementation class is recorded under the page it belongs to, below. It never appears in the page itself. See [WRITING-STYLE.md](WRITING-STYLE.md), "Name the evidence to yourself, never to the reader".

## Page plan

### Top level

**`README.md`** — the entry point.
- The three layers in one diagram and one paragraph each.
- Reading order for a new developer, and a short "I want to ..." task index.
- Group to concept map: which artifact group holds what.

**`GLOSSARY.md`** — one line per term, alphabetical.
- Access, Cmd Resolver, Denotation type, Entity type, Evaluator, Gm Session, ITW, Managed instance, Manipulation, Maybe, Meta Data, Model, Model declaration, Model Oracle, Module, Reason, Service domain, Service request, Smood, Wire context, Wire Contract, Wire Space

Add more terms as you encounter them.

### wire/ — complete

**`wire.md`** — written.
- Why the wiring is Java code, and what that buys: the compiler checks the assembly, the IDE navigates and renames it, the debugger steps into it, and loops and conditions are allowed.
- The four elements: contract, space, managed instance, wire context.
- A complete minimal example, end to end.
- Comparison table with annotation based injection, to set expectations.
- Under the hood, at the end of the page. Correct the record on two points that earlier documentation states wrongly. Wire is not a build step: `WireManagedSpaceFactory` rewrites the space classes at runtime, when the context is built, through `WireEnricherClassLoader`. And Wire is not reflection free: it uses `Class.forName`, `MethodHandles`, and a JDK dynamic proxy in `ContractAggregationInvocationHandler`.

**`contracts-and-spaces.md`** — written.
- Covered: `WireSpace` and its two default methods, Contract as public API, the naming convention, `@Import` with a Contract versus a Space, the eight binding methods of the Wire Context builder, `ContractSpaceResolver` and `ContractResolution`, and `@ContractAggregation`.
- `BeanSpace` and the `@Bean` legacy variant stay out, as decided.

Findings, now used in the page. The default packages and suffixes come from `bindContracts(String basePackage)` and the constants `DEFAULT_CONTRACT_SUFFIX` and `DEFAULT_SPACE_SUFFIX` in `WireContextBuilderImpl`; the resolver is `NameConventionContractSpaceResolver`. `@ContractAggregation` is implemented by `ContractAggregationInvocationHandler`, which maps only the methods of the extended Contracts. `@Import` fields are made public and recorded by the Space enricher.

**`managed-instances.md`** — written.
- Correction found while writing: `@Default` and `@Name` have nothing to do with Managed Instances. They are read by the property lookup mechanism, so they moved to `patterns.md`.
- Correction found while writing: class level `@Managed` is not only a default scope. Without it a Space is not enriched at all, so its `@Managed` methods do nothing. Evidence: the `enrich` entry point of `ClassFileManagedSpaceEnricher`.
- Key finding, now the heart of the cycles section: Wire publishes an instance to its holder at every assignment of the local variable that the method returns. A method that returns an expression publishes nothing, so a cycle through it cannot close. Evidence: `ManagedMethodCodeTransform` in `ClassFileManagedSpaceEnricher`, which collects `publishedVariableSlots` from the last load before `areturn`.
- Parameterized instances: `SingletonScope.ParameterizedSupplier` keeps a `Map<Object, SingletonInstanceHolder>` keyed by the arguments, and never evicts. `ScopeContextualizedSupplier` handles the `ScopeContext` case.
- Scope semantics: `AggregateScope` keys on the first element of the current instance path and closes the sub scope with its owner; `CallerScope` takes the scope of the last element of the path, falling back to `SingletonScope`. The default scope of a Wire Context is `SingletonScope`.

Further findings, now used in the page.
- `Scope` has five values: `inherit`, `singleton`, `caller`, `aggregate` and `prototype`. `@Managed` defaults to `inherit`. Evidence: `Scope` and `Managed` in `wire-api`.
- `@Managed` on the class sets the default scope for the space; it does not make the methods managed. A method is managed only if it carries `@Managed` itself, and a managed method that does not name a scope takes the class default. Evidence: `factoryMethods` and `scopeType` in `ClassFileManagedSpaceEnricher`, and `AbstractSpace` in `wire-test`, annotated `@Managed(Scope.prototype)`.
- The change happens at runtime, when the context is built, not during the build. `WireManagedSpaceFactory` enriches the space class, and `ClassFileManagedSpaceEnricher` shows exactly which code is inserted around the original method body.
- A space class must be in a package that the class name filter of `WireManagedSpaceFactory` covers. If it is not, the class is loaded unenriched, and Wire reports that the space "was not instrumented". A reader meets this as an error message, so the page names it.

**`wire-context.md`** — written.
- Override mechanism, verified: `TransitiveModuleConfigurer` collects Modules depth first into a `LinkedHashSet`, so a dependency configures before the Module that needs it, and `bindContract` writes into a map where the last write wins.

**`patterns.md`** — written.
- Property lookups are `PropertyLookups.create(iface, Function<String,String>)`. Named in the page although the class sits in an `impl` package, because Reflex and application code call it directly and there is no other entry point.
- Supported conversions come from the static converter registry plus `buildConverter`: enums by constant name, and `List`, `Set`, `Map` as comma separated, URL decoded values with `key=value` entries.
- Test support is `AbstractWiredTest` in `wire-junit-support`, which builds the Wire Context in `@Before` and closes it in `@After`.

### generic-model/ — complete

**`generic-model.md`** — written.
- Why a type system for entities and not plain POJOs: reflection without Java reflection, portable to JavaScript, models as data, metadata per property.
- The type lattice: simple types, enum types, entity types, collection types, base type.
- `GenericModelTypeReflection` as the entry point.
- What a "model" is, and why models are artifacts.

**`entity-types.md`** — written.
- `GenericEntity`, the `EntityType<X> T = EntityTypes.T(X.class)` constant, and why the type is an interface.
- Property declaration, property name constants, `Property`, `TransientProperty`.
- Enums: `EnumBase`, `EnumType`, `EnumTypes`.
- Identity: `id`, `globalId`, `partition`.
- Structural annotations: `@Abstract`, `@Initializer`, `@Transient`, `@ForwardDeclaration`, `@SelectiveInformation`, `@ToStringInformation`, `@TypeRestriction`.
- Cloning and traversing: named here as a capability of `EntityType` only. The topic gets its own page, see `cloning-and-traversing.md`.
- Under the hood: instant type weaving — who creates the implementation class, and at which moment. Verify build step versus runtime before writing it.

**`cloning-and-traversing.md`** — written.
- Decision: a dedicated page. The topic is large enough — three artifacts (`gm-core-api`, `gm-traversing-api`, `basic-gm-traversing`) and about 90 types — and it is used from several other pages (sessions, access, services, marshalling), so it must be defined in one place and linked.
- Why generic traversal exists: a value is a graph of entities and collections, and most platform work is a walk over that graph.
- Cloning: `EntityType.clone(CloningContext, StrategyOnCriterionMatch)`, `CloningContext`, `StandardCloningContext`, `ConfigurableCloningContext`.
- What cloning is used for: detach from a session, cut a graph at a boundary, convert absent properties, copy between sessions.
- Traversing: `TraversingContext`, `TraversingVisitor`, `StandardTraversingContext`.
- Traversing criteria: `traversing-criteria-model`, `TraversingCriterion`, `CriterionType`, and the `TC` builder. How a criterion selects a part of the graph.
- `StrategyOnCriterionMatch`: `partialize`, `skip`, `reference`, and what each produces.
- Absence information: `absence-information-model`, and why a partially loaded entity is still a valid entity.
- The traversing engine: `GMT` in `basic-gm-traversing`, `Cloner`, `Skipper`, `PropertyTransferExpert`. Cover the entry point only, not the internals.
- Model paths: `ModelPath`, `ModelPathElement`, and where a path is used.
- Keep out of scope: the visitor internals in `impl`, and the GWT variants.

**`models.md`** — written.
- A model is an artifact. Ignore `model-declaration.xml`, the `<model>` asset nature, and model dependencies.
- The base models: `root-model`, `meta-model`, `resource-model`.
- Model naming convention: `-model`, `-api-model`, `-configuration-model`. Ignore `-deployment-model`, used in the deprecated Cortex platform.
- How a model is built and where the generated code goes.
- `Model`, `GmMetaModel`, and how a model is available at runtime.

**`model-apis.md`** — written.
- `meta-model` and `meta-data-model`: metadata is modeled data, attached to a model element.
- Declaration by annotation (`@Name`, `@Description`, `@Mandatory`, `@Unique`, `@Indexed`, ...) versus configuration in an initializer.
- Defining custom annotations for meta-data with a `META-INF/gmf.mda` file. Examples in `instant-type-weaving-test`
- Selectors: `selector-model`, `UseCaseSelector`, `RoleSelector`, and conflict resolution by priority.
- `cmd-resolver` (`cascading-meta-data`): how to resolve effective metadata for an entity, a property, or an enum constant.
- `model-oracle`: read a model structure without instances — types, properties, dependencies.
- Essential metadata: `essential-meta-data-model`.

**`classpath-resources.md`** — written.
- `META-INF/classpath-resources.txt`: the authored declaration of what an artifact publishes on the classpath.
- `META-INF/classpath-index.txt`: the generated expansion, carried by a jar.
- Resolution per classpath root, and why an IDE launch needs no builder.
- `META-INF/classpath-resource-only` and the pure resource artifact.
- The assembled application mirror and the `index.properties` format.
- The reserved folders `HICONIC-CONF/` and `HICONIC-APP-RESOURCES/` belong to Reflex and are described in `reflex/project-layout.md`.

**`resource.md`** — written.
- `Resource`: the type for unstructured data. The entity describes the file; the payload sits behind a `ResourceSource`.
- Properties: `name`, `mimeType`, `fileSize`, `md5`, `created`, `creator`, `tags`, `specification`, `resourceSource`.
- `ResourceSource` and its kinds. Transient versus persisted, `isTransient()`, `isStreamable()`.
- Creating a Resource: `Resource.createTransient(...)` and the `Resources` helpers for a file, a string and a stream provider.
- Reading the payload: `openStream()`, `writeToStream()`, and what the Session must support.
- The Session API: `ResourceAccess` with `create()`, `retrieve()`, `update()`, `delete()` and `url()`.
- `ResourceSpecification` and the specifications for images, PDF, page count and OCR.
- The Service Requests in `resource-api-model`: upload, download, stream, delete, enrich.
- Declaring a Resource property in a Model, and what that means for the Access.
- Marshalling: link to `marshalling.md`, do not repeat it.

Findings already verified, for Phase 3. These name implementation classes on purpose; none of it goes into the page.
- `Resource` is declared in `gm-core-api` with `@ForwardDeclaration("com.braintribe.gm:resource-model")` and `@SelectiveInformation("${name}")`. It extends `StandardStringIdentifiable`. It carries default methods `isTransient()`, `isStreamable()`, `openStream()`, `writeToStream()`, `assignTransientSource(...)` and the static `createTransient(InputStreamProvider)`.
- `openStream()` first tries a `StreamableSource`. Otherwise it needs a Session that implements `HasResourceReadAccess`, and it fails with `GmSessionRuntimeException` if the Session does not.
- `ResourceSource` is abstract and has one property, `useCase`. `StreamableSource` and `TransientSource` sit in `gm-core-api`; the concrete sources are in `basic-resource-model`: `BlobSource`, `ClassPathSource`, `ConversionServiceSource`, `FileSystemSource`, `FileUploadSource`, `PackagedSource`, `SqlSource`, `StaticSource`, `StringSource`, `TemplateSource`, `UploadSource`, `UrlUploadSource`. `FileResource` sits in `transient-resource-model`.
- `Resources` in `gm-core-api` has static factories for a text, a text with a charset, a `File` and an `InputStreamProvider`.
- The Session API is `ResourceAccess` in `gm-session-api`, extending `ResourceReadAccess`, with the builders `ResourceCreateBuilder`, `ResourceRetrieveBuilder`, `ResourceUpdateBuilder`, `ResourceDeleteBuilder` and `ResourceUrlBuilder`.
- Specifications in `basic-resource-model`: `ImageSpecification`, `RasterImageSpecification`, `VectorImageSpecification`, `PixelDimensionSpecification`, `PhysicalDimensionSpecification`, `PdfSpecification`, `PageCountSpecification`, `OcrSpecification`.
- `resource-api-model` holds the Service Requests, grouped by package: `stream` (`GetResource`, `StreamResource`, `DownloadResource`, `GetBinary`, `StreamBinary`, and the `Source` variants), `persistence` (`UploadResource`, `UploadResources`, `UpdateResource`, `DeleteResource`, `StoreBinary`, `ManipulateBinary` with `PutBinary`, `AppendBinary`, `InsertBinary`, `DeleteBinaryRange`), and `enrichment` (`EnrichResource`). `resource-api-commons-model` adds `StreamRange` and `StreamCondition`.
- Decide in Phase 3 whether `CallStreamCapture` belongs on this page or is out of scope.

**`sessions.md`** — written.
- `GmSession`, `ManagedGmSession`, `PersistenceGmSession`, and what each adds.
- Session creation: `basic-gm-session-factory`, `basic-managed-gm-session`, `basic-persistence-gm-session`.
- `session.create(Type.T)`, attach and detach, `commit()`.
- Manipulations: `manipulation-model`, how a change becomes a recorded manipulation, and why that matters.
- Transient versus persistent sessions.

**`unit-testing.md`** — written.
- The `-test` artifact convention, the `test` archetype, and `gm-unit-test-deps` as the single dependency. JUnit 4 everywhere; no JUnit 5 in the code base.
- Building a Model for a test: `new NewMetaModelGeneration().buildMetaModel(name, types)`, and `TestModelTestTools.createTestModelMetaModel()` for the shared one. Do **not** use `MetaModelTools.provideRawModel(...)` in any example: the maintainer said it will be removed. Several tests still call it; the model-oracle tests show the form to document.
- `GmTestTools`: memory only and temporary file Smood Accesses, Sessions, the bare Smood, in memory Resource storage.
- `GmAssertions` from `gm-assertj-assertions`: entity assertions and `Maybe` assertions.
- Deliberately left out: the older `gm-assertions` artifact (`com.braintribe.utils.junit.assertions.GmAssertions`). It is used by nothing except its own test, while `gm-assertj-assertions` is used by 20 artifacts. Documenting both would only invite the wrong choice.
- Traps taken from javadoc and stated in the page: a memory only Access ignores self assigned partitions unless `setIgnorePartitions(false)` is called; one `InMemoryResourceAccessFactory` stands for one Access and must be shared by every Session on it.

**`access.md`** — written.
- `IncrementalAccess` and `access-api`: the persistence contract.
- Note typical access implementation is `HibernateAccess`, for special cases an in memory only `SmoodAccess`. Ignore `CollaborativeSmoodAccess`
- Queries: `query-model`, `EntityQuery`, `SelectQuery`, `PropertyQuery`, the query parser, and `query-tools`.
- `aop-access` and aspects as the interception point.

**`services.md`** — written.
- `ServiceRequest` as a modeled, evaluable API. Request in, response out, both entity types.
- `Evaluator<ServiceRequest>`, `request.eval(evaluator).get()` and `getReasoned()`.
- The request hierarchy: `StandardRequest`, `AuthorizedRequest`, `DomainRequest`, `DispatchableRequest`.
- Processors: `ServiceProcessor`, `ReasonedServiceProcessor`, `AccessRequestProcessor`.
- Dispatching: `DispatchingServiceProcessor`, `DispatchConfiguration`.
- Interceptors: pre, post, and around processors.
- Service domains and where a request is evaluated.
- `service-annotations` and how a request declares its processor.

**`reasons.md`** — written.
- Why value based error handling instead of exceptions.
- `Reason`, `Maybe<T>`, `Reasons`, `ReasonException`, `UnsatisfiedMaybeTunneling`.
- `essential-reason-model`: `NotFound`, `InvalidArgument`, `Forbidden`, `Canceled`, ...
- How to model an application specific reason, and when it is worth it.
- Aggregation and nesting: `ReasonAggregator`, `whyUnsatisfied()`, `asString()`.
- The rule for choosing between an exception and a reason.

**`configuration.md`** — written.
- Configuration as a model: `modeled-config-api`, `modeled-yaml-config`.
- `configuration-assembly-model` and how a configuration entity is filled.
- Placeholders and variable resolution: `value-descriptor-model`.
- Link to `reflex/configuration.md` for the platform level rules.

**`marshalling.md`** — written.
- `marshaller-api`, `Marshaller`, `GmSerializationOptions`, `GmDeserializationOptions`.
- The marshallers: `yaml-marshaller`, `json-marshaller`, `basic-marshallers`.
- Type information in the output, and the `_type` convention.
- `resource-model` and `Resource`: how binary data is modeled and streamed.

Design rationale confirmed by the maintainer, for use across the documentation.
- Why the type system is small: every kind must be implementable on every integrated platform — SQL databases, YAML values, the JavaScript type system. A larger set is tedious and in places impossible; mapping Java `float` and `double` onto JavaScript numbers is already awkward. And a larger set would buy no modeling power: further calendar or numeric types, or `char` beside `String`, are already expressible. Only the closed value set belongs to this argument. Do not claim "no behaviour on a type", "a flat property model" or "no generics" as reasons; the maintainer did not confirm them.
- Where the modeling flexibility comes from: multiple inheritance of Entity Types, entities as property values and inside collections, and the base type as an escape hatch. Exactly those three.
- Metadata is **not** a modeling mechanism. It is configuration on top of a modeled type. Do not present it as a way to model.

Decisions taken during the review of the Generic Model section.
- No marketing. The rule is in WRITING-STYLE.md, first in the Accuracy section. The old "why not plain Java objects" section of `generic-model.md` was the trigger and is rewritten.
- JavaScript: mentioned once, in `generic-model.md`, and used as motivation nowhere. The documentation describes the Java side.
- Why an Entity Type is an interface: a declaration must contain the data and nothing else; Java offers only an interface for that, and the alternative is a separate syntax plus a build step. An interface also allows several implementations of one type (plain, enhanced, proxy), hides the implementation's utility methods, and permits multiple inheritance. Written up as its own section in `entity-types.md`.

Findings from writing the Generic Model section, kept for the consistency pass.
- Implementation classes for Entity Types are generated at runtime, not by the build. `JvmGenericModelTypeReflection.createEntityType` analyses the compiled interface and calls `GenericModelTypeSynthesis.ensureEntityType`. The model build produces only the Model declaration and the `_XyzModel_` constant class; there is no generated source in a project. Same trap as Wire's word *compile*.
- `EntityType.create()` is `createRaw()` plus `initialize()`, which applies the `@Initializer` defaults. Evidence: `AbstractEntityType`.
- The `@Transient` javadoc says the feature is not supported. That is stale: `Reason`, `TransientSource` and `CallStreamCapture` all use it, and ITW builds transient properties. The page describes it as working.
- `DispatchingServiceProcessor` in `service-weaving` is deprecated. The current base class is `AbstractDispatchingServiceProcessor` in `service-api`.
- `IncrementalAccess` lives in `access-interfaces`, not in `access-api`.

### reflex/

**`reflex.md`** — overview.
- What a Reflex application is: a set of modules assembled into one Wire context, with a service processing core.
- The design principles: modular, declarative configuration, no application server.
- The boot phases at a glance.
- Artifact kinds in a Reflex project.

**`getting-started.md`**
- Create a project from `reflex-project-template`, `reflex-app-template`, `reflex-module-template`, `reflex-model-template`.
- Build and run.
- `hello-world-app` walked through completely.

**`application.md`**
- The application lifecycle: model configuration, service domain configuration, processor registration, platform configuration, `onDeploy`, `onApplicationReady`, `onApplicationShutdown`.
- `RxApplicationState`, `RxApplicationStateManager`, `RxApplicationLivenessChecker`.
- `RxPlatform` and `reflex-platform`.
- Application files and paths: `RxApplicationFilesContract`.

**`modules.md`**
- `RxModule<M extends RxModuleContract>` and `RxModuleContract`: every lifecycle hook, with what belongs in it.
- The `-rx-module` artifact convention, and the `-module-api` companion artifact.
- Module discovery and ordering.
- Exports: `RxExportContract`, `Exports`, and how one module consumes another module's beans.
- The platform contracts a module may import: `RxPlatformContract` and its twelve sub contracts, each with one line.
- A complete worked example from `demo-rx-module`.

Findings already verified, for Phase 3. These name implementation classes on purpose; none of it goes into the page.
- `RxModule<M extends RxModuleContract>` extends `WireTerminalModule<RxModuleContract>`, and its default `configureContext` binds `RxModuleContract` to the class taken from the generic parameter `M`. So the module class names its space through the type argument. Evidence: `RxModule` in `reflex-module-api`.
- `RxModuleContract` declares eleven hooks, all with an empty default: `configureModels`, `configureMainPersistenceModel`, `configureServiceDomains`, `configureMainServiceDomain`, `registerCrossDomainInterceptors`, `registerServiceProcessors`, `registerFallbackProcessors`, `configurePlatform`, `onDeploy`, `onApplicationReady`, `onApplicationShutdown`. Evidence: `RxModuleContract` in `reflex-module-api`. The calling order still has to be read from the platform.
- `RxPlatformContract` gives access to twelve sub contracts: `application()`, `applicationFiles()`, `auth()`, `configuration()`, `execution()`, `marshalling()`, `packagedResources()`, `packagedPublicResources()`, `processLaunch()`, `reflection()`, `serviceProcessing()`, `transientData()`. It also extends `DeprecatedRxPlatformContract`, which Phase 3 must inspect before it recommends anything from it. Evidence: `RxPlatformContract` in `reflex-module-api`.

**`configuration.md`**
- Configuration assembly: how YAML files, environment properties, and defaults become a configuration entity.
- `RxConfigurationContract`, `RxPropertiesContract`, `EnvironmentPropertiesContract`, `SystemPropertiesContract`, `PropertyResolver`.
- Configuration model convention: `-configuration-model`.
- Profiles and overrides, and the precedence order.

**`service-processing.md`**
- Service domains: `ServiceDomain`, `ServiceDomains`, `PlatformServiceDomains`, `ServiceDomainConfiguration`.
- Model configuration: `ModelConfiguration`, `ConfiguredModel`, `ModelSymbol`.
- Processor registration: `ServiceProcessorRegistry`, `ServiceProcessorRegistration`, `ServiceProcessorSymbol`.
- Interceptors: `InterceptorBuilder`, `InterceptorSymbol`, cross domain interceptors.
- How a request reaches a processor, end to end.

**`persistence.md`**
- `access-rx-module`, `smood-access-rx-module`, `hibernate-access-rx-module`.
- How an access is configured and named.
- Initializers: `initializer-manager-api`, `InitializerContract`, `InitializerBackend`, and the JDBC backend.
- `db-rx-module`, `db-configuration-model`, connection pools and checks.
- Resource storage is a separate topic, see `resources.md`.

**`web-api.md`** — the HTTP layer below REST.
- `web-server-rx-module`, `web-server-configuration-model`, `web-server-module-api`: how the embedded server is configured and how a module adds a servlet.
- RPC: `web-rpc-server-rx-module` and `web-rpc-client-rx-module`. Evaluate a service request in another process, with the same request types.
- `web-api-client-rx-module` and `web-api-client-meta-data-model`: call a remote web API as if it were a local service.
- Websockets: `websocket-server-rx-module`.
- Streaming: `web-streaming-server-rx-module`, and its relation to `Resource`, see `resources.md`.
- REST is a separate topic, see `rest-api.md`.

**`rest-api.md`** — the REST layer.
- Two servers exist, and a reader must know which one to use. State the difference first.
- **Service API** (`web-api-server-rx-module`): every service request becomes an HTTP endpoint. `WebApiV1Server`, `StandardWebApiMappingOracle`, `AbstractDdraRestServlet`.
- Mapping a request to a path: `web-api-mapping-meta-data-model`, and how a mapping is declared.
- Endpoint parameters: `web-api-endpoints-model`, `EndpointInput`, `EndpointInputAttribute`.
- Marshalling of request and response, content negotiation, and multipart for `Resource` properties.
- **CRUD API** (`rest-server-rx-module`): generic create, read, update, delete over entities of an access. `RestV2Server` and the handler per HTTP method.
- Reason to HTTP status mapping: `web-api-reason-model`, `DdraEndpointsExceptionHandler`.
- `openapi-v3-rx-module`: the generated contract, and which metadata feeds it.
- Example: the endpoints of `demo-rx-module` and `demo-web-app`, show examples for curl.

Findings already verified, for Phase 3. These name implementation classes on purpose; none of it goes into the page.
- Two REST servers exist and they are not variants of each other. `web-api-server-rx-module` contains `WebApiV1Server` and `StandardWebApiMappingOracle`, and works from request mappings. `rest-server-rx-module` contains `RestV2Server` and one handler class per HTTP method, over entities. Evidence: the source folders of both artifacts.

**`security.md`**
- `security-rx-module`, `web-security-rx-module`, `security-configuration-model`.
- Authentication and user sessions: `security-service-api-model`, `user-session-model`.
- Authorization: `RxAuthContract`, `RoleAuthorization`, `AuthorizedRequest`, role metadata.

**`background-work.md`**
- Workers: `worker-rx-module`, `worker-api`.
- Scheduling: `cron-scheduling-rx-module`.
- Clustering: `cluster-singleton-rx-module`, `leadership-via-locking-rx-module`, `topology-via-messaging-rx-module`, `LiveInstances`.
- Messaging: `messaging-base-rx-module`, `dmb-messaging-rx-module`, `jdbc-messaging-rx-module`.
- Locking: `jdbc-locking-rx-module`.

**`observability.md`**
- Logging: `RxLogManager`, `log-reflection-rx-module`, `logs-rx-module`.
- Health and checks: `check-rx-module`, `check-module-api`, `check-base-model`.
- Platform reflection: `PlatformReflectionContract`, `platform-reflection-api-model`.

**`cli.md`**
- `cli-rx-module`, `cli-base`, `posix-cli-parser`, `cli-api-model`.
- How a service request becomes a command line command.
- `OutputChannel`, `OutputChannels`, `console-output-model`.
- Example: `demo-cli-app`.

**`project-layout.md`**
- Artifact naming conventions, with the rule for each suffix: `-model`, `-api-model`, `-configuration-model`, `-deployment-model`, `-module-api`, `-rx-module`, `-app`, `-setup`, `-test`.
- Group conventions and package conventions.
- What belongs in which artifact, and the dependency directions that must not be broken.

**`testing.md`**
- `reflex-test-commons`, `reflex-platform-test`.
- Testing a module in isolation with a Wire context.
- `gm-assertions` and `gm-assertj-assertions`.
- Example: `initializer-manager-jdbc-test`.

**`resources.md`** — where a Resource payload is stored. The Resource type itself belongs to `generic-model/resource.md`; this page links there and does not repeat it.
- `ResourceStorage` (`reflex-module-api`): the contract a storage must implement, and `AbstractResourceStorage` as the base class.
- The payload API: `resource-storage-api-model` — `StoreResourcePayload`, `GetResourcePayload`, `DownloadResourcePayload`, `PipeResourcePayload`, `DeleteResourcePayload`. Binary access goes through service requests, like everything else.
- Configuration: `resource-storage-configuration-model` — `ResourceStorageConfiguration`, `ResourceStorage`, `FileSystemResourceStorage`.
- Implementation 1, file system: `FsResourceStorage` in `reflex-platform`. The default.
- Implementation 2, database: `JdbcResourceStorage` in `jdbc-resource-storage-rx-module`, configured by `JdbcResourceStorage` in `jdbc-resource-storage-configuration-model`.
- How a storage is selected and bound: `ResourceStorageDeploymentExpert`, `ResourceStorageHelper`.
- Packaged resources: `PackagedResourceStorage`, `RxPackagedResourcesContract`, `RxPackagedPublicResourcesContract`, `packaged-resource-model`. Static files shipped inside an artifact.
- How to write a third storage. Use `AbstractResourceStorageRxTest` from `reflex-test-commons` to verify it.
- Example: `jdbc-resource-storage-rx-module-test` and `reflex-platform-test/FsResourceStorageTest`.

## Work order

Phase 2 creates every file above with headings and a one sentence abstract per section.

Phase 3 expands the pages in this order. The order follows dependency: a page is written after the pages it links to.

1. `wire/wire.md`
2. `wire/contracts-and-spaces.md`
3. `wire/managed-instances.md`
4. `wire/wire-context.md`
5. `wire/patterns.md`
6. `generic-model/generic-model.md`
7. `generic-model/entity-types.md`
8. `generic-model/cloning-and-traversing.md`
9. `generic-model/models.md`
10. `generic-model/reasons.md`
11. `generic-model/model-apis.md`
12. `generic-model/sessions.md`
13. `generic-model/resource.md`
14. `generic-model/access.md`
15. `generic-model/services.md`
16. `generic-model/marshalling.md`
17. `generic-model/configuration.md`
18. `generic-model/classpath-resources.md`
19. `generic-model/unit-testing.md`
20. `reflex/reflex.md`
21. `reflex/modules.md`
22. `reflex/application.md`
23. `reflex/configuration.md`
24. `reflex/service-processing.md`
25. `reflex/persistence.md`
26. `reflex/resources.md`
27. `reflex/web-api.md`
28. `reflex/rest-api.md`
29. `reflex/security.md`
30. `reflex/background-work.md`
31. `reflex/observability.md`
32. `reflex/cli.md`
33. `reflex/project-layout.md`
34. `reflex/testing.md`
35. `reflex/getting-started.md`
36. `README.md` and `GLOSSARY.md`

Phase 4 is the consistency pass: terminology, cross links, duplicate removal, link check.
