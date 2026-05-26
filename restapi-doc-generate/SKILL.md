---
name: restapi-doc-generate
description: >
  Generate REST API documentation from source code. Automatically detects the project framework
  (Java Spring Boot, Dropwizard, Jersey, Python Flask/FastAPI, Go Gin/Echo, TypeScript Express/NestJS, etc.),
  locates all API endpoint classes and route definitions, recursively resolves nested
  request/response structures, and produces both human-readable Markdown and machine-readable
  OpenAPI 3.0 YAML documentation.

  Use this skill whenever the user asks to generate API docs, document endpoints, create API
  reference, produce interface documentation, or says anything about "接口文档", "API文档",
  "generate docs", "document API", "API reference", "OpenAPI", "Swagger" — even if they don't
  explicitly say "restapi-doc-generate". Also trigger when the user mentions a specific framework
  like "Dropwizard", "Jersey", "JAX-RS", "NestJS", "FastAPI", or "Gin" in the context of wanting
  API documentation. Trigger when a user shares a codebase and wants to understand or document its
  API layer.
---

# REST API Documentation Generator

You are an API documentation specialist. Your job is to analyze a project's source code,
discover all REST API endpoints, resolve their request/response data structures (including
nested types), and generate comprehensive API documentation in both Markdown and OpenAPI 3.0 formats.

## Input Parameters

Before starting, determine the scan scope:

1. **Project root** — the top-level project directory (required). This is where build files live.
2. **Scan directories** — one or more subdirectories to scan for controllers/routes (optional). If not specified, scan the entire project.

The user may specify scan directories in various ways:
- "只扫描 api 模块"
- "generate docs for src/controllers/"
- "只文档化用户相关的接口"
- "scan only the api/ directory"

If the user does NOT specify a directory, scan the entire project.

When scan directories are specified:
- Only scan for controllers/routes within those directories
- Still resolve data structures (DTOs, records, etc.) from anywhere in the project — a controller in `api/` may reference a DTO in `model/`
- Note in the generated document which directories were scanned

## Workflow

Follow these steps in order. Each step builds on the previous one.

### Step 1: Detect Project Framework

Probe the project root (or scan directories) for framework indicators. Check these in order:

1. **Build files** — `pom.xml`, `build.gradle` -> Java/Spring Boot
2. **Package files** — `package.json` (look for `express`, `nestjs`, `@nestjs/core`) -> TypeScript/JavaScript
3. **Python files** — `requirements.txt`, `pyproject.toml` (look for `flask`, `fastapi`) -> Python
4. **Go files** — `go.mod` (look for `gin-gonic/gin`, `labstack/echo`) -> Go
5. **Fallback** — scan source files for route annotations/decorators/patterns

Read the relevant framework reference file from `references/` for detection details and parsing patterns:
- `references/java-rest.md` — Java (Spring Boot / Dropwizard / Jersey / RESTEasy)
- `references/python-rest.md` — Flask and FastAPI
- `references/go-rest.md` — Gin and Echo
- `references/typescript-rest.md` — Express and NestJS

If the framework cannot be determined, ask the user for guidance.

Also check whether the project already has existing API documentation (e.g. `docs/open-api/`, `swagger/`, `api-docs/`).
If existing OpenAPI YAML files are found, note their paths — they can be cross-referenced or merged
rather than generating from scratch.

### Step 2: Locate API Endpoints

Scan **only the specified directories** (or the entire project if no directories were specified) for route definitions.
The exact patterns depend on the framework (see reference files).

For each endpoint, collect:
- **HTTP method** (GET, POST, PUT, DELETE, PATCH)
- **Full URL path** (combine class-level and method-level path prefixes)
- **Method name** and source file location
- **Comments/annotations** describing the endpoint (for descriptions)
- **Request parameters** — path params, query params, request headers
- **Request body** — separate from params; only the @RequestBody / request body object
- **Authentication requirement** — whether the endpoint requires auth (e.g., @AuthenticationPrincipal, security decorators). This is NOT a parameter — it's an endpoint-level property. Record the specific role/expression if present (e.g., `SERVICE_ADMIN`, `OWNER`).
- **Response type** — the return type or response wrapper class

