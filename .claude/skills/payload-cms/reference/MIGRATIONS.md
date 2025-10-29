# Payload CMS Database Migrations Reference

Complete reference for managing database migrations in Payload CMS 3.x, covering schema changes, migration commands, and best practices for SQL and MongoDB databases.

## Migration Commands

```bash
# Create a schema migration (after config changes)
# Use --skip-empty to prevent empty migrations
pnpm payload migrate:create --skip-empty [descriptive-slug]

# Create a data migration (only for custom data migration logic)
# Use --force-accept-warning to create empty migration you'll edit - you must use this to generate the required json files.
pnpm payload migrate:create --force-accept-warning [descriptive-slug]

# Run pending migrations
pnpm payload migrate

# Check migration status
pnpm payload migrate:status

# Roll back last migration batch
pnpm payload migrate:down

# Reset all migrations (development only) - ONLY EVER USE WHEN EXPLICITLY INSTRUCTED - EVEN IF PERMISSIONS ALLOW
pnpm payload migrate:reset

# Refresh migrations (rollback all + re-run) - ONLY EVER USE WHEN EXPLICITLY INSTRUCTED - EVEN IF PERMISSIONS ALLOW
pnpm payload migrate:refresh

# Fresh migration (drop all + re-run) - ONLY EVER USE WHEN EXPLICITLY INSTRUCTED - EVEN IF PERMISSIONS ALLOW
pnpm payload migrate:fresh
```

**Note**: Always provide a descriptive slug to create readable migration filenames (e.g., `add-user-roles`, `migrate-clinic-data`).

## Migration File Structure

### Basic Migration Template

**Schema migrations** are auto-generated and should NOT be manually edited.

**Data migrations** are created empty with `--force-accept-warning` and you edit them to add custom logic.

```ts
import { MigrateUpArgs, MigrateDownArgs } from '@payloadcms/your-db-adapter'

export async function up({ payload, req }: MigrateUpArgs): Promise<void> {
  // Perform changes to your database here.
  // You have access to `payload` as an argument, and
  // everything is done in TypeScript.
}

export async function down({ payload, req }: MigrateDownArgs): Promise<void> {
  // Revert changes for schema correctness.
  // Doesn't need perfect data reversal, just schema correctness.
}
```

## CI/CD Integration

### Build Script Configuration

```json
{
  "scripts": {
    "dev": "next dev --turbo",
    "build": "next build",
    "payload": "cross-env NODE_OPTIONS=--no-deprecation payload",

    // CI build script runs migrations before build
    "ci": "payload migrate && pnpm build"
  }
}
```

### Vercel Deployment

Set your build command to `ci` script:

```bash
# In Vercel settings
Build Command: pnpm ci
```

### Runtime Migrations (Alternative)

For long-running servers or restricted build environments:

```ts
// Import migrations from generated index.ts
import { migrations } from './migrations'
import { buildConfig } from 'payload'

export default buildConfig({
  // your config here
  db: postgresAdapter({
    // your adapter config here
    prodMigrations: migrations,
  }),
})
```

## Schema Generation

### Generate Drizzle Schema (SQL Only)

```bash
# Generate type-safe Drizzle schema
npx payload generate:db-schema
```

Run this command whenever your Payload config changes (collections, fields, etc.) to ensure type safety.

### Access Generated Schema

```ts
// Accessing generated tables
const myTables = payload.db.tables

// Accessing generated enums
const myEnums = payload.db.enums

// Accessing generated relations
const myRelations = payload.db.relations
```

## Advanced Schema Customization

### Add Custom Tables with `beforeSchemaInit`

```ts
import { postgresAdapter } from '@payloadcms/db-postgres'
import { pgTable, serial, integer } from '@payloadcms/db-postgres/drizzle/pg-core'

postgresAdapter({
  beforeSchemaInit: [
    ({ schema, adapter }) => {
      // Add completely new table
      adapter.rawTables.myTable = {
        name: 'my_table',
        columns: {
          my_id: {
            name: 'my_id',
            type: 'serial',
            primaryKey: true,
          },
        },
      }

      // Add column to existing Payload table
      adapter.rawTables.posts.columns.customColumn = {
        name: 'custom_column',
        type: 'integer',
        notNull: true,
      }

      // Add index to existing table
      adapter.rawTables.posts.indexes.customColumnIdx = {
        name: 'custom_column_idx',
        unique: true,
        on: ['custom_column'],
      }

      return schema
    },
  ],
})
```

### Extend Schema with `afterSchemaInit`

