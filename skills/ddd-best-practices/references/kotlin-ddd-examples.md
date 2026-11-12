# Kotlin DDD Examples (CodelyTV Pattern)

Kotlin ports of the TypeScript DDD building blocks in `typescript-ddd-examples.md`, based on the CodelyTV `typescript-ddd-example` repository. These show the full stack of building blocks working together in a real bounded context. `Promise` maps to `suspend`; Jest maps to JUnit 5, MockK, and AssertJ.

Aggregates here are immutable and failures are values (Arrow `Either`); see `fp-best-practices` for the functional style these examples follow.

Source: https://github.com/CodelyTV/typescript-ddd-example

---

## Folder Structure

**Intent:** Organize code by Bounded Context, not by technical layer.

**How it works:** A `contexts` package holds one package per bounded context (e.g., `mooc`, `backoffice`). Each context contains sub-domains organized by aggregate. A `shared` context holds cross-cutting building blocks (error marker, buses, criteria). Inside each aggregate package, the split is always `domain` → `application` → `infrastructure`.

**Example:**
```
src/
  main/kotlin/com/example/contexts/
    mooc/
      courses/
        domain/
          Course.kt                    # Aggregate Root
          CourseResult.kt              # State + Events
          CourseError.kt               # Typed failures
          CourseRepository.kt          # Port (interface)
          CourseName.kt                # Value Object
          CourseDuration.kt            # Value Object
          CourseDomainEvent.kt         # Domain Events (sealed)
        application/
          create/
            CourseCreator.kt           # Use Case (Application Service)
            CreateCourseCommand.kt     # Command DTO
            CreateCourseCommandHandler.kt
        infrastructure/
          persistence/
            postgres/
              PostgresCourseRepository.kt # Driven Adapter
    shared/
      domain/
        DomainError.kt
  test/kotlin/com/example/contexts/
    mooc/
      courses/
        domain/CourseMother.kt         # Object Mother
        application/CreateCourse...Test.kt
```

**Practical heuristic:** If you cannot tell from the package name alone which business capability the code serves, the structure is wrong. Package names like `controllers` or `services` are red flags.

---

## Value Objects in Kotlin

**Intent:** Give domain values explicit construction, immutable representation, and semantic equality.

**How it works:** A `value class` or `data class` with a private constructor and a validating factory that returns `Either` — invalid input is a typed error value, not an exception. Both generate structural `equals`/`hashCode`, so no shared base class is needed. Override equality only when the generated one is wrong for the domain: `Array` properties compare by reference, and `BigDecimal` equality includes scale.

**Example:**
```kotlin
// contexts/shared/domain/DomainError.kt
interface DomainError

// contexts/mooc/courses/domain/CourseError.kt
sealed interface CourseError : DomainError {
    data class InvalidCourseName(val value: String) : CourseError
    data class InvalidCourseId(val value: String) : CourseError
    data object InvalidCriteria : CourseError
    data class InfrastructureError(val reason: String, val cause: Throwable? = null) : CourseError
}

// contexts/mooc/courses/domain/CourseName.kt
@JvmInline
value class CourseName private constructor(val value: String) {
    companion object {
        fun create(raw: String): Either<CourseError.InvalidCourseName, CourseName> = either {
            val value = raw.trim()
            ensure(value.isNotEmpty() && value.length <= 30) { CourseError.InvalidCourseName(value) }
            CourseName(value)
        }
    }
}

// contexts/mooc/shared/domain/courses/CourseId.kt
@JvmInline
value class CourseId private constructor(val value: String) {
    companion object {
        fun create(value: String): Either<CourseError.InvalidCourseId, CourseId> = either {
            ensure(isValidUuid(value)) { CourseError.InvalidCourseId(value) }
            CourseId(value)
        }
    }
}
```

**Practical heuristic:** Transport DTOs, commands, and messages normally carry primitives. Convert at a deliberate application/domain boundary, then use Value Objects in domain APIs where semantic distinction or guarantees justify them.

---

## Aggregate Result (State + Events)

**Intent:** Return the aggregate's new state and the domain events it produced together, as one immutable value — no mutable event list hidden inside the aggregate.

