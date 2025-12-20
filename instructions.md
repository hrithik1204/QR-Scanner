# Backend Development Instructions for QR Inventory Tracking System

## Project Context
This is a NestJS backend for a QR-based inventory tracking system. The backend must be completely decoupled from frontend and serve as a platform-agnostic API service for both web and future mobile applications.

---

## Technology Stack
- **Runtime**: Node.js (v18+)
- **Framework**: NestJS
- **Language**: TypeScript (strict mode)
- **Database**: PostgreSQL
- **ORM**: Prisma
- **Authentication**: JWT (access + refresh tokens)
- **Authorization**: NestJS Guards with Role-Based Access Control

---

## Project Structure

```
backend/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── auth/
│   │   ├── auth.module.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── dto/
│   │   │   ├── login.dto.ts
│   │   │   ├── register.dto.ts
│   │   │   └── refresh-token.dto.ts
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts
│   │   │   └── roles.guard.ts
│   │   ├── decorators/
│   │   │   └── roles.decorator.ts
│   │   └── strategies/
│   │       ├── jwt.strategy.ts
│   │       └── jwt-refresh.strategy.ts
│   ├── users/
│   │   ├── users.module.ts
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── dto/
│   │   │   ├── create-user.dto.ts
│   │   │   └── update-user.dto.ts
│   │   └── entities/
│   │       └── user.entity.ts
│   ├── items/
│   │   ├── items.module.ts
│   │   ├── items.controller.ts
│   │   ├── items.service.ts
│   │   ├── dto/
│   │   │   ├── create-item.dto.ts
│   │   │   ├── update-item-status.dto.ts
│   │   │   └── scan-item.dto.ts
│   │   └── entities/
│   │       └── item.entity.ts
│   ├── inventory-events/
│   │   ├── inventory-events.module.ts
│   │   ├── inventory-events.controller.ts
│   │   ├── inventory-events.service.ts
│   │   ├── dto/
│   │   │   └── create-inventory-event.dto.ts
│   │   └── entities/
│   │       └── inventory-event.entity.ts
│   ├── prisma/
│   │   ├── prisma.module.ts
│   │   └── prisma.service.ts
│   └── common/
│       ├── filters/
│       │   └── http-exception.filter.ts
│       ├── interceptors/
│       │   └── transform.interceptor.ts
│       └── pipes/
│           └── validation.pipe.ts
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── .env.example
├── .env
├── package.json
├── tsconfig.json
└── nest-cli.json
```

---

## Database Schema (Prisma)

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role {
  ADMIN
  OPERATOR
  QC
  VIEWER
}

enum InventoryStatus {
  CREATED
  STORED
  VERIFIED
  DISPATCHED
  CLOSED
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  password  String
  role      Role     @default(VIEWER)
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  inventoryEvents InventoryEvent[]
  refreshTokens   RefreshToken[]

  @@map("users")
}

model RefreshToken {
  id        String   @id @default(uuid())
  token     String   @unique
  userId    String
  expiresAt DateTime
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("refresh_tokens")
}

model Item {
  id            String          @id @default(uuid())
  name          String
  qrCode        String          @unique
  currentStatus InventoryStatus @default(CREATED)
  createdAt     DateTime        @default(now())
  updatedAt     DateTime        @updatedAt

  inventoryEvents InventoryEvent[]

  @@map("items")
}

model InventoryEvent {
  id            String          @id @default(uuid())
  itemId        String
  fromStatus    InventoryStatus
  toStatus      InventoryStatus
  action        String
  updatedBy     String
  updatedAt     DateTime        @default(now())

  item Item @relation(fields: [itemId], references: [id], onDelete: Cascade)
  user User @relation(fields: [updatedBy], references: [id])

  @@map("inventory_events")
}
```

---

## Environment Variables

```env
# .env.example

# Application
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL="postgresql://username:password@localhost:5432/inventory_db?schema=public"

