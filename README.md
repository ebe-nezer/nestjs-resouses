# 📘 NestJS Complete Cheatsheet

> A comprehensive reference guide for building production-ready NestJS applications. From core concepts to advanced patterns, everything you need in one place.

## 📑 Table of Contents

- [Core Building Blocks](#core-building-blocks)
- [Request Lifecycle](#request-lifecycle)
- [HTTP Route Decorators](#http-route-decorators)
- [Parameter Decorators](#parameter-decorators)
- [Dependency Injection](#dependency-injection)
- [Middleware](#middleware)
- [Guards](#guards)
- [Interceptors](#interceptors)
- [Pipes](#pipes)
- [Exception Filters](#exception-filters)
- [Custom Decorators](#custom-decorators)
- [Validation with DTOs](#validation-with-dtos)
- [Configuration](#configuration)
- [TypeORM Integration](#typeorm-integration)
- [Testing](#testing)
- [CLI Commands](#cli-commands)
- [Common Patterns](#common-patterns)

***

## 🏗️ Core Building Blocks

### Modules

Modules organize your application into cohesive feature blocks.

```typescript
@Module({
  imports: [DatabaseModule, UsersModule],
  controllers: [AppController],
  providers: [AppService],
  exports: [AppService]
})
export class AppModule {}
```


### Controllers

Controllers handle incoming requests and return responses.

```typescript
@Controller('users')
export class UsersController {
  constructor(private usersService: UsersService) {}
  
  @Get()
  findAll() {
    return this.usersService.findAll();
  }
  
  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }
  
  @Post()
  create(@Body() createDto: CreateUserDto) {
    return this.usersService.create(createDto);
  }
  
  @Put(':id')
  update(@Param('id') id: string, @Body() updateDto: UpdateUserDto) {
    return this.usersService.update(id, updateDto);
  }
  
  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(id);
  }
}
```


### Providers (Services)

Providers contain business logic and are injected via dependency injection.

```typescript
@Injectable()
export class UsersService {
  private users = [];
  
  findAll() {
    return this.users;
  }
  
  findOne(id: string) {
    return this.users.find(user => user.id === id);
  }
}
```


***

## 🔄 Request Lifecycle

Understanding the request lifecycle is critical for knowing where to place your logic.

### Complete Execution Order:

1. **Incoming Request** - Client sends HTTP request
2. **Middleware** - Global → Module-bound (in order of binding)
3. **Guards** - Global → Controller → Route level
4. **Interceptors (Before)** - Pre-processing phase
5. **Pipes** - Validation and transformation
6. **Route Handler** - Controller method executes
7. **Interceptors (After)** - Post-processing phase
8. **Exception Filters** - If errors occur
9. **Response** - Sent back to client

***

## 🌐 HTTP Route Decorators

```typescript
@Get('profile')           // GET /users/profile
@Post('create')           // POST /users/create
@Put(':id')              // PUT /users/123
@Patch(':id')            // PATCH /users/123
@Delete(':id')           // DELETE /users/123
@Options('*')            // OPTIONS /users
@Head('info')            // HEAD /users/info
@All('*')                // Any HTTP method
```


***

## 📥 Parameter Decorators

Extract data from incoming requests:

```typescript
@Get(':id')
findOne(
  @Param('id') id: string,              // Route parameters
  @Query('filter') filter: string,       // Query strings
  @Body() createDto: CreateDto,          // Request body
  @Headers('authorization') auth: string, // Headers
  @Req() request: Request,               // Full request object
  @Res() response: Response,             // Full response object
  @Session() session: Record<string, any>, // Session data
  @Ip() ip: string,                      // Client IP address
  @HostParam() host: string              // Host parameter
) {
  return 'Data extracted';
}
```


***

## 💉 Dependency Injection

NestJS uses constructor-based injection for automatic dependency management.

```typescript
// Basic injection
@Injectable()
export class CatsService {
  constructor(private database: DatabaseService) {}
}

// Custom providers
@Module({
  providers: [
    // Class provider
    CatsService,
    
    // Value provider
    { provide: 'CONFIG', useValue: { apiKey: 'abc' } },
    
    // Factory provider
    {
      provide: 'CONNECTION',
      useFactory: (config: ConfigService) => {
        return createConnection(config.get('DB_HOST'));
      },
      inject: [ConfigService]
    },
    
    // Alias provider
    { provide: 'AliasedService', useExisting: CatsService }
  ]
})
```


***

## 🔌 Middleware

Middleware functions execute **before** route handlers.

```typescript
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`${req.method} ${req.path}`);
    next();
  }
}

// Apply in module
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}

// Functional middleware
export function logger(req: Request, res: Response, next: NextFunction) {
  console.log('Request...');
  next();
}
```


***

## 🛡️ Guards

Guards determine if a request should be handled based on certain conditions (authentication, authorization).

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    return this.validateRequest(request);
  }
}

// Usage
@Controller('cats')
@UseGuards(AuthGuard)  // Controller-level
export class CatsController {
  @Get()
  @UseGuards(RolesGuard)  // Route-level
  findAll() {}
}

// Global guard
app.useGlobalGuards(new AuthGuard());
```


***

## 🔄 Interceptors

Interceptors bind extra logic before/after method execution.

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');
    const now = Date.now();
    
    return next.handle().pipe(
      tap(() => console.log(`After... ${Date.now() - now}ms`))
    );
  }
}

// Transform response
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data => ({ data, timestamp: new Date() }))
    );
  }
}

// Usage
@UseInterceptors(LoggingInterceptor)
@Controller('cats')
export class CatsController {}
```


***

## 🔧 Pipes

Pipes transform and validate input data.

```typescript
// Built-in pipes
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('active', ParseBoolPipe) active: boolean
) {}

// Validation pipe with class-validator
export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  name: string;
  
  @IsEmail()
  email: string;
  
  @IsInt()
  @Min(18)
  age: number;
}

@Post()
create(@Body(ValidationPipe) createDto: CreateUserDto) {}

// Custom pipe
@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```


***

## ⚠️ Exception Filters

Handle and format exceptions across your application.

```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    const status = exception.getStatus();
    
    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: exception.message
    });
  }
}

// Usage
@UseFilters(HttpExceptionFilter)
@Controller('cats')
export class CatsController {}
```


***

## 🎨 Custom Decorators

Create reusable decorators for common patterns.

```typescript
// Simple parameter decorator
export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  }
);

// With data extraction
export const UserProperty = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return data ? request.user?.[data] : request.user;
  }
);

// Metadata decorator
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// Composed decorator
export function Auth(...roles: string[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth()
  );
}

// Usage
@Get('profile')
getProfile(@User() user: UserEntity) {}

@Get('admin')
@Auth('admin', 'superuser')
getAdmin() {}
```


***

## ✅ Validation with DTOs

```typescript
// create-user.dto.ts
export class CreateUserDto {
  @IsString()
  @MinLength(3)
  @MaxLength(20)
  username: string;
  
  @IsEmail()
  email: string;
  
  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password too weak'
  })
  password: string;
  
  @IsOptional()
  @IsInt()
  @Min(18)
  @Max(100)
  age?: number;
}

// Enable globally
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,        // Strip non-whitelisted properties
  forbidNonWhitelisted: true, // Throw error if extra props
  transform: true,        // Auto-transform to DTO instances
  transformOptions: {
    enableImplicitConversion: true
  }
}));
```


***

## ⚙️ Configuration

```typescript
// .env file
DATABASE_HOST=localhost
DATABASE_PORT=5432
API_KEY=secret123

// Load configuration
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env'
    })
  ]
})

// Use in service
@Injectable()
export class AppService {
  constructor(private configService: ConfigService) {}
  
  getDatabaseHost() {
    return this.configService.get<string>('DATABASE_HOST');
  }
}
```


***

## 🗄️ TypeORM Integration

```typescript
// Entity
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column()
  name: string;
  
  @Column({ unique: true })
  email: string;
  
  @CreateDateColumn()
  createdAt: Date;
  
  @OneToMany(() => Post, post => post.user)
  posts: Post[];
}

// Repository injection
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>
  ) {}
  
  async findAll(): Promise<User[]> {
    return this.usersRepository.find({ relations: ['posts'] });
  }
  
  async create(user: CreateUserDto): Promise<User> {
    const newUser = this.usersRepository.create(user);
    return this.usersRepository.save(newUser);
  }
}

// Module setup
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      port: 5432,
      username: 'test',
      password: 'test',
      database: 'test',
      entities: [User],
      synchronize: true
    }),
    TypeOrmModule.forFeature([User])
  ],
  providers: [UsersService]
})
```


***

## 🧪 Testing

```typescript
// Unit test
describe('CatsService', () => {
  let service: CatsService;
  
  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CatsService,
        { provide: 'REPOSITORY', useValue: mockRepository }
      ]
    }).compile();
    
    service = module.get<CatsService>(CatsService);
  });
  
  it('should return all cats', () => {
    expect(service.findAll()).toEqual([]);
  });
});

// E2E test
describe('CatsController (e2e)', () => {
  let app: INestApplication;
  
  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule]
    }).compile();
    
    app = moduleFixture.createNestApplication();
    await app.init();
  });
  
  it('/cats (GET)', () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect([]);
  });
});
```


***

## 🛠️ CLI Commands

```bash
# Create new project
nest new project-name

# Generate resources
nest g module users
nest g controller users
nest g service users
nest g resource users  # Creates module, controller, service, DTO

# Other generators
nest g guard auth
nest g interceptor logging
nest g pipe validation
nest g filter http-exception
nest g middleware logger
nest g decorator roles

# Build and run
npm run start          # Development
npm run start:dev      # Watch mode
npm run start:prod     # Production
npm run build          # Build project
```


***

## 🎯 Common Patterns

### Global Prefix

```typescript
app.setGlobalPrefix('api/v1');
```


### CORS

```typescript
app.enableCors({
  origin: 'http://localhost:3000',
  credentials: true
});
```


### Versioning

```typescript
app.enableVersioning({
  type: VersioningType.URI
});

@Controller({ version: '1', path: 'cats' })
```


### Response Headers

```typescript
@Header('Cache-Control', 'none')
@Get()
findAll() {}
```


### Status Codes

```typescript
@HttpCode(201)
@Post()
create() {}
```


### Async Providers

```typescript
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection();
    return connection;
  }
}
```


***

## 📚 Additional Resources

- [Official Documentation](https://docs.nestjs.com)
- [NestJS GitHub](https://github.com/nestjs/nest)
- [Official Courses](https://courses.nestjs.com)
- [Awesome NestJS](https://github.com/juliandavidmr/awesome-nestjs)

***

## 📄 License

This cheatsheet is provided as-is for educational purposes.

***

## 🤝 Contributing

Feel free to submit issues and enhancement requests!
