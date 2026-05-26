# Go — Gin & Echo Detection & Parsing Guide

## Detection

Look for:
- `go.mod` containing `gin-gonic/gin` or `labstack/echo`
- Source files importing `gin` or `echo` packages

## Gin Endpoint Discovery

### Route Registration

```go
r := gin.Default()
r.GET("/users", listUsers)
r.POST("/users", createUser)
r.GET("/users/:id", getUser)
r.PUT("/users/:id", updateUser)
r.DELETE("/users/:id", deleteUser)

// Group routes
api := r.Group("/api/v1")
{
    api.GET("/users", listUsers)
    api.POST("/users", createUser)
}
```

Look for:
- `r.GET()`, `r.POST()`, `r.PUT()`, `r.DELETE()`, `r.PATCH()`
- `r.Group()` — path prefix for a set of routes
- Handler function name and source location

### Parameters

- Path params: `c.Param("id")` → from URL `:id`
- Query params: `c.Query("name")` → `?name=value`
- Header: `c.GetHeader("Authorization")`
- Body: `c.ShouldBindJSON(&req)` or `c.BindJSON(&req)` → request body

### Response Types

Look for handler functions that:
- `c.JSON(code, data)` → response structure
- `c.JSON(http.StatusOK, gin.H{...})` → anonymous response (infer from gin.H)
- `c.JSON(http.StatusOK, response)` → named response type

## Echo Endpoint Discovery

### Route Registration

```go
e := echo.New()
e.GET("/users", listUsers)
e.POST("/users", createUser)
e.GET("/users/:id", getUser)

// Group routes
api := e.Group("/api/v1")
api.GET("/users", listUsers)
```

Look for:
- `e.GET()`, `e.POST()`, `e.PUT()`, `e.DELETE()`
- `e.Group()` — path prefix
- `echo.HandlerFunc` implementations

### Parameters

- Path params: `c.Param("id")`
- Query params: `c.QueryParam("name")`
- Header: `c.Request().Header.Get("Authorization")`
- Body: `c.Bind(&req)` → request body

## Data Structure Resolution

### Struct Definitions

```go
type CreateUserRequest struct {
    Username string `json:"username" binding:"required"`
    Password string `json:"password" binding:"required"`
    Email    string `json:"email"`
}
```

Extract from struct tags:
- `json:"name"` → field name in JSON
- `binding:"required"` → required field
- `validate:"required"` → required field (if using validator)
- `form:"name"` → form field name
- `query:"name"` → query parameter name
- `uri:"name"` → URI parameter name

### Nested Types

```go
type UserResponse struct {
    ID    int        `json:"id"`
    Name  string     `json:"name"`
    Role  RoleInfo   `json:"role"`
    Tags  []string   `json:"tags"`
}
```

- Custom types (not `string`, `int`, `bool`, `float64`, etc.) → recurse
- `[]Type` → array of Type
- `*Type` → pointer to Type (nullable)
- `map[string]interface{}` → free-form object

### Anonymous Structs

```go
c.JSON(200, gin.H{
    "code": 0,
    "data": user,
})
```

When `gin.H{}` is used, look at the literal keys and values to infer structure.

## Comment Extraction

Go comments above handler functions and struct definitions:

```go
// GetUser retrieves a user by ID.
// 获取用户信息
func getUser(c *gin.Context) {
```

```go
// UserResponse represents user information.
type UserResponse struct {
    // ID is the unique identifier
    ID int `json:"id"`
}
```
