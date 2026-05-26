# REST API Doc Generate

A Claude Code skill that automatically generates REST API documentation from source code.

## Features

- **Auto-detect project framework** — Java Spring Boot, Python Flask/FastAPI, Go Gin/Echo, TypeScript Express/NestJS
- **Locate all API endpoints** — scans controllers, routers, decorators, annotations
- **Recursive data structure resolution** — follows nested DTOs, records, structs, including cross-module references
- **Dual output format** — generates both Markdown and OpenAPI 3.0 YAML
- **Smart descriptions** — extracts from code comments/annotations first, falls back to name analysis

## Installation

Copy the `restapi-doc-generate/` directory to your Claude Code skills directory:

```bash
cp -r restapi-doc-generate/ ~/.claude/skills/
```

Or use with Claude Code's skill system directly by placing it in your project.

## Usage

In Claude Code, simply ask:

```
帮我生成这个项目的 API 接口文档
```

```
Generate API docs for this project
```

The skill will:
1. Detect your project's framework
2. Find all API endpoints
3. Resolve request/response data structures (including nested types)
4. Generate `API.md` and `openapi.yaml` in your project root

## Output Example

The generated Markdown includes:
- Table of contents with anchor links
- Endpoints grouped by Controller/Router
- Request parameters, request body, and response structures
- Nested data structures with reference links
- Enum definitions
- Unified error code section

The generated OpenAPI 3.0 YAML can be imported into Swagger Editor, Postman, or any OpenAPI-compatible tool.

## Supported Frameworks

| Language | Framework | Detection |
|----------|-----------|-----------|
| Java | Spring Boot | `@RestController`, `@RequestMapping` |
| Python | Flask | `@app.route()`, Blueprints |
| Python | FastAPI | `@router.get()`, Pydantic models |
| Go | Gin | `r.GET()`, `r.Group()` |
| Go | Echo | `e.GET()`, `e.Group()` |
| TypeScript | Express | `router.get()`, `app.use()` |
| TypeScript | NestJS | `@Controller()`, `@Get()` |

## Structure

```
restapi-doc-generate/
├── SKILL.md                      # Core instructions & template
└── references/
    ├── java-springboot.md        # Java Spring Boot parsing guide
    ├── python.md                 # Flask + FastAPI parsing guide
    ├── go.md                     # Gin + Echo parsing guide
    └── typescript.md             # Express + NestJS parsing guide
```

## License

MIT
