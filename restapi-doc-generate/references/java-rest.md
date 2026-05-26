# Java — REST API Detection & Parsing Guide

Java has two major annotation styles for REST APIs. Detect which one the project uses, then apply the corresponding parsing rules.

## Detection

### Step 1: Identify annotation style

| Style | Key annotations | Frameworks |
|-------|----------------|------------|
| **Spring Web** | `@RestController`, `@RequestMapping`, `@GetMapping` | Spring Boot, Spring MVC |
| **JAX-RS** | `@Path`, `@GET`, `@POST`, `@Produces` | Dropwizard, Jersey, RESTEasy, CXF |

### Step 2: Detect framework from build files

**Spring Boot:**
- `pom.xml` containing `spring-boot` or `spring-web`
- `build.gradle` / `build.gradle.kts` containing `spring-boot` or `spring-web`

**JAX-RS / Dropwizard:**
- `pom.xml` containing `dropwizard`, `jersey`, `resteasy`, `cxf`
- `build.gradle` containing `io.dropwizard`, `org.glassfish.jersey`
- `config.yml` / `*.yml` with `server` block containing `applicationContextPath` (Dropwizard)
- Source files importing `javax.ws.rs.*` or `jakarta.ws.rs.*`

**If both styles are present** (e.g., Spring Boot with Jersey servlet), prefer the annotation style actually used on the resource classes.

---

## Style A: Spring Web (Spring Boot / Spring MVC)

### Route Annotations

| Annotation | Level | Example |
|-----------|-------|---------|
| `@RequestMapping` | Class/Method | `@RequestMapping("/api/users")` |
| `@GetMapping` | Method | `@GetMapping("/{id}")` |
| `@PostMapping` | Method | `@PostMapping("/login")` |
| `@PutMapping` | Method | `@PutMapping("/{id}")` |
| `@DeleteMapping` | Method | `@DeleteMapping("/{id}")` |
| `@PatchMapping` | Method | `@PatchMapping("/{id}")` |

**Path resolution**: Combine class-level `@RequestMapping` prefix with method-level path.
Example: class `@RequestMapping("/api/users")` + method `@GetMapping("/{id}")` = `GET /api/users/{id}`

### Parameter Annotations

| Annotation | Source | Example |
|-----------|--------|---------|
| `@PathVariable` | path | `@PathVariable Long id` |
| `@RequestParam` | query | `@RequestParam String name` |
| `@RequestHeader` | header | `@RequestHeader String token` |
| `@RequestBody` | body | `@RequestBody LoginRequest req` |
| `@CookieValue` | cookie | `@CookieValue String sessionId` |

Check for `required = false` or `Optional<T>` to determine if parameter is optional.

### Response Types

Look at the method's return type:
- `ResponseEntity<T>` → unwrap T
- `Result<T>` / `R<T>` / `ApiResponse<T>` → custom wrapper, unwrap T
- Direct object → that's the response type

Check for `@ResponseStatus` on the method or exception handler.

---

## Style B: JAX-RS (Dropwizard / Jersey / RESTEasy)

### Route Annotations

| Annotation | Level | Example |
|-----------|-------|---------|
| `@Path` | Class/Method | `@Path("/api/users")` |
| `@GET` | Method | `@GET` |
| `@POST` | Method | `@POST` |
| `@PUT` | Method | `@PUT` |
| `@DELETE` | Method | `@DELETE` |
| `@PATCH` | Method | `@PATCH` |
| `@HEAD` | Method | `@HEAD` |
| `@OPTIONS` | Method | `@OPTIONS` |

**Path resolution**: Combine class-level `@Path` prefix with method-level `@Path`.
Example: class `@Path("/api/users")` + method `@Path("/{id}")` + `@GET` = `GET /api/users/{id}`

Note: `@GET`/`@POST` etc. do NOT take a path argument. The path comes from `@Path` on the method.

**Content type annotations**:
- `@Produces("application/json")` — response content type
- `@Consumes("application/json")` — request body content type

### Parameter Annotations

