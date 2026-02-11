# Schema Drift Is Inevitable. Your Migration Strategy Shouldn't Be Manual.

*How we built a tool that diffs two SQL Server databases and generates FluentMigrator code — so deployments stop being a gamble.*

---

## The Problem Nobody Wants to Talk About

Every enterprise development team has a database deployment process. Very few of them are honest about how well it works.

In theory, database changes are versioned, reviewed, and deployed through the same pipeline as application code. In practice, someone runs a script against staging on a Thursday afternoon, forgets to check it into source control, and three weeks later the production deployment fails because the schema doesn't match what the migration expects.

This is schema drift — the slow, inevitable divergence between what your database looks like and what your code thinks it looks like. In a single-tenant application, it's annoying. In a multi-tenant SaaS platform where each client can be running a different database version with custom objects, it's an operational nightmare.

We lived in this nightmare for long enough that we decided to build our way out of it.

## What We Built

A command-line tool that connects to two SQL Server databases — a source (what the schema should look like) and a target (what it currently looks like) — and generates a complete set of FluentMigrator C# migration classes that transform the target into the source. No manual SQL scripts. No spreadsheets tracking which ALTER TABLE statements need to run against which environment. Point it at two databases, and it produces deployable, version-controlled, rollback-capable migration code.

The tool doesn't just handle tables. It diffs and generates migrations for every structural element in the database: schemas, tables with full column definitions, foreign keys (including composite keys with cascading rules), indexes, unique constraints, check constraints, views, stored procedures, functions, table-valued functions, triggers, and user-defined table types. If SQL Server tracks it in `sys.objects`, this tool can diff it and generate a migration for it.

## How It Works

The architecture has three layers: introspection, comparison, and code generation.

### Introspection

The tool interrogates both databases through their system catalog views — `sys.tables`, `sys.columns`, `sys.indexes`, `sys.foreign_key_columns`, `sys.sql_modules`, and the `INFORMATION_SCHEMA` views. Each database entity type has a dedicated query that extracts every property needed to fully reconstruct the object: column types, precision, scale, nullability, default values, identity seeds, computed columns, temporal table configuration, index filter definitions, and foreign key cascade rules.

This isn't a surface-level comparison. When the tool reads a table, it captures everything down to whether a column is a hidden system-time column on a temporal table, or whether an identity column seeds at 1 and increments by 1 versus seeding at 1000 and incrementing by 10. These details matter because a migration that creates a table without the correct identity specification will silently produce a schema that behaves differently from the original.

### Comparison

For each entity type, the tool loads definitions from both databases and runs a three-way classification. If an entity exists in the source but not the target, it's a **create** — the migration needs to add it. If it exists in both but the definitions differ, it's an **update** — the migration needs to alter it. If it exists only in the target, it's ignored (the tool is intentionally conservative about drops).

The comparison logic is type-aware. Two tables are considered equal only if every column matches on name, type, precision, nullability, default value, identity configuration, and primary key membership. Two indexes match only if they cover the same columns in the same order with the same included columns, filter definitions, and uniqueness constraints. For stored procedures, views, and functions, the tool compares the full SQL definition text from `sys.sql_modules`.

### Dependency Resolution

This is where things get interesting. Database objects have dependencies — a view references tables, a stored procedure calls functions, a foreign key depends on the target table existing. If you generate migrations in the wrong order, the deployment fails.

The tool queries `sys.sql_expression_dependencies` to build a dependency graph across the entire database. When it generates a migration for a view that references three tables and a function, it ensures those dependencies are processed first. The processing pipeline uses Reactive Extensions (`System.Reactive`) to handle this: each entity type publishes to an observable stream when it generates a migration, and dependent generators subscribe to those streams to ensure proper ordering.

The generated migration files are sequenced with monotonically increasing version numbers (derived from UTC timestamps), so FluentMigrator executes them in the exact order the dependency graph requires.

### Code Generation

Here's the part that makes this tool unusual: it doesn't generate SQL. It generates C# classes.

Each migration is a proper FluentMigrator class with `Up()` and `Down()` methods, class-level `[Migration]` attributes with version numbers, and the full FluentMigrator fluent API for schema operations. The tool builds these classes programmatically using `System.CodeDom` — constructing namespaces, class declarations, method bodies, and conditional logic as an abstract syntax tree, then rendering them to `.cs` files through the C# code provider.

For structural entities like tables and indexes, the generated code uses FluentMigrator's native API: `Create.Table().InSchema().WithColumn()` chains that are readable, reviewable, and modify-able by developers. For objects where FluentMigrator doesn't have a native abstraction — stored procedures, views, functions, triggers — the tool generates `Execute.Sql()` calls that reference external `.sql` files containing the full object definition.

