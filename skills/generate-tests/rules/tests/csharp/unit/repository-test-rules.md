---
title: Repository and Persistence Adapter Test Rules
impact: HIGH
impactDescription: tests data access against the real database engine instead of a fake that cannot fail like it
tags: csharp, tests, repository, persistence, database, sqlite, dapper, ef-core, migrations
---

## Repository and Persistence Adapter Test Rules

Test the classes that talk to the database — repositories, units of work, migrations, any
adapter that issues SQL — against the real database engine, in a database created for the test.

### Why Persistence Adapters Are the One I/O Exception

`domain-service-rules.md` and `csharp-test-template.md` keep I/O out of unit tests. A
persistence adapter is the one exception this distribution allows, because its behavior
*is* the SQL: the query, the mapping, and the constraints and triggers the schema enforces.
Replacing the connection with a fake leaves everything the adapter exists for untested.

The exception is scoped to persistence adapters. The services that use them still receive a
fake of the port.

### Test Setup

- No test sees another test's data:
  - **SQLite**: each test creates its own database in a temporary file and deletes it at the end (`using var` on an `IDisposable` helper)
  - **Server databases** (SQL Server, PostgreSQL, MySQL): follow the setup the project's existing database tests use — typically a container or a database created per test class, with each test rolling back its transaction or cleaning up what it wrote
- Create the schema with the project's real migrations, never with SQL written in the test
- With SQLite, prefer a temporary file over an in-memory database when the project relies on behavior that differs in memory (WAL, several connections, file locking). The EF Core in-memory provider is not a database: it enforces no constraints and runs no SQL
- Look for an existing helper in the test project (a temporary database, a seeded store, a container fixture) before writing one. If the project has no way to create a test database yet, say so and ask before adding one — a container needs Docker on every machine that runs the tests

**FORBIDDEN:** Faking `DbConnection`, `IDbConnection`, the `DbContext` or the Dapper calls in a repository test, and pointing a test at a shared or developer database.

**Incorrect:**

```csharp
// Faking the connection - the SQL never runs
public sealed class OrderRepositoryTests
{
    [Fact]
    public void Save_ValidOrder_InsertsRow()
    {
        var connection = new FakeDbConnection();
        var repository = new OrderRepository(connection);

        repository.Save(new Order("order-123", "product-1", 5));

        Assert.Contains("INSERT INTO orders", connection.LastCommandText);
    }
}
```

**Correct:**

```csharp
public sealed class OrderRepositoryTests
{
    [Fact]
    public void Save_ValidOrder_IsReadBackEqual()
    {
        // Given
        using var database = TemporaryDatabase.Migrated();
        var repository = new OrderRepository(database.ConnectionString);

        // When
        repository.Save(new Order("order-123", "product-1", 5));

        // Then
        Assert.Equal(new Order("order-123", "product-1", 5), repository.FindById("order-123"));
    }
}

/// <summary>Database in a temporary folder, deleted when the test ends.</summary>
public sealed class TemporaryDatabase : IDisposable
{
    private readonly string _folder = Directory.CreateTempSubdirectory("tests-").FullName;

    public string ConnectionString => $"Data Source={Path.Combine(_folder, "test.db")}";

    public static TemporaryDatabase Migrated()
    {
        var database = new TemporaryDatabase();
        new Migrator(database.ConnectionString).Migrate();
        return database;
    }

    public void Dispose()
    {
        // The connection pool keeps the file open
        SqliteConnection.ClearAllPools();
        Directory.Delete(_folder, recursive: true);
    }
}
```

### What to Test in Persistence Adapters

1. **Round trip**: what is written is read back equal — types, nulls, dates, money
2. **Queries**: filters and ordering return the rows the contract promises
3. **Constraints and triggers**: the database rejects what the schema forbids
4. **Transactions**: a failure in the middle leaves nothing written
5. **Migrations**: they apply to an empty database and to one at the previous version

### Asserting on the Database

- Prefer reading back through the adapter's public API (`prefer-public-apis.md`)
- Query with SQL in the Then section only for effects the adapter does not expose — a row it never reads back, a trigger, the stored format of a column — and keep that query inside the test, where it can be read (`keep-cause-effect-clear.md`)
- For a rejection, assert the exception type and the stable part of the message — the text of the constraint or trigger — not the whole message

```csharp
// The trigger is the behavior: assert the rejection and its stable fragment
var actualException = Assert.Throws<SqliteException>(() =>
    Execute(connection, "UPDATE outbox SET payload = '{}';"));

Assert.Contains("event is immutable", actualException.Message);
```
