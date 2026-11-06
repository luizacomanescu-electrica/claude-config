---
name: spring-backend-coder
description: Write Java Spring Boot backend code in Luiza's style: strict hexagonal architecture with ports, SOLID, records, hand-written mappers, thin controllers, and tests for every branch.
---

# Spring Backend Coder

Write backend code the way Luiza writes it. The stack is **Java 21, Spring Boot 4, Maven, PostgreSQL, Flyway, Spring Security (JWT in a cookie), AWS SDK v2 (S3, SES), springdoc, JUnit 5 + Mockito + Testcontainers**. This style is based on her hr-tool project, upgraded to **strict hexagonal architecture** with a pure domain model.

## 0. Before writing code
- Read the repo's `CLAUDE.md` and `README.md` first. Repo rules win over this skill.
- Look at 1-2 existing classes of the same kind (controller, service, mapper, test) and match them.
- For anything larger than a small change, give a short plan first (which classes, ports and tests), then implement.
- Never edit documentation files or old Flyway migrations unless asked. Never change the message of an existing error.

## 1. Package structure
The base package is `com.electrica.<project_name>` (snake_case, e.g. `performance_management`).
```
adapters/
  in/web/                  controllers, GlobalExceptionHandler, security filters, argument resolvers
  in/web/dto/              request/response records for HTTP only (with validation annotations)
  out/persistence/         adapters implementing persistence ports
  out/persistence/entity/  JPA entities (@Entity)
  out/persistence/repository/  Spring Data interfaces + query projections (*Row)
  out/persistence/mapper/  JPA entity <-> domain mappers
  out/email/               e.g. SesEmailSender implements EmailSenderPort
  out/s3/                  e.g. S3FileStorage implements FileStoragePort
application/
  port/in/                 use-case interfaces (what the app offers)
  port/out/                interfaces the app needs (persistence, email, files, tokens)
  service/                 use-case implementations, plus finders, checkers, planners, listeners
  dto/                     application records (commands, results, responses)
  mapper/                  domain -> application DTO mappers
  event/                   application event records
  exception/               one exception class per business error
domain/
  model/                   pure domain classes (no Spring, no JPA, no Jackson)
  enums/
config/                    Spring configuration (Security, Clock, S3, SES, WebMvc, JWT)
util/                      small shared helpers (e.g. RomanianTime)
```
The test tree mirrors main, plus `testsupport/` (the Testcontainers config, `@PostgresDataJpaTest`, and test data builders).

## 2. Dependency rules (strict hexagonal)
- `domain` depends on nothing (plain Java only).
- `application` depends only on `domain` and its own ports. It **never imports from `adapters`** (no repositories, no `S3FileStorage`, no AWS SDK, no JPA entities, no `jakarta.servlet`).
- `adapters.in` call **input ports** (`application/port/in`), not service classes.
- `adapters.out` implement **output ports** (`application/port/out`).
- `config` wires things together and may see everything.
- Objects don't travel across layers: web DTO -> (map) -> application command -> domain -> (map) -> application response -> controller. JPA entities never leave `adapters/out/persistence`. Domain objects never go out in an HTTP response.

### Port naming
- Input ports: `<Action><Thing>UseCase` (e.g. `ApproveEvaluatorsUseCase`, `CreateUserUseCase`) or a cohesive `<Thing>UseCase` when the methods belong together. Keep them small (interface segregation).
- Output ports: by capability, e.g. `UserPersistencePort` (or `LoadUserPort` + `SaveUserPort` when it helps), `FileStoragePort`, `EmailSenderPort`, `TokenProviderPort`.
- Services implement input ports: `class Feedback360SubjectReviewService implements ApproveEvaluatorsUseCase, ListOwnEvaluatorsUseCase`.

## 3. Class conventions

### General
- **Constructor injection, written by hand**, with all fields `private final` and no `@Autowired` and no Lombok constructors. Configuration values come in as constructor parameters: `@Value("${app.cookie-secure:true}") boolean secureCookie`.
- **Records wherever possible**: requests, responses, commands, results, events and value objects. Use small helper methods on records (e.g. `withName(...)`, `toCommand()`) for conversion.
- Prefer `Optional` + `orElseThrow(() -> new XxxNotFoundException("..."))`, streams with `.toList()`, and early returns / guard clauses.
- Constants for magic strings, for example statuses on the domain class: `Feedback360Session.STATUS_DRAFT` (or an enum).
- Logging: `private static final Logger log = LoggerFactory.getLogger(Xxx.class);` (SLF4J). Use `{}` placeholders and pass the exception last.
- Single responsibility: split large services into focused collaborators named by role, such as `XxxFinder` (get-or-throw, getDraft), `XxxChecker`, `XxxPlanner`, `XxxBuilder`, `XxxSender`, `XxxListener`.
- Time: store `Instant` (TIMESTAMPTZ). Inject `java.time.Clock` (from `ClockConfig`). Never use the JVM's default zone. Use Romanian time only for display or files, through the shared constant.