# JWT
JWT_SECRET=your-super-secret-jwt-key-change-this-in-production
JWT_EXPIRES_IN=15m
JWT_REFRESH_SECRET=your-super-secret-refresh-key-change-this-in-production
JWT_REFRESH_EXPIRES_IN=7d

# CORS (comma-separated for multiple origins)
CORS_ORIGIN=http://localhost:5173,http://localhost:3001
```

---

## Authentication & Authorization Rules

### JWT Token Structure
```typescript
// Access Token Payload
{
  sub: string;      // user id
  email: string;
  role: Role;
  iat: number;
  exp: number;
}

// Refresh Token Payload
{
  sub: string;      // user id
  tokenId: string;  // refresh token id in database
  iat: number;
  exp: number;
}
```

### Role-Based Permissions

| Role     | Permissions                                                      |
|----------|------------------------------------------------------------------|
| ADMIN    | All operations, manage users, view all data                      |
| OPERATOR | Create items, update status CREATED→STORED→DISPATCHED            |
| QC       | Update status STORED→VERIFIED                                    |
| VIEWER   | Read-only access to items and history                            |

### Status Transition Rules

```typescript
// Allowed transitions by role
const ROLE_TRANSITIONS = {
  ADMIN: {
    CREATED: ['STORED', 'CLOSED'],
    STORED: ['VERIFIED', 'DISPATCHED', 'CLOSED'],
    VERIFIED: ['DISPATCHED', 'CLOSED'],
    DISPATCHED: ['CLOSED'],
    CLOSED: [] // Terminal state
  },
  OPERATOR: {
    CREATED: ['STORED'],
    STORED: ['DISPATCHED'],
    VERIFIED: ['DISPATCHED'],
    DISPATCHED: ['CLOSED'],
    CLOSED: []
  },
  QC: {
    STORED: ['VERIFIED'],
    VERIFIED: [], // QC can only verify
    CREATED: [],
    DISPATCHED: [],
    CLOSED: []
  },
  VIEWER: {
    // No transitions allowed
  }
};
```

---

## API Endpoints Specification

### Authentication Endpoints

```typescript
// POST /auth/register
Request: {
  email: string;
  password: string;
  role?: Role; // Only ADMIN can set this
}
Response: {
  user: { id, email, role, isActive };
  accessToken: string;
  refreshToken: string;
}

// POST /auth/login
Request: {
  email: string;
  password: string;
}
Response: {
  user: { id, email, role, isActive };
  accessToken: string;
  refreshToken: string;
}

// POST /auth/refresh
Request: {
  refreshToken: string;
}
Response: {
  accessToken: string;
  refreshToken: string;
}

// POST /auth/logout
Headers: { Authorization: Bearer <token> }
Request: {
  refreshToken: string;
}
Response: { message: 'Logged out successfully' }
```

### Item Endpoints

```typescript
// POST /items (Protected: ADMIN, OPERATOR)
Headers: { Authorization: Bearer <token> }
Request: {
  name: string;
}
Response: {
  id: string;
  name: string;
  qrCode: string;
  currentStatus: InventoryStatus;
  createdAt: string;
  updatedAt: string;
}

// GET /items/:id (Protected: All authenticated users)
Headers: { Authorization: Bearer <token> }
Response: {
  id: string;
  name: string;
  qrCode: string;
  currentStatus: InventoryStatus;
  createdAt: string;
  updatedAt: string;
}

// GET /items (Protected: All authenticated users)
Headers: { Authorization: Bearer <token> }
Query: { page?: number; limit?: number; status?: InventoryStatus }
Response: {
  data: Item[];
  total: number;
  page: number;
  limit: number;
}

// POST /items/scan (Protected: ADMIN, OPERATOR, QC)
Headers: { Authorization: Bearer <token> }
Request: {
  qrCode: string;
  newStatus: InventoryStatus;
}
Response: {
  item: Item;
  event: InventoryEvent;
}

