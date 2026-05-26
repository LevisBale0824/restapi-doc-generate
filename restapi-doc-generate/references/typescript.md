# TypeScript — Express & NestJS Detection & Parsing Guide

## Detection

Look for:
- `package.json` containing `express`, `nestjs`, `@nestjs/core`, `@nestjs/common`
- Source files importing `express`, `@nestjs/common`

## Express Endpoint Discovery

### Route Patterns

```typescript
// Direct app routes
app.get('/users', getUsers);
app.post('/users', createUser);
app.put('/users/:id', updateUser);
app.delete('/users/:id', deleteUser);

// Router
const router = express.Router();
router.get('/', listUsers);
router.post('/', createUser);
app.use('/api/users', router);
```

Look for:
- `app.get()`, `app.post()`, `app.put()`, `app.delete()`, `app.patch()`
- `router.get()`, `router.post()`, etc.
- `app.use('/path', router)` — mount point (prefix)

### Parameters

Express doesn't have built-in parameter typing. Infer from:
- `req.params.id` → path param
- `req.query.name` → query param
- `req.headers['authorization']` → header
- `req.body` → request body (look for body-parser middleware)

### Response Types

Look for:
- `res.json(data)` → response structure
- `res.status(200).json({...})` → inline response
- Type assertions: `res.json(data as UserResponse)`

## NestJS Endpoint Discovery

### Decorator Patterns

```typescript
@Controller('users')
export class UsersController {

  @Get()
  findAll(): UserResponse[] {}

  @Get(':id')
  findOne(@Param('id') id: string): UserResponse {}

  @Post()
  create(@Body() dto: CreateUserDto): UserResponse {}

  @Put(':id')
  update(@Param('id') id: string, @Body() dto: UpdateUserDto): UserResponse {}

  @Delete(':id')
  remove(@Param('id') id: string): void {}
}
```

### Parameter Decorators

| Decorator | Source |
|-----------|--------|
| `@Param('id')` | path parameter |
| `@Query('name')` | query parameter |
| `@Body()` | request body |
| `@Headers('auth')` | header |
| `@Req()` | full request object |

### Response Types

- Method return type annotation: `findOne(): UserResponse`
- `@ApiResponse({ type: UserResponse })` — Swagger decorator
- `@HttpStatus(201)` — response status

## Data Structure Resolution

### Interface / Type Definitions

```typescript
interface CreateUserRequest {
  username: string;
  password: string;
  email?: string; // optional
  roles: string[];
  profile: UserProfile; // nested type
}
```

Extract:
- Field name, type
- `?` optional marker → not required
- Array types: `string[]`, `number[]`, `Type[]`
- Union types: `string | null` → nullable

### Class DTOs (NestJS)

```typescript
export class CreateUserDto {
  @ApiProperty({ description: '用户名' })
  @IsString()
  @IsNotEmpty()
  username: string;

  @ApiProperty({ description: '密码', required: false })
  password?: string;
}
```

Extract from decorators:
- `@ApiProperty({ description: '...' })` → field description
- `@IsNotEmpty()`, `@IsDefined()` → required
- `@IsOptional()` → not required
- `@IsString()`, `@IsNumber()`, `@IsEmail()` → type hints

### Nested Types

- Custom interfaces/types/classes → recurse
- `Array<T>` or `T[]` → array of T
- `Record<string, T>` → map/object
- `Partial<T>`, `Omit<T, K>`, `Pick<T, K>` → transform T accordingly
- Standard types to NOT recurse: `string`, `number`, `boolean`, `Date`, `any`, `unknown`

## JSDoc / TSDoc Extraction

```typescript
/**
 * 用户登录接口
 * @param dto 登录请求
 * @returns 登录响应，包含 token
 */
@Post('login')
login(@Body() dto: LoginDto): LoginResponse {}
```

Parse JSDoc tags for descriptions: `@param`, `@returns`, `@throws`, `@deprecated`.