**How it works:** Every aggregate operation returns a `CourseResult`: the resulting `Course` plus the events that explain the change. `plus` chains steps — the later state wins and events accumulate in order. Nothing is drained or cleared, so the only copy of an event cannot be lost before durable handoff.

**Example:**
```kotlin
// contexts/mooc/courses/domain/CourseResult.kt
data class CourseResult(val course: Course, val events: List<CourseDomainEvent>) {
    operator fun plus(other: CourseResult): CourseResult =
        CourseResult(other.course, events + other.events)
}
```

**Practical heuristic:** The application persists the whole `CourseResult` atomically — state and Outbox rows in one transaction. The Aggregate never publishes; use `infrastructure-design` for transactional Outbox delivery.

---

## Aggregate Root (Concrete)

**Intent:** Implement an immutable aggregate whose factory enforces valid creation and returns the resulting state together with its domain event.

**How it works:** `Course` is a `data class` of Value Objects, never raw primitives. `Course.create()` is a pure function: event id and time are passed in (from an injected clock and UUID generator), and it returns a `CourseResult` holding the new `Course` and `CourseCreated`. `version` is `null` until the first persist; the adapter uses it for optimistic locking. Reconstitution is the adapter building the data class directly from stored values — no events.

**Example:**
```kotlin
// contexts/mooc/courses/domain/Course.kt
data class Course(
    val id: CourseId,
    val name: CourseName,
    val duration: CourseDuration,
    val version: Long?,
) {
    companion object {
        // Creation: a new Course plus the event that records it
        fun create(
            id: CourseId,
            name: CourseName,
            duration: CourseDuration,
            eventId: UUID,
            now: Instant,
        ): CourseResult =
            CourseResult(
                course = Course(id, name, duration, version = null),
                events = listOf(CourseDomainEvent.CourseCreated(eventId, now, id, name, duration)),
            )
    }
}
```

**Practical heuristic:** Keep creation and reconstitution semantically separate so loading cannot emit creation events. Only domain functions that change state return events; the adapter's reconstitution returns a bare `Course`.

---

## Domain Events (Sealed)

**Intent:** Give every domain event a consistent structure — event ID, timestamp, aggregate ID — while letting each concrete event define its own typed attributes.

**How it works:** A `sealed interface` per aggregate lists every event it can produce, so adapters handle them with an exhaustive `when`. Each event is a `data class` of Value Objects. Metadata is passed in, never generated inside the domain: the use case supplies `eventId` and `occurredOn` from an injected UUID generator and `Clock`, which keeps creation deterministic and testable. Serialization belongs to the adapter.

**Example:**
```kotlin
// contexts/mooc/courses/domain/CourseDomainEvent.kt
sealed interface CourseDomainEvent {
    data class CourseCreated(
        val eventId: UUID,
        val occurredOn: Instant,
        val courseId: CourseId,
        val name: CourseName,
        val duration: CourseDuration,
    ) : CourseDomainEvent
}
```

**Practical heuristic:** Name domain events in the past tense. The event name the adapter writes (`course.created`) and its wire format are the stable contract between producers and consumers — treat them as an immutable API once deployed, even when the Kotlin class is renamed.

---

## Repository Port (Interface)

**Intent:** Declare what the domain needs from persistence in domain language, with no reference to any database technology.

**How it works:** The repository interface lives in the `domain` package — it is part of the domain layer, not infrastructure. Method names use domain vocabulary. Return types are `Either` of domain objects or aggregates, never raw database rows; infrastructure failures are values too. A single `persist(result)` takes the whole `CourseResult` so state and events are written together. The interface is the driven port; concrete adapters (Postgres, in-memory) implement it in `infrastructure`.

**Example:**
```kotlin
// contexts/mooc/courses/domain/CourseRepository.kt
interface CourseRepository {
    suspend fun persist(result: CourseResult): Either<CourseError.InfrastructureError, Unit>
    suspend fun searchAll(): Either<CourseError.InfrastructureError, List<Course>>
}
```

**Practical heuristic:** If you find yourself adding methods like `findByRawQuery()` or `executeSql()` to a repository interface, the abstraction has broken down. Keep the interface in domain terms only.

---

