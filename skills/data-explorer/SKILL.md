---
name: data-explorer
description: "Creates an interactive data explorer playground for any database schema, API data model, or entity-relationship diagram — an animated JointJS graph with two-tone entity nodes showing PK and field count, field-level search, full field table modal with PK/FK/UQ/NN badges, sample queries, router toggle, and BFS path highlighting. Use when the user wants to visualize data structures, table relationships, JSON schemas, or entity models — from local schema files, ORM models, migration files, or described schemas. Examples: 'visualize my database schema', 'create a data map for our API models', 'show me the entity relationships in our app', 'explore this Postgres schema', 'build a data explorer for this project', 'map out our database', 'explore our migrations', 'build a JointJS data playground', 'create a JointJS schema explorer'."
---

# Data Explorer Playground

Generates a self-contained HTML file with an animated JointJS graph showing database schemas and entity models: two-tone entity nodes with `PK: <field> · N fields` sublabels, field-level search, FK-labeled relationship links, a full field table modal, and a prompt output bar.

Two modes depending on whether the schema is **described/known** (use as given) or **in local files** (read files first).

---

## Mode 1: Described or Known Schema

Use when the user pastes a schema, describes tables, or names a well-known data model.

1. **Load the template** from `../graph-explorer/templates/graph-explorer.md` — Shared Framework + Data Explorer Spec
2. **Build NODES from the schema** — one node per table/entity/model
3. **Build CONNECTIONS from FK relationships** — with FK column name as `label`
4. **Write the HTML file** — filename: `<subject>-data-explorer.html`
5. **Open in browser** — `open <filename>-data-explorer.html`

---

## Mode 2: Local Schema Files

Use when the user says "this project", "our database", "our migrations", or points to a local path. Read schema files first.

### Phase 1 — Discover schema files

Search in priority order — stop when you find a match:

```
# SQL migrations
Glob('migrations/**/*.sql')
Glob('db/migrate/**/*.rb')        # Rails
Glob('database/migrations/**/*.php') # Laravel

# ORM model definitions
Glob('src/**/models/*.{ts,js}')   # TypeScript/JS ORMs (TypeORM, Prisma, Mongoose)
Glob('app/models/**/*.rb')        # Rails ActiveRecord
Glob('models/**/*.py')            # Django, SQLAlchemy
Glob('**/schema.prisma')          # Prisma

# Schema dumps / introspection output
Glob('**/*.sql')                  # schema.sql, dump.sql
Glob('**/schema.{json,yaml,yml}') # JSON Schema, OpenAPI components
```

Also check: `package.json` for ORM name (prisma, typeorm, sequelize, mongoose) to guide parsing strategy.

### Phase 2 — Extract entities and fields

**SQL migrations / schema dumps:**
```sql
CREATE TABLE orders (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users(id),  -- → FK link to users, label: 'user_id'
  total numeric(10,2) NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
```
Extract: table name, columns (name + type), PRIMARY KEY, FOREIGN KEY references, UNIQUE constraints, NOT NULL.

**Prisma schema:**
```prisma
model User {
  id        String   @id @default(uuid())
  email     String   @unique
  orders    Order[]          -- → relation, emit link Order→User
}
```
Extract: model name, fields (name + type + attributes), `@relation` → FK links.

**TypeORM / Sequelize / Django models:**
- Look for `@Entity`, `@Column`, `@ManyToOne`, `@OneToMany` decorators (TypeORM)
- Or `models.Model` subclasses with field definitions (Django)
- Extract class name → entity, field names + types, FK decorators → links

**Rails ActiveRecord:**
- `create_table :orders` in migrations → entity
- `t.references :user` → FK link to users
- `belongs_to :user` in model file → confirms relationship

**OpenAPI / JSON Schema:**
- `components/schemas` → one node per schema object
- `$ref` to another schema → link

### Phase 3 — Build NODES and CONNECTIONS

```js
// One node per table/model
{
  id: 'orders',
  label: 'orders',
  sub: 'schema_name_or_file_path',
  layer: 'derived_domain_group',  // see Phase 4
  fields: [
    { name: 'id',         type: 'uuid',         pk: true,  nullable: false },
    { name: 'user_id',    type: 'uuid',         fk: true,  nullable: false },
    { name: 'total',      type: 'numeric(10,2)', nullable: false },
    { name: 'created_at', type: 'timestamptz',   nullable: false },
  ],
  snippet: `-- generated sample query\nSELECT * FROM orders WHERE id = $1;`,
}

// One connection per FK / relation
{ from: 'orders', to: 'users', type: 'foreignKey', label: 'user_id' }
```

### Phase 4 — Derive layers from naming conventions

Group tables into domain layers by name prefix/suffix patterns:

| Pattern | Layer |
|---|---|
| `user`, `account`, `session`, `role`, `permission` | `auth` |
| `product`, `item`, `inventory`, `category`, `sku` | `product` |
| `order`, `cart`, `checkout`, `line_item` | `order` |
| `payment`, `invoice`, `charge`, `refund`, `transaction` | `payment` |
| `event`, `log`, `audit`, `metric`, `stat` | `analytics` |
| `setting`, `config`, `feature_flag` | `config` |
| Uncategorized | `core` |

Assign one color per layer using the LAYERS palette from the Data Explorer Spec — rename keys to match what was found.

### Phase 5 — Generate

Write the HTML file using the graph-explorer template (Shared Framework + Data Explorer Spec), substituting the extracted NODES, CONNECTIONS, and derived LAYERS. Add presets:
- `full` — always
- One preset per major domain group found (e.g. `auth`, `transactions`)

**Filename:** `<project-name>-data-explorer.html` where project name = root directory name or `package.json` name.

### Pitfalls for local schema mode

| Problem | Fix |
|---|---|
| Many small lookup/enum tables | Merge into a `config` or `shared` layer node; list them in `desc` |
| No FK constraints in schema (app-level only) | Read model files for `belongs_to`/`@ManyToOne` to recover relationships |
| Prisma enums | Skip as nodes — mention in the field's `type` string |
| Migration files out of order | Read the latest state: prefer a `schema.sql` dump over individual migration files |
| Too many tables (>25) | One node per domain group — aggregate fields from all tables in the group into a summary node |

---

## Key features (beyond the shared framework)

- **EntityNode** — two-tone rect: color strip header + body, `PK: <field> · N fields` sublabel
- **Field-level search** — `getVisibleIds()` also matches field names and types (e.g. type "uuid" to find all UUID fields)
- **Field table modal** — full field list with PK/FK/UQ/NN constraint badges, sample query, note textarea
- **FK labels on links** — `label` property rendered as a small annotation at `distance: 0.45`
- **Record hint** — optional badge (e.g. "~2M rows") shown in the modal header

## Quick reference

- **Output**: single self-contained HTML file
- **CDN**: `https://cdn.jsdelivr.net/npm/@joint/core@4.2.4/dist/joint.min.js`
- **Template**: `../graph-explorer/templates/graph-explorer.md` → Shared Framework + Data Explorer Spec
- **Filename**: `<subject>-data-explorer.html`
- **Node size**: 170×50px
- **Namespace**: `datamap.EntityNode`, `datamap.Link`