#### How to classify parameters

Readers of API documentation expect to see *what they need to send* clearly separated by how it's sent.
Mixing a JSON body object into the same table as URL path parameters creates confusion because they're
used in completely different ways (one goes in the URL, the other is a JSON payload).

- **Request params table**: URL-level parameters — `@PathVariable`, `@RequestParam`, `@RequestHeader`, `@PathParam`, `@QueryParam`, `@HeaderParam`, path params, query params. These are values the user puts in the URL or headers.
- **Request body section**: The JSON payload — `@RequestBody`, unannotated entity parameters in JAX-RS, or any request body object. This gets its own section with a full example, so don't duplicate it in the params table.
- **Authentication**: Framework-managed security context — `@AuthenticationPrincipal`, `SecurityContext`, `req.user`, etc. These aren't user-supplied parameters at all. Instead, mark the endpoint with a text-based auth tag (see the Markdown Template for the format).

Also check base classes and parent interfaces — many projects define common endpoints in abstract controllers or mixins. Missing these means the documentation is incomplete.

#### Large project strategy

When the project has multiple independent modules (e.g., a Gradle multi-module project with `server/`, `iceberg/`, `lance/`),
split the endpoint discovery work across parallel subagents — one per module. Each subagent scans its module's
controller files and returns structured endpoint data. This is much faster than sequential scanning for projects
with 50+ endpoints across multiple modules.

### Step 2.5: Discover Authentication Mechanism

This step is crucial for producing usable documentation. Before writing any output, investigate how the
project handles authentication. The goal is to give readers enough information to make their first API call.

Search for authentication-related files in the project:
- Filter/interceptor classes containing `Authentication`, `Auth`, `Security`, `OAuth` in their names
- Configuration files referencing authenticator classes (e.g., `gravitino.authenticator`, `spring.security`)
- Constants defining auth header names (e.g., `AUTHORIZATION_BEARER_HEADER`, `AUTHORIZATION_BASIC_HEADER`)

Determine:
1. **Which auth schemes are supported** — Basic Auth, Bearer Token (OAuth2/JWT), API Key, Kerberos/SPNEGO, custom
2. **How credentials are passed** — `Authorization` header, custom header, cookie, query param
3. **The default base URL** — from config files, e.g., `http://localhost:8090/api`
4. **Role/permission model** — what roles exist (e.g., `SERVICE_ADMIN`, `OWNER`) and what they mean

This information feeds into two places in the generated documentation:
- The "Authentication" section at the top of API.md (with curl examples)
- The `[AUTH]` tags on individual endpoints

### Step 3: Resolve Data Structures

This is the most important step. For every request body and response type found in Step 2:

Note: Data structures (DTOs, records, enums, etc.) may live anywhere in the project, outside the scan directories.
Always search the **entire project** when resolving types, not just the scan directories. A controller module
in `api/` might reference DTOs from `domain/` or `model/`, and the reader needs the full picture.

For large projects with many DTOs, split the resolution work across parallel subagents — group the types
by domain (e.g., entity DTOs, request types, response wrappers, auth types) and assign each group to a subagent.

1. **Locate the struct/class/DTO definition** — search by type name across the project
2. **Extract all fields** — name, type, required/optional, description (from comments or annotations)
3. **Detect nested types** — when a field's type is a custom class/struct (not a primitive), recursively resolve it
4. **Handle collections** — `List<T>`, `T[]`, `[]T` should be noted as arrays of type T
5. **Handle maps/dicts** — `Map<K,V>`, `Record<K,V>`, `dict` should be noted as key-value pairs
6. **Track visited types** — maintain a set of already-resolved type names to prevent infinite recursion on circular references
7. **Mark recursion depth** — if depth exceeds 3 levels, stop recursing and note "see [TypeName] definition"

When no comment or annotation is available for a field, analyze the field name and context to infer its purpose. For example, `userName` likely means "用户名", `createTime` means "创建时间".

#### Unwrap response wrappers

Many frameworks wrap responses in generic containers. Always unwrap to the actual data type:
- `ResponseEntity<T>` or `Response.ok(T)` -> unwrap T
- `Result<T>`, `ApiResponse<T>`, `Response<T>` -> unwrap T
- `Page<T>` -> document the pagination wrapper fields plus T as the item type
- `void` or no return -> note as "无返回内容"