// PATCH /items/:id/status (Protected: Role-based)
Headers: { Authorization: Bearer <token> }
Request: {
  newStatus: InventoryStatus;
}
Response: {
  item: Item;
  event: InventoryEvent;
}

// GET /items/:id/history (Protected: All authenticated users)
Headers: { Authorization: Bearer <token> }
Response: {
  events: InventoryEvent[];
}
```

### User Endpoints (Admin Only)

```typescript
// GET /users (Protected: ADMIN)
Headers: { Authorization: Bearer <token> }
Response: { users: User[] }

// GET /users/:id (Protected: ADMIN or Self)
Headers: { Authorization: Bearer <token> }
Response: { user: User }

// PATCH /users/:id (Protected: ADMIN)
Headers: { Authorization: Bearer <token> }
Request: {
  isActive?: boolean;
  role?: Role;
}
Response: { user: User }
```

---

## Business Logic Rules

### QR Code Generation
```typescript
// QR Code format: ITEM-{UUID}
// Example: ITEM-550e8400-e29b-41d4-a716-446655440000

function generateQRCode(itemId: string): string {
  return `ITEM-${itemId}`;
}
```

### Status Update Transaction Flow
```typescript
// Every status update must:
// 1. Validate user role has permission for transition
// 2. Validate current status allows transition
// 3. Create inventory event record
// 4. Update item current status
// 5. All in a single database transaction

async updateItemStatus(
  itemId: string,
  newStatus: InventoryStatus,
  userId: string,
  userRole: Role
): Promise<{ item: Item; event: InventoryEvent }> {
  // Use Prisma transaction
  return await prisma.$transaction(async (tx) => {
    // 1. Get current item
    const item = await tx.item.findUnique({ where: { id: itemId } });
    
    // 2. Validate transition
    validateTransition(item.currentStatus, newStatus, userRole);
    
    // 3. Create event
    const event = await tx.inventoryEvent.create({
      data: {
        itemId,
        fromStatus: item.currentStatus,
        toStatus: newStatus,
        action: `Status changed from ${item.currentStatus} to ${newStatus}`,
        updatedBy: userId
      }
    });
    
    // 4. Update item
    const updatedItem = await tx.item.update({
      where: { id: itemId },
      data: { currentStatus: newStatus }
    });
    
    return { item: updatedItem, event };
  });
}
```

### Duplicate Scan Prevention
```typescript
// Prevent duplicate status updates
// If current status === newStatus, return error
if (item.currentStatus === newStatus) {
  throw new BadRequestException(
    `Item is already in ${newStatus} status`
  );
}
```

---

## Error Handling Standards

### HTTP Status Codes
- **200**: Success
- **201**: Created
- **400**: Bad Request (validation errors, invalid transitions)
- **401**: Unauthorized (missing/invalid token)
- **403**: Forbidden (insufficient permissions)
- **404**: Not Found
- **409**: Conflict (duplicate QR code, duplicate scan)
- **500**: Internal Server Error

### Error Response Format
```typescript
{
  statusCode: number;
  message: string | string[];
  error: string;
  timestamp: string;
  path: string;
}
```

### Custom Exception Examples
```typescript
// Invalid transition
throw new BadRequestException(
  `Cannot transition from ${fromStatus} to ${toStatus} with role ${role}`
);

// QR code not found
throw new NotFoundException(`Item with QR code ${qrCode} not found`);

// Duplicate QR code
throw new ConflictException(`Item with QR code ${qrCode} already exists`);

// Insufficient permissions
throw new ForbiddenException('Insufficient permissions for this operation');
```

---

## Validation Rules

### DTOs Validation
Use class-validator decorators:

```typescript
// Example: CreateItemDto
import { IsString, IsNotEmpty, MinLength } from 'class-validator';

export class CreateItemDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(3)
  name: string;
}

// Example: UpdateItemStatusDto
import { IsEnum, IsNotEmpty } from 'class-validator';