## Event Delivery (Outbox)

**Intent:** Deliver domain events to other processes reliably, without the domain or the use case depending on RabbitMQ, Kafka, or any specific broker.

**How it works:** `persist(result)` writes the aggregate state and one Outbox row per event in the same database transaction, so either both are stored or neither is. A separate relay reads the Outbox and publishes to the broker, retrying until acknowledged. Delivery is therefore at-least-once: consumers deduplicate by event ID and are idempotent.

**Practical heuristic:** Neither the Aggregate nor the use case publishes. If an event must survive a crash, it goes through the Outbox; use `infrastructure-design` for relay, ordering, retry, and cleanup details.

---

## Use Case (Application Service)

**Intent:** Orchestrate the domain — load, decide via the aggregate, persist the result — with no business logic of its own.

**How it works:** `CourseCreator` receives dependencies through constructor injection, including a `Clock` and a UUID generator so the domain stays pure. Inside `either {}`, each step short-circuits on the first error. The Command Handler translates the flat DTO command into typed Value Objects; invalid input returns a `Left` instead of throwing.

**Example:**
```kotlin
// contexts/mooc/courses/application/create/CourseCreator.kt
class CourseCreator(
    private val repository: CourseRepository,
    private val clock: Clock,
    private val generateUUID: () -> UUID,
) {
    suspend fun run(id: CourseId, name: CourseName, duration: CourseDuration): Either<CourseError, Unit> =
        either {
            val result = Course.create(id, name, duration, generateUUID(), clock.instant())
            repository.persist(result).bind()
        }
}

// contexts/mooc/courses/application/create/CreateCourseCommandHandler.kt
class CreateCourseCommandHandler(
    private val courseCreator: CourseCreator,
) : CommandHandler<CreateCourseCommand> {

    override fun subscribedTo(): KClass<CreateCourseCommand> = CreateCourseCommand::class

    override suspend fun handle(command: CreateCourseCommand): Either<DomainError, Unit> = either {
        val id = CourseId.create(command.id).bind()
        val name = CourseName.create(command.name).bind()
        val duration = CourseDuration(command.duration)
        courseCreator.run(id, name, duration).bind()
    }
}
```

**Practical heuristic:** A use case may branch for orchestration, absence, authorization outcomes, retries, or policy results, but it must not own domain policy that belongs in the model. Treat it as a traffic controller, not as a rule-free syntax exercise.

---

## Infrastructure: Postgres Repository (Driven Adapter)

**Intent:** Implement the repository port with Postgres and jOOQ, writing state and events atomically and mapping stored rows back to typed domain objects.

**How it works:** `persist` runs in one transaction: it inserts a new course (`version == null`) or updates an existing one only if the stored version still matches, then inserts one Outbox row per event. An update that matches 0 rows means someone else changed the course first — returned as an error, never silently overwritten. An exhaustive `when` over the sealed events owns the event name and payload. Exceptions are caught at this boundary and become `InfrastructureError`. Reconstitution goes through the Value Object factories, so corrupt stored data surfaces as an error instead of a crash.