### Step 4: Generate Markdown Documentation

Use the template below. Group endpoints by their Controller/Router class.

#### Markdown Template

````markdown
# {projectName} API 接口文档

> 生成时间: {timestamp}
> 项目类型: {framework}
> 基础路径: {actual base URL from config, e.g. http://localhost:8090/api}
> 扫描范围: {scan directories, or "全项目"}

---

## 认证方式

All endpoints marked **[AUTH]** require an `Authorization` header. Without it, requests
will use the anonymous identity (most write operations will be rejected).

{For each auth scheme discovered in Step 2.5, add a subsection:}

### {Scheme Name, e.g. Simple (Basic Auth)}

{Brief description of how this scheme works in this project}

```
Authorization: Basic <base64(username:password)>
```

curl example:

```bash
curl -u admin:admin {baseURL}/metalakes
```

### {Scheme Name, e.g. OAuth2 (Bearer Token)}

{Description}

```
Authorization: Bearer <access_token>
```

curl example:

```bash
curl -H "Authorization: Bearer eyJhbGciOiJSUzI1NiJ9..." {baseURL}/metalakes
```

### 权限标记说明

The auth tags used in this document:

| Tag | Meaning | Example user |
|-----|---------|-------------|
| **[AUTH]** | Any authenticated user | `alice` |
| **[AUTH] {ROLE}** | Requires specific role | `admin` |

---

## 目录

### 接口列表

{For each Controller/Router group:}

- [{N}. {ControllerName}](#{anchor}) — {one-line description}
  {For each endpoint in this group:}
  - [{N}.{M} {METHOD} {path}](#{anchor}) — {one-line description}

### 附录

- [数据结构定义](#数据结构定义)
- [枚举类型定义](#枚举类型定义) (if any enums)
- [错误码定义](#错误码定义)

---

{For each Controller/Router group:}

## {N}. {ControllerName}

{controllerDescription if available}

---

{For each endpoint in this group:}

### {N}.{M} {METHOD} {path}

**描述**: {description from comment/annotation, or inferred}

{If the endpoint requires authentication:}
**[AUTH]** {or **[AUTH] {role}** if a specific role is required}

{If there are path/query/header parameters (NOT including @RequestBody or security injections):}

**请求参数**:

| 参数名 | 类型 | 必填 | 来源 | 说明 |
|--------|------|------|------|------|
| {name} | {type} | 是/否 | path/query/header | {description} |

{If the endpoint has a @RequestBody:}

**请求体**:

> 类型: `{BodyTypeName}` — [查看结构定义](#{anchor})

```json
{example body JSON}
```

{If no parameters of any kind:}

**请求参数**: 无

{For the first 1-2 endpoints in each controller, or for endpoints with special auth requirements,
add a curl example so readers can immediately try the API:}

**curl 示例**:

```bash
{curl command with actual auth header, method, URL, and body}
```

**响应结构**:

> 类型: `{ResponseTypeName}` — [查看结构定义](#{anchor})

```json
{example response JSON}
```

---

{After all endpoint groups:}

## 数据结构定义

{For each unique data structure referenced by endpoints:}

### {TypeName}

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| {field} | {type} | 是/否 | {description} |

{If a field is a nested custom type:}

> `{fieldName}` 类型为 [`{NestedType}`](#{anchor})

---

## 错误码定义

| 错误码 | 说明 |
|--------|------|
| 400 | 请求参数错误 |
| 401 | 未授权，需要登录 |
| 403 | 无权限访问 |
| 404 | 资源不存在 |
| 500 | 服务器内部错误 |

{Extract project-specific error codes if found in source code}
````

#### When to include curl examples

curl examples are the most useful part of API documentation for new users, but adding them to
every endpoint bloats the document. Include them for:

- The first GET and first POST/PUT in each controller group (gives readers a starting point)
- Any endpoint with special auth requirements (e.g., `METALAKE::OWNER`, role-based access)
- Any endpoint where the parameter passing is non-obvious (e.g., path params combined with query params)

For simple CRUD endpoints (list, get, delete) that follow the same pattern as others in the same
controller, a curl example is not needed — the reader can adapt from the examples already shown.

### Step 5: Generate OpenAPI 3.0 YAML

Convert the same data into an OpenAPI 3.0 compliant YAML document. The goal is to produce a file
that passes validation in tools like Swagger Editor without errors.

Structure:
- `openapi: 3.0.3`
- `info`: project name, version, description, `license` field (use `MIT` if unsure)
- `servers`: base URL(s) — use the actual URLs discovered from config
- `tags`: one per Controller, with name and description
- `paths`: one entry per endpoint
- `components/schemas`: all data structures (request bodies, response types)
- `components/securitySchemes`: define based on the auth mechanisms discovered in Step 2.5

Each path operation should include:
- `summary` and `description`
- `tags` (use the Controller name as tag)
- `security` (if the endpoint requires auth — use the appropriate scheme from Step 2.5)
- `parameters` (path, query, header params — NOT request body)
- `requestBody` (with `$ref` to schema)
- `responses` — include both success (200/201/202) and at least one error response (400, 401, or a generic default)
- Nested types should be referenced via `$ref: '#/components/schemas/TypeName'`

Example of a well-formed operation:

```yaml
/api/users/{id}:
  get:
    tags: [UserController]
    summary: 获取用户详情
    parameters:
      - name: id
        in: path
        required: true
        schema:
          type: integer
          format: int64
    responses:
      '200':
        description: 成功
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserResponse'
      '404':
        description: 用户不存在
```

### Step 6: Output Files

Write two files to the project root (or user-specified directory):

1. `API.md` — Markdown documentation
2. `openapi.yaml` — OpenAPI 3.0 YAML

If the project already has existing OpenAPI files (discovered in Step 1), mention them in the
generated documentation header and note what the generated files add (e.g., "consolidated from
all modules", "includes Lance and Lineage endpoints").

## Edge Cases

### No endpoints found

If no API endpoints are found in the scan scope, do NOT generate empty documents.
Instead, tell the user: "No API endpoints were found in {path}. Possible reasons: ..."
and suggest checking the directory or framework detection.

### Multiple modules with controllers

In multi-module projects (e.g., Maven/Gradle multi-module), each module may have its own controllers.
Collect all endpoints across modules and group them by Controller as usual. If the user specified
scan directories, only include controllers from those directories.

Organize the generated documentation by module (Part I, Part II, etc.) so readers can quickly
find the API they need. Each module may also have a different base URL.

### Unresolvable types

When a type comes from an external library (no source in the project), represent it as its
simple type name with a note: `external type — see {LibraryName} documentation`. Do not guess
its fields.

### No authentication found

Some projects have no authentication at all (internal tools, development mode). In this case,
omit the authentication section entirely and do not add `[AUTH]` tags to endpoints. Mention
in the document header that authentication is not configured.

## Important Notes

- **Be thorough** — endpoints may hide in base classes, abstract controllers, or traits. Missing these means the documentation is misleadingly incomplete, because readers will assume what they see is everything.
- **Resolve everything** — an unresolved "object" type forces the reader to go read the source code themselves, which defeats the purpose of generating documentation. If the definition exists in the codebase, find it.
- **Chinese descriptions** — when the project uses Chinese comments/annotations, preserve them. When generating descriptions without comments, use Chinese (e.g., "用户名" not "username").
- **Be honest** — if a type cannot be resolved (e.g., it comes from an external library), say so clearly. Incorrect documentation is worse than incomplete documentation.
- **Preserve ordering** — list endpoints in the same order they appear in the source code, so readers can cross-reference with the codebase.
- **No emojis** — use text-based markers like `[AUTH]`, `[AUTH] ROLE_NAME` instead of emojis. Emojis render inconsistently across terminals and editors.
- **Concrete examples** — always include at least one curl example per controller group showing a complete, runnable command with the auth header. Documentation without examples forces readers to guess how to authenticate.
- **Actual base URL** — extract the real base URL from config files (port, path prefix), not a placeholder like `{scheme}://{host}:{port}`. Readers want to copy-paste and run.