This hybrid approach gives you the best of both worlds: type-safe, fluent C# for structural changes, and raw SQL for programmable objects. The generated code is indistinguishable from hand-written migrations, because it uses exactly the same API a developer would use.

### Idempotent by Design

The generated migrations don't assume a clean slate. Table migrations check whether the table exists before creating it, and whether each column exists before adding or altering it. This means you can run the same migration set against a database that's partially migrated — from a failed previous run, or from manual changes that were applied out of band — and it converges to the correct state without errors.

## What a Generated Migration Looks Like

For a table, the tool produces something like this:

```csharp
[Migration(2402151030001)]
public class dbo_CustomerAccounts_7284930 : Migration
{
    public override void Up()
    {
        if (!Schema.Schema("dbo").Table("CustomerAccounts").Exists())
        {
            Create.Table("CustomerAccounts").InSchema("dbo")
                .WithColumn("Id").AsInt32().NotNullable().PrimaryKey("PK_CustomerAccounts").Identity(1, 1)
                .WithColumn("AccountName").AsString(255).NotNullable()
                .WithColumn("Balance").AsDecimal(18, 4).NotNullable().WithDefaultValue(RawSql.Insert("0"))
                .WithColumn("IsActive").AsBoolean().NotNullable().WithDefaultValue(RawSql.Insert("1"))
                .WithColumn("CreatedDate").AsDateTime().NotNullable().WithDefaultValue(RawSql.Insert("getutcdate()"));
        }
        else
        {
            // Per-column existence checks and alter statements
        }
    }

    public override void Down() { }
}
```

For a stored procedure, it generates a class that drops and recreates from an external SQL file:

```csharp
[Migration(2402151030042)]
public class dbo_usp_RecalculateBalances : Migration
{
    public override void Up()
    {
        Execute.Sql(@"drop procedure if exists [dbo].[usp_RecalculateBalances]");
        Execute.Sql(System.IO.File.ReadAllText(@"./StoredProcedures/Up/dbo_usp_RecalculateBalances.sql"));
    }

    public override void Down()
    {
        Execute.Sql(@"drop procedure if exists [dbo].[usp_RecalculateBalances]");
        Execute.Sql(System.IO.File.ReadAllText(@"./StoredProcedures/Down/dbo_usp_RecalculateBalances.sql"));
    }
}
```

The `Down()` method restores the original target definition, giving you a genuine rollback path — not just a "drop and hope" strategy.

## Architecture Decisions Worth Noting

**Why FluentMigrator instead of raw SQL?** Because FluentMigrator gives you a version table, ordered execution, rollback support, and a runner that integrates with CI/CD pipelines. The generated code is a proper .NET assembly that can be executed by the FluentMigrator runner, which means database deployments go through the same build-test-deploy pipeline as application code.

**Why CodeDOM instead of string concatenation?** Because CodeDOM produces syntactically correct C# by construction. You can't accidentally generate a class with mismatched braces or a method that references an undeclared variable. The abstract syntax tree approach also makes it trivial to add conditional logic — like the table-exists checks — without building a string template engine.

**Why Reactive Extensions for the processing pipeline?** Because dependency resolution is inherently event-driven. When a table generator finishes processing a table, anything that depends on that table (foreign keys, indexes, triggers) needs to know immediately. The `Subject<T>` pattern lets each generator publish completion events that downstream generators subscribe to, naturally resolving the ordering problem without explicit topological sorting.

**Why a NuGet project template?** Because in a multi-tenant environment, each tenant's migrations need to be a separate assembly with a distinct namespace and version table. The `dotnet new` template scaffolds a new tenant migration project in seconds, pre-configured with the correct dependencies and runner extensions.

## What We Learned

Building this tool confirmed something we'd suspected for a while: the hard part of database migration isn't generating SQL. It's understanding what changed, in what order things need to be applied, and what happens when reality doesn't match your assumptions.

The dependency resolution alone accounted for more development time than every code generator combined. And the edge cases — temporal tables with hidden columns, composite foreign keys referencing unique constraints instead of primary keys, views that reference other views that reference functions — are the kind of problems you only discover when you point the tool at a real production database with 15 years of organic growth.

The tool got mothballed when organizational priorities shifted, which is a polite way of saying the business needed features more than it needed infrastructure improvements. But the core insight survived: if your database deployment strategy requires someone to remember which scripts to run in which order, you don't have a strategy. You have a single point of failure shaped like a person.

---

*Gregory Rebisz is a software developer and technical content writer specializing in enterprise SaaS, backend architecture, and the kind of infrastructure problems that only surface at 2 AM on a Saturday. He's based in South Africa and available for freelance technical writing.*
