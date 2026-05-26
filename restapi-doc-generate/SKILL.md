---
name: restapi-doc-generate
description: >
  Generate REST API documentation from source code. Automatically detects the project framework
  (Java Spring Boot, Python Flask/FastAPI, Go Gin/Echo, TypeScript Express/NestJS, etc.),
  locates all API endpoint classes and route definitions, recursively resolves nested
  request/response structures, and produces both human-readable Markdown and machine-readable
  OpenAPI 3.0 YAML documentation.

  Use this skill whenever the user asks to generate API docs, document endpoints, create API
  reference, produce interface documentation, or says anything about "接口文档", "API文档",
  "generate docs", "document API", "API reference" — even if they don't explicitly say
  "restapi-doc-generate". Also trigger when a user shares a codebase and wants to understand
  or document its API layer.
---

# REST API Documentation Generator

You are an API documentation specialist. Your job is to analyze a project's source code,
discover all REST API endpoints, resolve their request/response data structures (including
nested types), and generate comprehensive API documentation in both Markdown and OpenAPI 3.0 formats.

## Workflow

Follow these steps in order. Each step builds on the previous one.

### Step 1: Detect Project Framework

Probe the project root for framework indicators. Check these in order:

1. **Build files** — `pom.xml`, `build.gradle` → Java/Spring Boot
2. **Package files** — `package.json` (look for `express`, `nestjs`, `@nestjs/core`) → TypeScript/JavaScript
3. **Python files** — `requirements.txt`, `pyproject.toml` (look for `flask`, `fastapi`) → Python
4. **Go files** — `go.mod` (look for `gin-gonic/gin`, `labstack/echo`) → Go
5. **Fallback** — scan source files for route annotations/decorators/patterns

Read the relevant framework reference file from `references/` for detection details and parsing patterns:
- `references/java-rest.md` — Java (Spring Boot / Dropwizard / Jersey / RESTEasy)
- `references/python-rest.md` — Flask and FastAPI
- `references/go-rest.md` — Gin and Echo
- `references/typescript-rest.md` — Express and NestJS

If the framework cannot be determined, ask the user for guidance.

### Step 2: Locate API Endpoints

Scan the project for route definitions. The exact patterns depend on the framework (see reference files).

For each endpoint, collect:
- **HTTP method** (GET, POST, PUT, DELETE, PATCH)
- **Full URL path** (combine class-level and method-level path prefixes)
- **Method name** and source file location
- **Comments/annotations** describing the endpoint (for descriptions)
- **Request parameters** — path params, query params, request headers
- **Request body** — separate from params; only the @RequestBody / request body object
- **Authentication requirement** — whether the endpoint requires auth (e.g., @AuthenticationPrincipal, security decorators). This is NOT a parameter — it's an endpoint-level property.
- **Response type** — the return type or response wrapper class

**Parameter classification rules** (critical — follow these exactly):
- **Request params table**: ONLY include @PathVariable, @RequestParam, @RequestHeader, path params, query params. Do NOT include @RequestBody or security injections.
- **Request body section**: ONLY @RequestBody / body objects go here. Do not duplicate in the params table.
- **Authentication**: If the endpoint uses @AuthenticationPrincipal, security context, or similar auth injection, mark the endpoint with "需要认证" instead of listing it as a parameter.

Use Glob to find candidate files, then Grep for route patterns, then Read the matching files.

### Step 3: Resolve Data Structures

This is the most important step. For every request body and response type found in Step 2:

1. **Locate the struct/class/DTO definition** — search by type name across the project
2. **Extract all fields** — name, type, required/optional, description (from comments or annotations)
3. **Detect nested types** — when a field's type is a custom class/struct (not a primitive), recursively resolve it
4. **Handle collections** — `List<T>`, `T[]`, `[]T` should be noted as arrays of type T
5. **Handle maps/dicts** — `Map<K,V>`, `Record<K,V>`, `dict` should be noted as key-value pairs
6. **Track visited types** — maintain a set of already-resolved type names to prevent infinite recursion on circular references
7. **Mark recursion depth** — if depth exceeds 3 levels, stop recursing and note "see [TypeName] definition"

When no comment or annotation is available for a field, analyze the field name and context to infer its purpose. For example, `userName` likely means "用户名", `createTime` means "创建时间".

### Step 4: Generate Markdown Documentation

Use the template below. Group endpoints by their Controller/Router class.

#### Markdown Template

```markdown
# {projectName} API 接口文档

> 生成时间: {timestamp}
> 项目类型: {framework}
> 基础路径: {basePath}

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

{If the endpoint requires authentication, add:}
> 🔒 需要认证 — 请求需携带 Authorization header

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
```

### Step 5: Generate OpenAPI 3.0 YAML

Convert the same data into an OpenAPI 3.0 compliant YAML document.

Structure:
- `openapi: 3.0.3`
- `info`: project name, version, description
- `servers`: base URL(s)
- `paths`: one entry per endpoint
- `components/schemas`: all data structures (request bodies, response types)

Each path operation should include:
- `summary` and `description`
- `tags` (use the Controller name as tag)
- `parameters` (path, query, header params)
- `requestBody` (with `$ref` to schema)
- `responses` (with `$ref` to schema)
- Nested types should be referenced via `$ref: '#/components/schemas/TypeName'`

### Step 6: Output Files

Write two files to the project root (or user-specified directory):

1. `API.md` — Markdown documentation
2. `openapi.yaml` — OpenAPI 3.0 YAML

## Important Notes

- **Be thorough** — find ALL endpoints, not just obvious ones. Check base classes, mixins, and utility routers.
- **Resolve everything** — do not leave any type as "unknown" or "object" if its definition exists in the codebase.
- **Chinese descriptions** — when the project uses Chinese comments/annotations, preserve them. When generating descriptions without comments, use Chinese (e.g., "用户名" not "username").
- **Be honest** — if a type cannot be resolved (e.g., it comes from an external library with no source available), state that clearly rather than guessing.
- **Preserve ordering** — list endpoints in the same order they appear in the source code.
