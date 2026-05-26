# Python — Flask & FastAPI Detection & Parsing Guide

## Detection

Look for:
- `requirements.txt` or `pyproject.toml` containing `flask` or `fastapi`
- Source files importing `flask.Flask`, `fastapi.FastAPI`, `flask.Blueprint`, `fastapi.APIRouter`

## Flask Endpoint Discovery

### Route Patterns

```python
@app.route('/api/users', methods=['GET'])
def list_users():
    pass

@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    pass
```

Also check for:
- `@app.get()`, `@app.post()`, `@app.put()`, `@app.delete()` (Flask 2.0+)
- Blueprint routes: `@bp.route()`
- Method-based views: `MethodView`

### Parameters

- URL parameters: `<int:user_id>`, `<username>` → path params
- `request.args.get('key')` → query params
- `request.get_json()` → request body
- `request.headers.get('Authorization')` → header params

### Response Types

Look for:
- `jsonify()` calls → infer response structure from the dict inside
- `return jsonify({...})`
- Docstring type hints: `:returns: dict with keys ...`

## FastAPI Endpoint Discovery

### Route Patterns

```python
@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    pass

@router.post("/users", response_model=UserResponse)
async def create_user(user: UserCreate):
    pass
```

### Parameters

FastAPI uses type hints extensively:

```python
async def endpoint(
    user_id: int,                      # path param (if in URL)
    name: str = Query(None),           # query param (optional)
    token: str = Header(None),         # header param (optional)
    body: CreateUserRequest = Body()   # request body
):
```

Look for:
- `Path()`, `Query()`, `Header()`, `Body()`, `Cookie()`, `Form()`, `File()`
- Default value `None` or `Optional[T]` → optional parameter
- Pydantic models as parameters → request body

### Response Types

- `response_model=UserResponse` in decorator → response type
- If no `response_model`, check return type annotation
- Pydantic models define the schema

## Data Structure Resolution (Pydantic / Dataclass)

### Pydantic Models

```python
class UserResponse(BaseModel):
    id: int
    name: str
    email: Optional[str] = None
    roles: List[str] = []

    class Config:
        # or model_config in Pydantic v2
        schema_extra = {
            "example": {"id": 1, "name": "admin"}
        }
```

Extract:
- Field name, type, default value
- `Optional[T]` → not required
- No default and not Optional → required
- `Field(description="...")` → field description

### Dataclasses

```python
@dataclass
class CreateUserRequest:
    username: str
    password: str
    email: str = ""  # has default → optional
```

### Nested Types

- `List[UserResponse]` → array of UserResponse
- `Optional[Address]` → nullable Address reference
- `Dict[str, Any]` → free-form object
- Recurse into any custom type (class defined in the project)

## Docstring Extraction

Python docstrings are valuable for descriptions:

```python
def get_user(user_id: int):
    """获取用户信息

    Args:
        user_id: 用户ID

    Returns:
        UserResponse: 用户详细信息
    """
```

Parse Google-style, NumPy-style, or reST-style docstrings for parameter and return descriptions.