export class UpdateItemStatusDto {
  @IsEnum(InventoryStatus)
  @IsNotEmpty()
  newStatus: InventoryStatus;
}
```

---

## Security Best Practices

### Password Hashing
```typescript
import * as bcrypt from 'bcrypt';

// Hash password (salt rounds: 10)
const hashedPassword = await bcrypt.hash(password, 10);

// Compare password
const isMatch = await bcrypt.compare(plainPassword, hashedPassword);
```

### JWT Guards Implementation
```typescript
// All protected routes must use:
@UseGuards(JwtAuthGuard)

// Role-protected routes must use:
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(Role.ADMIN, Role.OPERATOR)
```

### CORS Configuration
```typescript
// Enable CORS for specified origins only
app.enableCors({
  origin: process.env.CORS_ORIGIN.split(','),
  credentials: true,
});
```

---

## Testing Requirements

### Unit Tests
- All services must have unit tests
- Mock Prisma client using jest
- Test business logic in isolation

### E2E Tests
- Test authentication flow (register, login, refresh, logout)
- Test item creation and QR generation
- Test status transitions with different roles
- Test authorization guards

### Test File Naming
- Unit tests: `*.service.spec.ts`
- E2E tests: `*.e2e-spec.ts`

---

## Code Quality Standards

### TypeScript Rules
- Always use strict mode
- Never use `any` type
- Use interfaces for data structures
- Use enums for fixed value sets
- Use proper return types on all functions

### NestJS Best Practices
- Keep controllers thin (only routing and validation)
- Put business logic in services
- Use dependency injection
- Use DTOs for all inputs
- Use Prisma transactions for multi-step operations
- Use pipes for validation
- Use filters for exception handling
- Use interceptors for response transformation

### Naming Conventions
- Files: kebab-case (e.g., `inventory-events.service.ts`)
- Classes: PascalCase (e.g., `InventoryEventsService`)
- Methods: camelCase (e.g., `updateItemStatus`)
- Constants: UPPER_SNAKE_CASE (e.g., `JWT_EXPIRES_IN`)

---

## Development Workflow

### Initial Setup
```bash
# Install dependencies
npm install

# Generate Prisma client
npx prisma generate

# Run migrations
npx prisma migrate dev --name init

# Seed database (optional)
npx prisma db seed
```

### Running the Application
```bash
# Development
npm run start:dev

# Production
npm run build
npm run start:prod
```

### Database Commands
```bash
# Create new migration
npx prisma migrate dev --name migration_name

# Reset database
npx prisma migrate reset

# Open Prisma Studio
npx prisma studio
```

---

## Deployment Considerations

### Environment-Specific Settings
- Use different DATABASE_URL for dev/staging/prod
- Use strong JWT secrets in production
- Enable SSL for database connections in production
- Set appropriate CORS_ORIGIN for production

### Health Check Endpoint
```typescript
// GET /health
Response: {
  status: 'ok',
  database: 'connected',
  timestamp: string
}
```

---

## Module Dependencies

```typescript
// App Module imports
@Module({
  imports: [
    PrismaModule,
    AuthModule,
    UsersModule,
    ItemsModule,
    InventoryEventsModule,
  ],
})
export class AppModule {}
```

---

## Critical Reminders for Copilot

1. **Never expose sensitive data in responses** (passwords, full JWT secrets)
2. **Always use transactions** for status updates
3. **Always validate role permissions** before state transitions
4. **Never allow status to be updated without creating an inventory event**
5. **QR codes must be unique** and auto-generated
6. **Inventory events are immutable** - never update or delete them
7. **Use proper TypeScript types** - no `any`
8. **All routes must be protected** except register/login
9. **Refresh tokens must be stored in database** and validated
10. **Password must be hashed** using bcrypt before storage

---

## Next Steps After Backend Completion

Once backend is stable:
1. Write comprehensive tests
2. Deploy backend to Render/Railway
3. Start frontend development
4. Test integration between frontend and backend
5. Prepare for mobile app (ensure APIs are platform-agnostic)

---

**END OF BACKEND INSTRUCTIONS**