| Annotation | Source | Example |
|-----------|--------|---------|
| `@PathParam` | path | `@PathParam("id") Long id` |
| `@QueryParam` | query | `@QueryParam("name") String name` |
| `@HeaderParam` | header | `@HeaderParam("Authorization") String token` |
| `@FormParam` | form | `@FormParam("username") String username` |
| `@CookieParam` | cookie | `@CookieParam("sessionId") String sessionId` |
| (unannotated entity param) | body | `LoginRequest req` (no annotation, just the type) |
| `@BeanParam` | multiple | `@BeanParam UserFilter filter` (combines multiple params) |

**Body parameter rule**: In JAX-RS, the request body is a method parameter with NO parameter annotation. There can be at most one such parameter per method.

**Optional parameters**: Use `@DefaultValue` or wrap with `Optional<T>`:
```java
@QueryParam("page") @DefaultValue("1") int page
@QueryParam("search") Optional<String> search
```

### Response Types

Look for:
- `Response` (javax.ws.rs.core.Response) → inspect `entity` or the object returned by `.entity(data).build()`
- `ResponseEntity<T>` (if using Spring bridge) → unwrap T
- Direct POJO return → that's the response type (auto-serialized to JSON by Jackson)

**Dropwizard-specific patterns**:
```java
// Common Dropwizard response pattern
public Response getUser(@PathParam("id") Long id) {
    User user = userDao.findById(id);
    if (user == null) {
        return Response.status(Response.Status.NOT_FOUND).build();
    }
    return Response.ok(user).build();  // response type is User
}
```

Also check for `@ExceptionMapper` classes that define error responses.

### Dropwizard Configuration

Look for `*.yml` / `*.yaml` config files for:
- `server.applicationContextPath` — URL prefix (e.g., `/api`)
- `server.rootPath` — servlet path (e.g., `/*`)
- These affect the full URL path and should be prepended to all routes.

---

## Shared: Data Structure Resolution

Applies to both Spring Web and JAX-RS styles.

### Locating DTOs

Search for class/record definitions by name. Common locations:
- `dto/`, `model/`, `entity/`, `request/`, `response/`, `resource/`, `representation/` (Dropwizard convention)

Common DTO naming patterns:
- `*Request`, `*DTO`, `*VO`, `*Command`, `*Query`, `*Representation` (Dropwizard)
- `*Response`, `*Result`, `*Model`, `*Entity`, `*Info`

### Field Extraction

For each field in a class:
```java
@ApiModelProperty("用户名")
@NotNull
@JsonProperty("user_name")  // Jackson annotation — the actual JSON field name
private String userName;
```

Extract:
- Field name: use `@JsonProperty("name")` if present, otherwise the Java field name
- Type: the Java type
- Required: yes (from `@NotNull`)
- Description: from `@ApiModelProperty`, `@Schema`, or Javadoc

**Important: Jackson annotations** (used by both Spring Boot and Dropwizard):
- `@JsonProperty("custom_name")` — the JSON field name may differ from Java field name
- `@JsonIgnore` — field is excluded from serialization
- `@JsonInclude(Include.NON_NULL)` — nullable fields may be omitted from response
- `@JsonFormat(pattern = "yyyy-MM-dd")` — date format hint

Common validation annotations indicating required fields:
- `@NotNull`, `@NotEmpty`, `@NotBlank`, `@Valid`
- JAX-RS Bean Validation: same annotations via `javax.validation` / `jakarta.validation`

Common documentation annotations:
- `@ApiModelProperty("desc")` — Swagger 2
- `@Schema(description = "desc")` — OpenAPI 3 / SpringDoc
- `@ApiParam("desc")` — Swagger 2 (for parameters)
- Javadoc comments above the field

### Nested Types

When a field's type is not a Java primitive or standard library type:
1. Search the project for its class definition
2. Recursively extract its fields
3. Mark the relationship in the documentation

Standard types to NOT recurse into: `String`, `Integer`, `Long`, `Boolean`, `Double`, `Float`, `BigDecimal`, `Date`, `LocalDate`, `LocalDateTime`, `Instant`, `List`, `Map`, `Set`, `Optional`

### Inheritance

Check for `extends` keyword. If a DTO extends another, include parent fields too.

Also check for Jackson polymorphic handling:
- `@JsonTypeInfo` + `@JsonSubTypes` — indicates a type hierarchy with discriminator
- Document all subtypes and their additional fields
