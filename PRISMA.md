## 🎯 Prisma Core Concepts

### 1. The Big Picture

Prisma has **3 main parts** :[^3][^2]

**Prisma Schema** (`schema.prisma`) - Your single source of truth that defines:

- Database connection
- Data models (tables/collections)
- Relations between models
- Indexes and constraints[^2][^3]

**Prisma Client** - Auto-generated TypeScript client for type-safe database queries[^4][^3]

**Prisma Migrate** - Version control for your database schema[^5][^6]

### 2. Schema Basics

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Models = Tables
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  // Relations
  properties Property[]
  bookings   Booking[]
}

model Property {
  id          String   @id @default(cuid())
  title       String
  price       Float
  description String?
  
  // Foreign key
  ownerId     String
  owner       User     @relation(fields: [ownerId], references: [id])
  
  bookings Booking[]
  
  @@index([ownerId])
}
```

**Key decorators** :[^3][^2]

- `@id` - Primary key
- `@unique` - Unique constraint
- `@default()` - Default value
- `@relation()` - Define relationships
- `@@index()` - Add database indexes
- `@@unique()` - Composite unique constraints[^7]


### 3. Relations

**One-to-Many** (Most common) :[^8][^9]

```prisma
model User {
  id         String     @id
  properties Property[] // Array = "many" side
}

model Property {
  id      String @id
  ownerId String
  owner   User   @relation(fields: [ownerId], references: [id])
}
```

**One-to-One** :[^9][^10]

```prisma
model User {
  id      String   @id
  profile Profile?
}

model Profile {
  id     String @id
  userId String @unique // @unique makes it one-to-one
  user   User   @relation(fields: [userId], references: [id])
}
```

**Many-to-Many** :[^11][^12]

```prisma
model Property {
  id       String             @id
  features PropertyFeature[]
}

model Feature {
  id         String             @id
  properties PropertyFeature[]
}

// Join table
model PropertyFeature {
  propertyId String
  featureId  String
  property   Property @relation(fields: [propertyId], references: [id])
  feature    Feature  @relation(fields: [featureId], references: [id])
  
  @@id([propertyId, featureId])
}
```

**Multiple relations to same model** - Use `@relation("name")` :[^13][^14]

```prisma
model Booking {
  requestedBy   User   @relation("RequestedBookings", fields: [requestedById], references: [id])
  requestedById String
  
  ownerUser   User   @relation("OwnedBookings", fields: [ownerUserId], references: [id])
  ownerUserId String
}
```


### 4. CRUD Operations

**Setup Client** :[^15][^4]

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();
```

**Create** :[^16][^4]

```typescript
const user = await prisma.user.create({
  data: {
    email: 'john@example.com',
    name: 'John',
    properties: {
      create: [{ // Nested create
        title: 'Beach House',
        price: 500000
      }]
    }
  }
});
```

**Read** :[^4][^16]

```typescript
// Find many
const users = await prisma.user.findMany({
  where: { email: { contains: '@gmail.com' } },
  include: { properties: true }, // Include relations
  orderBy: { createdAt: 'desc' },
  take: 10, // Limit
  skip: 0   // Offset
});

// Find unique
const user = await prisma.user.findUnique({
  where: { id: 'user-123' }
});

// Find first
const user = await prisma.user.findFirst({
  where: { name: 'John' }
});
```

**Update** :[^16][^4]

```typescript
const user = await prisma.user.update({
  where: { id: 'user-123' },
  data: { name: 'John Doe' }
});

// Update many
await prisma.property.updateMany({
  where: { price: { lt: 100000 } },
  data: { status: 'BUDGET_FRIENDLY' }
});
```

**Delete** :[^4][^16]

```typescript
await prisma.user.delete({
  where: { id: 'user-123' }
});

// Delete many
await prisma.property.deleteMany({
  where: { status: 'SOLD' }
});
```


### 5. Advanced Queries (15 minutes)

**Filters** :[^17][^16]

```typescript
const properties = await prisma.property.findMany({
  where: {
    AND: [
      { price: { gte: 100000, lte: 500000 } },
      { owner: { email: { endsWith: '@gmail.com' } } }
    ]
  }
});
```

**Nested includes** :[^17][^16]

```typescript
const booking = await prisma.booking.findUnique({
  where: { id: 'booking-123' },
  include: {
    property: {
      include: { owner: true }
    },
    timeSlots: true
  }
});
```

**Relation queries** :[^18][^17]

```typescript
// Connect existing relation
await prisma.booking.update({
  where: { id: 'booking-123' },
  data: {
    activeTimeSlot: { connect: { id: 'slot-456' } }
  }
});

// Disconnect relation
await prisma.booking.update({
  where: { id: 'booking-123' },
  data: {
    activeTimeSlot: { disconnect: true }
  }
});
```


### 6. Migrations

**Development workflow** :[^6][^19][^5]

```bash
# 1. Edit schema.prisma

# 2. Create migration
npx prisma migrate dev --name add_timeslot_feature

# 3. Apply to database + generate Prisma Client
# (automatically runs with migrate dev)
```

**Production workflow** :[^20][^21][^6]

```bash
# Deploy migrations (never use migrate dev in production)
npx prisma migrate deploy
```

**Push schema directly** (for prototyping) :[^20]