**Example:**
```kotlin
import org.jooq.kotlin.coroutines.transactionCoroutine
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.reactive.asFlow
import kotlinx.coroutines.reactive.awaitSingle
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json

// contexts/mooc/courses/infrastructure/persistence/postgres/PostgresCourseRepository.kt
class PostgresCourseRepository(private val dsl: DSLContext) : CourseRepository {

    override suspend fun persist(result: CourseResult): Either<CourseError.InfrastructureError, Unit> = either {
        val written = Either.catch {
            dsl.transactionCoroutine { config ->
                val tx = config.dsl()
                val course = result.course
                val rows = when (val version = course.version) {
                    null -> tx.insertInto(COURSE)
                        .set(COURSE.ID, course.id.value)
                        .set(COURSE.NAME, course.name.value)
                        .set(COURSE.DURATION, course.duration.value)
                        .set(COURSE.VERSION, 0L)
                        .awaitSingle()
                    else -> tx.update(COURSE)
                        .set(COURSE.NAME, course.name.value)
                        .set(COURSE.DURATION, course.duration.value)
                        .set(COURSE.VERSION, version + 1)
                        .where(COURSE.ID.eq(course.id.value))
                        .and(COURSE.VERSION.eq(version))
                        .awaitSingle()
                }
                if (rows == 0) return@transactionCoroutine false
                result.events.forEach { event ->
                    when (event) {
                        is CourseDomainEvent.CourseCreated -> tx.insertInto(OUTBOX)
                            .set(OUTBOX.ID, event.eventId)
                            .set(OUTBOX.EVENT_NAME, "course.created")
                            .set(OUTBOX.AGGREGATE_ID, event.courseId.value)
                            .set(OUTBOX.OCCURRED_ON, event.occurredOn)
                            .set(
                                OUTBOX.PAYLOAD,
                                Json.encodeToString(mapOf("name" to event.name.value, "duration" to event.duration.value)),
                            )
                            .awaitSingle()
                    }
                }
                true
            }
        }.mapLeft { CourseError.InfrastructureError(it.message ?: "", it) }.bind()
        ensure(written) { CourseError.InfrastructureError("stale version") }
    }

    override suspend fun searchAll(): Either<CourseError.InfrastructureError, List<Course>> = either {
        val records = Either.catch { dsl.selectFrom(COURSE).asFlow().toList() }
            .mapLeft { CourseError.InfrastructureError(it.message ?: "", it) }
            .bind()
        records.map { r ->
            either {
                Course(CourseId.create(r.id).bind(), CourseName.create(r.name).bind(), CourseDuration(r.duration), r.version)
            }.mapLeft { CourseError.InfrastructureError("corrupt course ${r.id}: $it") }.bind()
        }
    }
}
```

**Practical heuristic:** The adapter is the only place that knows table columns, event names, and wire formats. Everything else in the system works with typed domain objects and `Either`.

---

## Command / Query Contracts

**Intent:** Give bus infrastructure type-safe command, query, response, and handler relationships.

**How it works:** Kotlin types are nominal, so marker interfaces distinguish command and query families — TypeScript's brand fields are unnecessary. A query carries its response type. Handlers return the `KClass` they subscribe to, not a message instance.

**Example:**
```kotlin
interface Command

interface Query<R>

// Serialized/external commands normally carry flat primitives.
// contexts/mooc/courses/application/create/CreateCourseCommand.kt
data class CreateCourseCommand(
    val id: String,
    val name: String,
    val duration: String,
) : Command
```

**Practical heuristic:** Serialized commands normally carry transport primitives and are converted at the handler boundary. A trusted in-process command may carry Value Objects when that contract is deliberate and no serialization boundary is implied.

---

## CommandBus / QueryBus — CQRS Bus Interfaces

**Intent:** Decouple the controller (driver adapter) from the use case by routing through a bus. The controller does not need to know which handler exists — it only dispatches a command or queries with a query.

**How it works:** `CommandBus` dispatches a command and returns completion or a domain error. `QueryBus` infers the response type carried by `Query<R>`. Both are interfaces implemented in infrastructure.

**Example:**
```kotlin
// contexts/shared/domain/CommandBus.kt
interface CommandBus {
    suspend fun dispatch(command: Command): Either<DomainError, Unit>
}

// contexts/shared/domain/QueryBus.kt
interface QueryBus {
    suspend fun <R> ask(query: Query<R>): R
}

// contexts/shared/domain/CommandHandler.kt
interface CommandHandler<T : Command> {
    fun subscribedTo(): KClass<T>
    suspend fun handle(command: T): Either<DomainError, Unit>
}

// contexts/shared/domain/QueryHandler.kt
interface QueryHandler<R, Q : Query<R>> {
    fun subscribedTo(): KClass<Q>
    suspend fun handle(query: Q): R
}
```

**Practical heuristic:** CQRS split is at the bus level: commands mutate state and return only success or a domain error; queries read state and return a response. Never mix the two in one handler.

---

## CommandBus / QueryBus — In-Memory Implementations

**Intent:** Provide an in-process bus for development and testing. The registry (`CommandHandlers`, `QueryHandlers`) maps command/query classes to their handlers.

**How it works:** The registry maps message classes to handlers. Handlers register the concrete class returned by `subscribedTo()`. The bus looks up `command::class` at dispatch time and fails explicitly when no handler exists.

