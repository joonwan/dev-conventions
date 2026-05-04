# Spring Boot AI Rules

Use these rules when generating or modifying Spring Boot code.
When rules are ambiguous, choose the most conservative design.

## Package Structure

- Use domain-based packages.
- Do NOT use top-level layer packages such as `controller/`, `service/`, or `repository/`.
- Put shared code under `global`.
- Put domain code under `domain/{domain}`.
- Each domain MAY contain `controller`, `service`, `repository`, `entity`, `client`, and `exception` packages.

## Controller

- Controllers MUST handle only HTTP request and response concerns.
- Controllers MUST NOT contain business logic.
- Controllers MUST validate request DTOs with `@Valid`.
- Controllers MUST convert `XxxRequest` to `XxxServiceRequest` before calling services.
- Controllers MUST NOT pass controller DTOs directly to services.
- Controllers MUST wrap responses with `ResponseEntity<ApiResponse<...>>`.
- Services MUST NOT return `ResponseEntity`.

## Service

- Services MUST contain business logic.
- Services MUST NOT return entities directly.
- Services MUST return service response DTOs such as `XxxServiceResponse`.
- Services MUST NOT accept controller DTOs such as `XxxRequest`.
- Services MUST accept service DTOs such as `XxxServiceRequest`.
- Convert entities to service response DTOs with `XxxServiceResponse.from(entity)`.
- Services MUST NOT instantiate service response DTOs directly with `new XxxServiceResponse(...)`.
- Add `@Transactional` ONLY to services that access repositories, `EntityManager`, or another data access component.
- For data-access services, add `@Transactional(readOnly = true)` at class level.
- Add method-level `@Transactional` ONLY to write methods.
- Do NOT add `@Transactional` to services without data access.
- Services MUST NOT call external APIs directly. Use `XxxClient`.

## Repository

- Repositories MUST extend `JpaRepository`.
- Use Spring Data JPA method names for simple queries.
- Use QueryDSL ONLY when complex dynamic queries are needed.

## External API Clients

- External API calls MUST be encapsulated in `XxxClient`.
- Put shared clients under `global/client/{xxx}`.
- Put domain-only clients under `domain/{domain}/client/{xxx}`.
- Client classes MUST be named `XxxClient`.
- External API settings MUST use `@ConfigurationProperties`.
- Do NOT inject external API settings with `@Value`.
- External API exceptions MUST be converted to dedicated `XxxException`.
- External API error cases MUST distinguish status errors, network errors, and timeouts when possible.
- `XxxClient` MUST contain only external API calls and exception conversion.
- `XxxClient` MUST NOT contain business logic.

## Entity

- Every entity MUST extend `global/entity/BaseEntity`.
- `createdAt` and `updatedAt` MUST be defined only in `BaseEntity`.
- Enable JPA auditing with `@EnableJpaAuditing`.
- Entity table names MUST use plural `snake_case`.
- Entity IDs SHOULD use `{entity}_id` column names.
- Entity no-args constructors MUST use `@NoArgsConstructor(access = AccessLevel.PROTECTED)`.
- Use `@Builder` on constructors, not on entity classes.
- Builder constructors MUST be `private`.
- Entity relationships MUST use `FetchType.LAZY`.
- Do NOT use `FetchType.EAGER`.
- Enums MUST use `@Enumerated(EnumType.STRING)`.
- Do NOT use `EnumType.ORDINAL`.
- Do NOT add `@Setter` to entities.
- Entity state changes MUST use intention-revealing methods.
- Relationship helper methods MUST be written on the owning side entity.

## DTO Conversion

- `XxxRequest` MUST define `toServiceRequest()`.
- Controllers MUST call `request.toServiceRequest()`.
- Controllers MUST NOT instantiate `XxxServiceRequest` directly.
- `XxxServiceResponse` MUST define a static factory method `from(Entity)`.
- Services MUST call `XxxServiceResponse.from(entity)`.
- Services MUST NOT instantiate `XxxServiceResponse` directly.

## Exceptions

- Domain exceptions MUST extend `GlobalException`.
- Domain error codes MUST implement `ErrorCode`.
- Throw domain exceptions with `orElseThrow()`.
- Do NOT call `Optional.get()` directly.
- All API responses MUST use `ApiResponse`.
- Error responses MUST also use `ApiResponse`.
- Do NOT create a separate `ErrorResponse`.
- `ApiResponse` MUST contain only `code`, `message`, and `data`.
- HTTP status codes MUST be handled by `ResponseEntity`, not by `ApiResponse`.
- `GlobalExceptionHandler` MUST be placed under `global/exception`.
- `ApiResponse` MUST be placed under `global/response`.

## Tests

- Service and repository tests MUST extend `IntegrationTestSupport`.
- Controller tests MUST extend `RestControllerTestSupport`.
- `IntegrationTestSupport` MUST use `@SpringBootTest`.
- `IntegrationTestSupport` MUST use `@Transactional`.
- External dependencies such as databases SHOULD use Testcontainers.
- Testcontainers MUST be declared as static singleton containers.
- `RestControllerTestSupport` MUST use `@WebMvcTest`.
- Controller dependencies in MVC tests MUST be mocked with `@MockitoBean`.
- Production code MUST NOT use field injection.
- Test code MAY use `@Autowired` field injection.

## Naming

- Service read methods MUST start with `find`.
- Service create methods MUST start with `create`.
- Service update methods MUST start with `update`.
- Service delete methods MUST start with `delete`.
- Service methods MUST NOT use `get`, `search`, or `fetch` as prefixes.
- Repository method names MUST follow Spring Data JPA naming conventions.

## Dependency Injection

- Production code MUST use constructor injection.
- Use `@RequiredArgsConstructor` with `final` fields.
- Do NOT use `@Autowired` field injection in production code.

## Final Checklist

- Follow Controller -> Service -> Repository dependency direction.
- Do not expose entities outside the service layer.
- Use `toServiceRequest()` and `from()` for DTO conversion.
- Follow service method prefix rules.
- Add domain-specific `XxxException` and `XxxErrorCode`.
- Encapsulate external API calls with `XxxClient`.
- Convert external API exceptions into dedicated exceptions.
- Use `@Transactional` only when the service accesses data.