```ts
import { postgresAdapter } from '@payloadcms/db-postgres'
import { index, integer } from '@payloadcms/db-postgres/drizzle/pg-core'

export default buildConfig({
  collections: [
    {
      slug: 'places',
      fields: [
        { name: 'country', type: 'text' },
        { name: 'city', type: 'text' },
      ],
    },
  ],
  db: postgresAdapter({
    afterSchemaInit: [
      ({ schema, extendTable, adapter }) => {
        // Extend existing table with column and composite index
        extendTable({
          table: schema.tables.places,
          columns: {
            extraIntegerColumn: integer('extra_integer_column'),
          },
          extraConfig: (table) => ({
            country_city_composite_index: index(
              'country_city_composite_index',
            ).on(table.country, table.city),
          }),
        })

        return schema
      },
    ],
  }),
})
```

### Import Existing Drizzle Schema

```ts
import { postgresAdapter } from '@payloadcms/db-postgres'
import { users, countries } from '../drizzle/schema'

postgresAdapter({
  beforeSchemaInit: [
    ({ schema, adapter }) => {
      return {
        ...schema,
        tables: {
          ...schema.tables,
          users,
          countries,
        },
      }
    },
  ],
})
```

Example external schema:

```ts
import {
  pgTable,
  uniqueIndex,
  serial,
  varchar,
  text,
} from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  fullName: text('full_name'),
  phone: varchar('phone', { length: 256 }),
})

export const countries = pgTable(
  'countries',
  {
    id: serial('id').primaryKey(),
    name: varchar('name', { length: 256 }),
  },
  (countries) => {
    return {
      nameIndex: uniqueIndex('name_idx').on(countries.name),
    }
  },
)
```

## Using Drizzle Query APIs

### Direct Drizzle Queries

```ts
import { posts } from './payload-generated-schema'
import { eq, sql, and } from '@payloadcms/db-postgres/drizzle'

// Drizzle's Querying API
const postsData = await payload.db.drizzle.query.posts.findMany()

// Drizzle's Select API
const result = await payload.db.drizzle
  .select()
  .from(posts)
  .where(
    and(eq(posts.id, 50), sql`lower(${posts.title}) = 'example post title'`),
  )
```

## Migration Workflows

### Schema Migrations vs Data Migrations

**Schema Migrations** (automatic):
- Generated automatically after config changes
- Use `--skip-empty` flag to prevent empty files
- DO NOT manually edit these files
- Created when adding/removing collections or fields

**Data Migrations** (manual):
- Custom logic for transforming existing data
- Use `--force-accept-warning` to create empty file
- Edit the `up()` and `down()` functions manually
- Required when data structure changes

### Multi-Phase Approach for Adding and Removing Fields

**Best Practice**: When add and removing fields in one task, separate into multiple phases to prevent the migration CLI from becoming interactive.

```bash
# Phase 1: Add new field
# Edit payload.config.ts - add new field
pnpm payload migrate:create --skip-empty add-new-field

# Phase 2: Data migration (if needed)
pnpm payload migrate:create --force-accept-warning migrate-data-to-new-field

# Phase 3: Remove old field
# Edit payload.config.ts - remove old field
pnpm payload migrate:create --skip-empty remove-old-field
```

**Why separate?** Combining additions and removals in one migration causes the CLI to prompt for manual input, which breaks automation and CI/CD workflows.

### Verify Schema Before Data Migrations

**IMPORTANT**: Before writing custom data migrations, check the most recent `*.json` file in your migrations folder to verify table and column existence.

```bash
# Find most recent migration JSON
ls -lt migrations/*.json | head -1

# Review to verify tables/columns exist
cat migrations/YYYYMMDD_HHMMSS_add-status-field.json
```

These JSON files contain the current schema as understood by the migration tool. Use them to verify structure before writing data transformation logic.

### Data Transformation Pattern

```ts
/**
 * DATA MIGRATION: Transform firstName and lastName to fullName
 * Prerequisites verified in migrations/YYYYMMDD_HHMMSS_add-fullname-field.json:
 * - posts table exists
 * - firstName, lastName, and fullName columns exist
 */
export async function up({ payload, req }: MigrateUpArgs): Promise<void> {
  // Fetch all documents that need transformation
  const { docs } = await payload.find({
    collection: 'posts',
    limit: 1000,
    pagination: false,
    req,
  })

  // Transform each document
  for (const doc of docs) {
    await payload.update({
      collection: 'posts',
      id: doc.id,
      data: {
        // Transform old structure to new
        fullName: `${doc.firstName} ${doc.lastName}`,
      },
      req,
    })
  }
}

export async function down({ payload, req }: MigrateDownArgs): Promise<void> {
  // Provide reasonable rollback (schema correctness)
  // Clear the fullName field that was populated in up()
  const { docs } = await payload.find({
    collection: 'posts',
    limit: 1000,
    pagination: false,
    req,
  })

  for (const doc of docs) {
    await payload.update({
      collection: 'posts',
      id: doc.id,
      data: {
        fullName: '',
      },
      req,
    })
  }
}
```