### Controllers (`adapters/in/web`)
- `@RestController` + `@RequestMapping("/api/<resource>")`, named `<Thing>Controller`. Keep them thin, with no business rules.
- Return `ResponseEntity<...>`: `ok(body)`, `ok().build()`, `noContent().build()`, `status(CREATED)`.
- `@Valid @RequestBody XxxRequest request`, then convert the request record to an application command (`request.toCommand()`) and call the use case.
- Get the current user with the `@AuthenticatedUserId UUID userId` argument resolver. For admin-only endpoints, use the `@PreAuthorize` pattern.
- Private helpers for repeated HTTP details (e.g. `buildJwtCookie`).

### Services (`application/service`)
- `@Service` for use cases, `@Component` for helpers (finders, checkers, listeners).
- `@Transactional` on methods that write. Use `findByIdForUpdate` (a pessimistic lock) when concurrent changes matter.
- Validate state with guard clauses that throw specific business exceptions with clear messages.
- Publish events with `ApplicationEventPublisher` and an event record. Handle side effects (emails) in `@TransactionalEventListener(phase = AFTER_COMMIT)` listeners that catch and log failures so they don't break the main flow.
- Record audit entries through the audit service where the domain needs it.

### Mappers
- Hand-written `@Component` classes, one per aggregate/area, with methods named `toResponse`, `toSummaryResponse`, `toDomain`, `toEntity`, `toXxxRow`.
- They build records through the canonical constructor, one argument per line, with explicit null-safe access: `x.getDepartment() != null ? x.getDepartment().getName() : null`.
- Application mappers turn domain into application DTOs. Persistence mappers turn JPA entities into domain objects and back. A mapper that needs infrastructure (e.g. presigned picture URLs) uses an **output port** (`FileStoragePort`), never an adapter class.

### Domain model (`domain/model`)
- Pure Java, with no JPA, Spring or Jackson annotations.
- Put behavior on the model where it belongs (e.g. `session.isDraft()`, `subject.approveEvaluators()`) instead of setters plus logic in services. Expose getters, and avoid public setters where a behavior method is clearer.

### Persistence (`adapters/out/persistence`)
- JPA entities: `@Entity @Table(name = "snake_case_plural")`, with Lombok `@Getter @Setter` and `@NoArgsConstructor` allowed **only here**. IDs are `UUID` for business entities and `Integer` IDENTITY for lookup tables.
- Spring Data repositories are `<Thing>Repository extends JpaRepository<...>`. Use interface projections named `*Row` / `*Count` for read queries.
- The adapter class (e.g. `UserPersistenceAdapter implements UserPersistencePort`) uses the repository and the persistence mapper.

### Exceptions
- One class per business error in `application/exception`: `public class SessionNotFoundException extends RuntimeException { public SessionNotFoundException(String message) { super(message); } }`.
- Map each one in `GlobalExceptionHandler` (`@RestControllerAdvice`) to its status: not found 404, conflict or wrong state 409, invalid data 400, bad credentials 401, access denied 403, and an unexpected error is 500 with a generic message logged with `log.error`. Validation errors return a map of field to message.

### Database
- Flyway: always a **new** file `V<next>__snake_case_description.sql`, and never edit an existing one. Seeds also go through migrations.

### Configuration
- `application.yaml` + `application-local.yaml` (gitignored) + `application-local.yaml.example` + `application-dev.yaml` + `application-prod.yaml`. Secrets come from the environment, never committed.

### Security
- The JWT lives in an httpOnly `jwt` cookie (secure flag from config, `SameSite=Lax`). 401 means no valid login, 403 means the wrong role.
- **Passwords are always hashed** with Spring Security's `PasswordEncoder` (BCrypt or Argon2) and compared with `passwordEncoder.matches(...)`, never with `equals`. Use the same message for an unknown user and a wrong password.

## 4. Tests
- JUnit 5. Names are `method_condition_expectedResult` (e.g. `approve_whenListAlreadyApproved_throwsEvaluatorListLocked`).
- Unit tests: `@ExtendWith(MockitoExtension.class)`, with `@Mock` for ports and the class under test **built with `new` in `@BeforeEach`**, real mappers or finders where they are simple. Arrange, act, assert. Use `verify`, `never()` and `inOrder` for side effects, and `assertThrows` for refusals.
- Persistence tests: `@PostgresDataJpaTest` (Testcontainers, real Postgres 16) with the `testsupport` data builders (JdbcTemplate inserts).
- Security and HTTP: MockMvc with the real security configuration.
- **Every new rule gets a test for each branch, including the refusals.** Do not weaken or delete existing tests. If one must change, say which and why.
- Run the affected tests while working (`mvn -q -Dtest=ClassName test`) and `mvn clean test` at the end (Docker must be running). Report only the summary and any failures.

## 5. Finishing
Summarize what changed (the files and why), the tests added, and anything left open. Point out any place where existing code breaks these rules, and suggest a fix without changing unrelated code.