**Example:**
```kotlin
// contexts/shared/infrastructure/commandbus/CommandHandlers.kt
class CommandHandlers(commandHandlers: List<CommandHandler<*>>) {
    private val handlers: Map<KClass<*>, CommandHandler<*>> =
        commandHandlers.associateBy { it.subscribedTo() }

    // Safe: each handler is registered under the class it handles
    @Suppress("UNCHECKED_CAST")
    fun <T : Command> get(command: T): CommandHandler<T> =
        handlers[command::class] as CommandHandler<T>?
            ?: throw CommandNotRegisteredError(command)
}

// contexts/shared/infrastructure/commandbus/InMemoryCommandBus.kt
class InMemoryCommandBus(private val commandHandlers: CommandHandlers) : CommandBus {
    override suspend fun dispatch(command: Command): Either<DomainError, Unit> =
        commandHandlers.get(command).handle(command)
}
```

**Practical heuristic:** The lookup key is the message class, not an instance. Mirror the same typed class registry for queries. More elaborate buses can use explicit stable message names when classes cannot cross serialization boundaries.

---

## Nullable Types — Absence

**Intent:** Express one canonical absence representation in a domain API.

**Example:**
```kotlin
// Kotlin's built-in T? is the single absence representation — no utility type needed
interface CourseRepository {
    suspend fun findById(id: CourseId): Either<CourseError.InfrastructureError, Course?>
}
```

**Practical heuristic:** Absence is `null` inside `Right`; `Left` is reserved for failures. Use `T?` in domain APIs rather than `java.util.Optional`, Arrow `Option`, or sentinel values. Accept other representations (Java platform types, `Optional`) only at an external boundary and normalize to `T?` immediately.

---

## Criteria Pattern — Flexible Repository Queries

**Intent:** Express genuinely dynamic query conditions without exposing SQL or storage syntax. Keep stable business searches as named methods and use a read-model query port for projections.

**How it works:** A validated immutable Criteria carries a typed predicate tree, stable logical fields, order, and one bounded pagination variant. Infrastructure maps logical fields/operators to backend-native expressions and rejects unsupported semantics.

**Example:**
```kotlin
sealed interface Predicate {
    data class Comparison(val field: Field, val operator: Operator, val value: String) : Predicate {
        enum class Field { NAME, PUBLISHED_AT, ID }
        enum class Operator { EQ, CONTAINS, GT }
    }

    data class And(val operands: List<Predicate>) : Predicate
}

data class Order(val field: Field, val direction: Direction) {
    enum class Field { PUBLISHED_AT, ID }
    enum class Direction { ASC, DESC }
}

data class Page(val limit: Int, val offset: Int)

data class Criteria(
    val predicate: Predicate?,
    val order: List<Order>,
    val page: Page,
)

fun courseCriteria(input: Criteria): Either<CourseError.InvalidCriteria, Criteria> = either {
    ensure(input.page.limit in 1..100) { CourseError.InvalidCriteria }
    ensure(input.page.offset >= 0) { CourseError.InvalidCriteria }
    ensure(input.order.lastOrNull()?.field == Order.Field.ID) { CourseError.InvalidCriteria }
    // vals are already immutable; copy the list in case the caller passed a MutableList
    input.copy(order = input.order.toList())
}

interface CourseRepository {
    suspend fun matching(criteria: Criteria): Either<CourseError.InfrastructureError, List<Course>>
}
```

**Practical heuristic:** Criteria's layer follows its vocabulary: domain-named selectors may live with an Aggregate repository; generic filters belong to the application/query core. No HTTP parameter, SQL identifier, or Elasticsearch DSL belongs in this contract. Use `design-patterns-best-practices` for converter, pagination, and testing details.

---

## Object Mother Pattern — Test Data Factories

**Intent:** Centralize valid test data creation so tests stay short and readable. Each domain concept gets its own Mother object with deterministic defaults and focused overrides.

**How it works:** Each Mother has `create(explicit params)` and may provide seeded generation for intentional variation. The test specifies every value relevant to its rule. If randomness is used, inject/print the seed so failures reproduce exactly.

