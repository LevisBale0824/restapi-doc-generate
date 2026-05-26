# Java Spring Boot — Detection & Parsing Guide

## Detection

Look for these files in the project root:
- `pom.xml` containing `spring-boot` or `spring-web`
- `build.gradle` / `build.gradle.kts` containing `spring-boot` or `spring-web`
- Source files with `@RestController`, `@Controller` annotations

## Endpoint Discovery

### Route Annotations

Scan for these annotation patterns:

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

## Data Structure Resolution

### Locating DTOs

Search for class definitions by name:
```
grep -r "class LoginRequest" --include="*.java"
```

Common DTO naming patterns:
- `*Request`, `*DTO`, `*VO`, `*Command`, `*Query`
- `*Response`, `*Result`, `*Model`, `*Entity`

### Field Extraction

For each field in a class:
```java
@ApiModelProperty("用户名")
@NotNull
private String userName;
```

Extract:
- Field name: `userName`
- Type: `String`
- Required: yes (from `@NotNull`)
- Description: "用户名" (from `@ApiModelProperty`)

Common validation annotations indicating required fields:
- `@NotNull`, `@NotEmpty`, `@NotBlank`, `@Valid`

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

Standard types to NOT recurse into: `String`, `Integer`, `Long`, `Boolean`, `Double`, `Float`, `BigDecimal`, `Date`, `LocalDate`, `LocalDateTime`, `List`, `Map`, `Set`

### Inheritance

Check for `extends` keyword. If a DTO extends another, include parent fields too.