### Migration Flags and Best Practices

```bash
# Schema migration: Skip empty files after config changes
pnpm payload migrate:create --skip-empty add-user-roles

# Data migration: Create empty file to edit manually
pnpm payload migrate:create --force-accept-warning migrate-user-roles-data
```

**Descriptive slugs**: Always provide a slug argument to create readable filenames. Multiple migrations can have the same slug (they'll have different timestamps).

## Environment-Specific Migrations

### Development

```bash
# Create and test migrations locally
pnpm payload migrate:create add-user-roles
pnpm payload migrate
```

### Staging/Production

```bash
# Check what will run
pnpm payload migrate:status

# Run pending migrations
pnpm payload migrate

# Verify success
pnpm payload migrate:status
```

## Best Practices

### 1. Always Test Migrations

```bash
# Create migration
pnpm payload migrate:create

# Test locally
pnpm payload migrate

# Verify changes
pnpm payload migrate:status

# Test rollback
pnpm payload migrate:down
```

### 2. Use Transactions

```ts
export async function up({ payload, req }: MigrateUpArgs): Promise<void> {
  // req includes transactionID for SQL databases
  // All operations using req will be in same transaction
  await payload.update({
    collection: 'posts',
    data: { /* ... */ },
    req, // Ensures transaction support
  })
}
```

### 3. Handle Large Datasets

```ts
export async function up({ payload, req }: MigrateUpArgs): Promise<void> {
  let page = 1
  let hasMore = true

  while (hasMore) {
    const { docs, hasNextPage } = await payload.find({
      collection: 'posts',
      limit: 100,
      page,
      req,
    })

    for (const doc of docs) {
      await payload.update({
        collection: 'posts',
        id: doc.id,
        data: { /* transform */ },
        req,
      })
    }

    hasMore = hasNextPage
    page++
  }
}
```

### 4. Separate Schema and Data Migrations

Keep schema changes and data transformations in separate migrations:

```bash
# ✅ CORRECT: Separate migrations
pnpm payload migrate:create --skip-empty add-status-field
pnpm payload migrate:create --force-accept-warning populate-status-field

# ❌ AVOID: Combining both in manual edits
# Creates confusion about which is schema vs data
```

### 5. Document Data Migrations

```ts
/**
 * DATA MIGRATION: Populate user roles with default values
 * Created: 2024-01-15
 *
 * Prerequisites (verified in migrations/YYYYMMDD_HHMMSS_add-user-roles.json):
 * - users table exists
 * - role column exists with type 'text'
 *
 * Changes:
 * - Sets default 'member' role for existing users
 * - Sets 'admin' role for users with @company.com email
 *
 * Rollback:
 * - Clears role field (schema correctness only)
 */
export async function up({ payload, req }: MigrateUpArgs): Promise<void> {
  // Migration code...
}
```

### 6. Always Run Migrations in Order

```bash
# Check status
pnpm payload migrate:status

# Run all pending
pnpm payload migrate

# Verify
pnpm payload migrate:status
```

Never skip, delete, or reorder migration files.

## Common Migration Scenarios

### Add New Field with Default Value

```ts
export async function up({ payload, req }: MigrateUpArgs): Promise<void> {
  const { docs } = await payload.find({
    collection: 'posts',
    limit: 1000,
    pagination: false,
    req,
  })

  for (const doc of docs) {
    await payload.update({
      collection: 'posts',
      id: doc.id,
      data: {
        status: 'draft', // Set default for existing records
      },
      req,
    })
  }
}
```

### Rename Field

Use the multi-phase approach to avoid interactive prompts:

```bash
# Phase 1: Add new field to config
pnpm payload migrate:create --skip-empty add-new-field-name

# Phase 2: Data migration - copy data
pnpm payload migrate:create --force-accept-warning copy-to-new-field-name

# Phase 3: Remove old field from config
pnpm payload migrate:create --skip-empty remove-old-field-name
```

Phase 2 migration example:

```ts
/**
 * DATA MIGRATION: Copy oldFieldName to newFieldName
 * Prerequisites verified in migrations/YYYYMMDD_HHMMSS_add-new-field-name.json
 */
export async function up({ payload, req }: MigrateUpArgs): Promise<void> {
  const { docs } = await payload.find({
    collection: 'posts',
    limit: 1000,
    pagination: false,
    req,
  })

  for (const doc of docs) {
    await payload.update({
      collection: 'posts',
      id: doc.id,
      data: {
        newFieldName: doc.oldFieldName,
      },
      req,
    })
  }
}

export async function down({ payload, req }: MigrateDownArgs): Promise<void> {
  // Copy back for schema correctness
  const { docs } = await payload.find({
    collection: 'posts',
    limit: 1000,
    pagination: false,
    req,
  })

  for (const doc of docs) {
    await payload.update({
      collection: 'posts',
      id: doc.id,
      data: {
        oldFieldName: doc.newFieldName,
      },
      req,
    })
  }
}
```

### Change Field Type

```ts
export async function up({ payload, req }: MigrateUpArgs): Promise<void> {
  const { docs } = await payload.find({
    collection: 'products',
    limit: 1000,
    pagination: false,
    req,
  })

  for (const doc of docs) {
    await payload.update({
      collection: 'products',
      id: doc.id,
      data: {
        // Convert string to number
        price: parseFloat(doc.price),
      },
      req,
    })
  }
}
```

### Populate Relationship Field

```ts
export async function up({ payload, req }: MigrateUpArgs): Promise<void> {
  const { docs } = await payload.find({
    collection: 'posts',
    limit: 1000,
    pagination: false,
    req,
  })

  for (const doc of docs) {
    // Find related user by email
    const { docs: users } = await payload.find({
      collection: 'users',
      where: { email: { equals: doc.authorEmail } },
      limit: 1,
      req,
    })

    if (users.length > 0) {
      await payload.update({
        collection: 'posts',
        id: doc.id,
        data: {
          author: users[0].id,
        },
        req,
      })
    }
  }
}
```

## Troubleshooting

### Migration CLI Becomes Interactive

**Cause**: Adding and removing fields in the same migration.

**Solution**: Use the multi-phase approach - separate add and remove into different migrations.

### Migration Stuck or Failed

```bash
# Check status
pnpm payload migrate:status

# Manually mark as rolled back if stuck
# (requires direct database access)

# Re-run migrations
pnpm payload migrate
```

### Schema Out of Sync

```bash
# Regenerate schema
npx payload generate:db-schema

# Create migration for changes
pnpm payload migrate:create sync-schema

# Run migration
pnpm payload migrate
```

### Rollback All Migrations

### ONLY EVER DO THE FOLLOWING WHEN EXPLICITLY INSTRUCTED BY USER - EVEN IF PERMISSIONS ALLOW
```bash
# Development only - destroys data!
pnpm payload migrate:reset

# Fresh start
pnpm payload migrate:fresh
```

## Adding Indexes

**Important**: Do NOT create migrations manually to add indexes. Instead, add indexes via field configuration or collection configuration.

### Single Field Indexes

Add indexes to individual fields in your collection definition:

```ts
// ✅ CORRECT: Add index via field config
{
  name: 'email',
  type: 'email',
  index: true,  // Creates index automatically
  unique: true,
}
```

See [FIELDS.md - Text Field](./FIELDS.md#text-field) for more information on field-level index configuration.

### Compound Indexes

For indexes on multiple fields, use the `indexes` property at the collection level:

```ts
import type { CollectionConfig } from 'payload'

export const MyCollection: CollectionConfig = {
  slug: 'my-collection',
  fields: [
    { name: 'title', type: 'text' },
    { name: 'status', type: 'select', options: ['draft', 'published'] },
    { name: 'createdAt', type: 'date' },
  ],
  // ✅ CORRECT: Compound index via collection config
  indexes: [
    {
      fields: ['status', 'createdAt'],
      unique: false, // Optional: set to true for unique constraint
    },
    {
      fields: ['title', 'status'],
      unique: true, // Ensure combination is unique
    },
  ],
}
```

When you run `pnpm payload migrate:create --skip-empty`, Payload will detect the indexes and generate the appropriate migration automatically.

## Related Documentation

- [COLLECTIONS.md](./COLLECTIONS.md) - Collection configuration
- [FIELDS.md](./FIELDS.md) - Field types and validation
- [ADAPTERS.md](./ADAPTERS.md) - Database adapters and configuration
- [QUERIES.md](./QUERIES.md) - Querying data with Payload API

## Resources

- [Payload Migrations Docs](https://payloadcms.com/docs/database/migrations)
- [Drizzle ORM](https://orm.drizzle.team/)
- [PostgreSQL Migrations](https://payloadcms.com/docs/database/postgres#migrations)
- [MongoDB Migrations](https://payloadcms.com/docs/database/mongodb#migrations)