**Example:**
```kotlin
// src/test/kotlin/.../contexts/mooc/courses/domain/CourseMother.kt
object CourseMother {
    fun create(id: CourseId, name: CourseName, duration: CourseDuration): Course =
        Course(id, name, duration, version = null)

    fun from(command: CreateCourseCommand): Course =
        create(
            CourseIdMother.create(command.id),
            CourseNameMother.create(command.name),
            CourseDurationMother.create(command.duration),
        )

    fun random(seed: Long): Course =
        create(
            CourseIdMother.random(seed),
            CourseNameMother.random(seed + 1),
            CourseDurationMother.random(seed + 2),
        )
}

// src/test/kotlin/.../contexts/mooc/courses/domain/CourseNameMother.kt
object CourseNameMother {
    fun create(value: String): CourseName =
        CourseName.create(value).getOrElse { error("invalid fixture: $it") }
    fun random(seed: Long): CourseName = create(WordMother.random(seed = seed, maxLength = 20))
    fun invalidName(): String = "a".repeat(40)
}

// src/test/kotlin/.../contexts/mooc/courses/application/CreateCourseCommandMother.kt
object CreateCourseCommandMother {
    fun create(id: CourseId, name: CourseName, duration: CourseDuration): CreateCourseCommand =
        CreateCourseCommand(id = id.value, name = name.value, duration = duration.value)

    fun random(seed: Long): CreateCourseCommand =
        create(
            CourseIdMother.random(seed),
            CourseNameMother.random(seed + 1),
            CourseDurationMother.random(seed + 2),
        )

    fun invalid(seed: Long): CreateCourseCommand =
        CreateCourseCommand(
            id = CourseIdMother.random(seed).value,
            name = CourseNameMother.invalidName(), // 40 chars — exceeds limit
            duration = CourseDurationMother.random(seed + 1).value,
        )
}
```

**Practical heuristic:** Command Mothers produce flat primitives; Domain Mothers produce Value Objects and unwrap factory `Either`s, failing loudly on an invalid fixture. Keep them at the right layer — don't mix them.

---

## Unit Test Structure — Command Handler + Mock Repository

**Intent:** Show the idiomatic test structure: inject a mock repository, a fixed clock, and a fixed UUID; use Object Mothers to build inputs and expectations; assert on the returned `Either` and on what was persisted.

**How it works:** JUnit 5 creates a new test class instance per test, so mocks declared as properties are fresh for every test. `Course`, `CourseResult`, and events are data classes, so the expected persisted result compares by value; the fixed clock and UUID make event metadata deterministic. Invalid input returns a `Left` and nothing is persisted.

**Example:**
```kotlin
// src/test/kotlin/.../contexts/mooc/courses/application/CreateCourseCommandHandlerTest.kt
class CreateCourseCommandHandlerTest {

    private val now = Instant.parse("2026-01-01T00:00:00Z")
    private val eventId = UUID.fromString("00000000-0000-0000-0000-000000000001")
    private val repository = mockk<CourseRepository> {
        coEvery { persist(any()) } returns Unit.right()
    }
    private val creator = CourseCreator(repository, Clock.fixed(now, ZoneOffset.UTC)) { eventId }
    private val handler = CreateCourseCommandHandler(creator)

    @Test
    fun `should create a valid course`() = runTest {
        val command = CreateCourseCommandMother.random(42)
        val course = CourseMother.from(command)

        assertThat(handler.handle(command)).isEqualTo(Unit.right())

        coVerify {
            repository.persist(
                CourseResult(course, listOf(CourseCreated(eventId, now, course.id, course.name, course.duration))),
            )
        }
    }

    @Test
    fun `should return an error if course name length is exceeded`() = runTest {
        assertThat(handler.handle(CreateCourseCommandMother.invalid(42)))
            .isEqualTo(InvalidCourseName("a".repeat(40)).left())

        coVerify(exactly = 0) { repository.persist(any()) }
    }
}
```

**Practical heuristic:** Mothers provide valid deterministic defaults; override values relevant to the behavior. Verify calls after Act so a missing interaction cannot skip an assertion. Use seeded generation only when variation is intentional and reproducible.