```bash
# Skip migrations, push schema directly
npx prisma db push
```

**Introspection** (from existing database) :[^5]

```bash
# Generate schema from existing database
npx prisma db pull
```


### 7. Transactions

**Sequential transactions** :[^22][^23]

```typescript
const result = await prisma.$transaction(async (tx) => {
  // Release old slot
  await tx.timeSlot.update({
    where: { id: oldSlotId },
    data: { isBooked: false }
  });
  
  // Book new slot
  await tx.timeSlot.update({
    where: { id: newSlotId },
    data: { isBooked: true }
  });
  
  // Update booking
  return tx.booking.update({
    where: { id: bookingId },
    data: { rescheduleCount: { increment: 1 } }
  });
});
```


### 8. Best Practices

**Connection pooling** :[^24][^20]

```typescript
// Reuse single instance
export const prisma = new PrismaClient();
```

**Error handling** :[^3]

```typescript
try {
  await prisma.user.create({ data: { email: 'test@test.com' } });
} catch (error) {
  if (error.code === 'P2002') {
    // Unique constraint violation
  }
}
```

**Soft deletes** :[^20]

```prisma
model User {
  id        String    @id
  deletedAt DateTime?
}
```


## 🚀 Quick Start Checklist

For your invusprop project :[^25][^1]

1. **Install**:
```bash
npm install prisma @prisma/client
npx prisma init
```

2. **Configure** `.env`:
```
DATABASE_URL="postgresql://user:password@host:5432/dbname"
```

3. **Create schema** → `prisma migrate dev` → Start coding![^2][^6]

## 📚 Reference

- Official docs: [prisma.io/docs](https://www.prisma.io/docs)[^26][^1]
- Video tutorial: "Learn Prisma In 60 Minutes"[^2]
- Error codes: Check `error.code` for specific Prisma errors[^3]

This covers 95% of what you'll use daily. Practice with your PropertyBooking and TimeSlot models - you'll master it quickly![^27][^2]
<span style="display:none">[^28][^29][^30][^31][^32][^33]</span>

[^1]: https://www.prisma.io/docs/getting-started

[^2]: https://www.youtube.com/watch?v=RebA5J-rlwg

[^3]: https://betterstack.com/community/guides/scaling-nodejs/prisma-orm/

[^4]: https://dev.to/arindam_1729/create-a-crud-app-with-prisma-orm-node-js-34b1

[^5]: https://www.prisma.io/docs/orm/prisma-migrate/understanding-prisma-migrate/mental-model

[^6]: https://www.answeroverflow.com/m/1338479942110941294

[^7]: https://www.prisma.io/docs/orm/prisma-client/special-fields-and-types/working-with-composite-ids-and-constraints

[^8]: https://www.prisma.io/docs/orm/prisma-schema/data-model/relations/one-to-many-relations

[^9]: https://www.prisma.io/docs/orm/prisma-schema/data-model/relations/one-to-one-relations

[^10]: https://dev.to/lemartin07/understanding-one-to-one-relations-with-prisma-orm-3i3m

[^11]: https://www.prisma.io/docs/orm/prisma-schema/data-model/relations

[^12]: https://stackoverflow.com/questions/73989814/disconnecting-a-many-to-many-relation-in-prisma

[^13]: https://stackoverflow.com/questions/72190270/prisma-throwing-error-ambiguous-relation-detected

[^14]: https://www.answeroverflow.com/m/1336399617725698138

[^15]: https://sabinadams.hashnode.dev/basic-crud-operations-in-prisma

[^16]: https://www.prisma.io/docs/orm/prisma-client/queries/crud

[^17]: https://www.prisma.io/docs/orm/prisma-client/queries/relation-queries

[^18]: https://github.com/prisma/prisma/discussions/23372

[^19]: https://github.com/prisma/prisma/discussions/24571

[^20]: https://planetscale.com/docs/vitess/tutorials/prisma-best-practices

[^21]: https://stackoverflow.com/questions/75735106/prevent-prisma-data-loss-in-production-when-migrate-schema

[^22]: https://www.prisma.io/docs/orm/prisma-client/queries/transactions

[^23]: https://wilfred9.hashnode.dev/mastering-database-transactions-with-prisma-a-step-by-step-guide-for-nodejs-developers-using-typescript

[^24]: https://www.paigeniedringhaus.com/blog/tips-and-tricks-for-using-the-prisma-orm/

[^25]: https://examples.tely.ai/prisma-setup-guide-for-beginners-a-step-by-step-tutorial/

[^26]: https://www.prisma.io/docs/guides

[^27]: https://www.youtube.com/watch?v=gimSKEsWYb4

[^28]: https://www.reddit.com/r/node/comments/1jiur0v/introduction_to_prisma_orm_tutorial_video_series/

[^29]: https://www.youtube.com/watch?v=uVOqu4DUrLg

[^30]: https://dev.to/sandrockjustin/the-prisma-orm-a-brief-overview-and-introduction-353m

[^31]: https://www.prisma.io/dataguide/types/relational/migration-strategies

[^32]: https://www.youtube.com/watch?v=iI7Yr4_RZqo

[^33]: https://www.prisma.io/blog/backend-prisma-typescript-orm-with-postgresql-data-modeling-tsjs1ps7kip1
