---
title: C# Test Template
impact: HIGH
impactDescription: ensures consistent test structure and keeps hosts and containers out of unit tests
tags: csharp, dotnet, tests, template, xunit, nunit, mstest, structure
---

## C# Test Template

Use the test framework the test project already references — xUnit, NUnit or MSTest — with consistent structure. The examples use xUnit; "NUnit and MSTest" below maps them. Keep the application host and the DI container out of unit tests.

### FORBIDDEN

- **FORBIDDEN** to build the application host or the DI container in unit tests (`Host.CreateApplicationBuilder`, `WebApplicationFactory`, `new ServiceCollection().BuildServiceProvider()`, the project's composition root) unless explicitly required by the rule file for that specific test type.

**Incorrect:**

```csharp
// Building the DI container for a unit test
public sealed class CalculatorServiceTests
{
    [Fact]
    public void Calculate_ValidInput_ReturnsResult()
    {
        using var provider = new ServiceCollection().AddCalculations().BuildServiceProvider();
        var calculatorService = provider.GetRequiredService<CalculatorService>();
        // ...
    }
}
```

**Correct:**

```csharp
namespace Shop.Calculations.Tests;

public sealed class CalculatorServiceTests
{
    private readonly FakeDependency _dependency = new();
    private readonly CalculatorService _calculatorService;

    public CalculatorServiceTests() => _calculatorService = new CalculatorService(_dependency);

    [Fact]
    public void Calculate_ValidInput_ReturnsResult()
    {
        // Given
        _dependency.Value = 10;

        // When
        var actualResult = _calculatorService.Calculate(5);

        // Then
        var expectedResult = 15;
        Assert.Equal(expectedResult, actualResult);
    }

    [Fact]
    public void Calculate_NegativeInput_ThrowsArgumentOutOfRangeException()
    {
        // Given-When-Then
        var actualException = Assert.Throws<ArgumentOutOfRangeException>(() => _calculatorService.Calculate(-1));
        Assert.Equal("input", actualException.ParamName);
    }
}
```

### Basic Template Structure

```csharp
namespace {TEST_PROJECT_NAMESPACE};

public sealed class {TestedClassName}Tests
{
    [Fact]
    public void {TestedMethod}_{GivenState}_{ExpectedOutcome}()
    {
        // Given
        // When
        // Then
    }

    [Fact]
    public void {TestedMethod}_AnotherState_ExpectedResult()
    {
        // Given-When-Then
    }
}
```

### Key Points

1. Place the test class in the test project of the SUT's project (`tests/{Project}.Tests/{TestedClassName}Tests.cs`), with the namespace the neighboring test files use
2. Construct the SUT with `new`, passing test doubles for its ports: the project's mocking library, or hand-written fakes when it has none (`domain-service-rules.md`, `argument-matching.md`)
3. Follow Given-When-Then pattern with comments
4. Mind the argument order of the assertions. xUnit `Assert.Equal(expected, actual)` and MSTest `Assert.AreEqual(expected, actual)` take the expected value first — the reverse of AssertJ's `assertThat(actual).isEqualTo(expected)` — and swapping them makes the failure message call the wrong value "expected". NUnit reads `Assert.That(actual, Is.EqualTo(expected))`
5. C# method names are PascalCase, so the format in `naming-conventions.md` becomes `{TestedMethod}_{GivenState}_{ExpectedOutcome}`: `CalculateTotal_EmptyList_ThrowsArgumentException`

### xUnit Equivalents of the General Rules

The general rules show their examples in Java. Translate them with this table:

| General rules (Java)                            | C# / xUnit v3                                                                                    |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `@Test`                                         | `[Fact]`                                                                                         |
| Parameterized test                              | `[Theory]` with `[InlineData]`, or `[MemberData]` with `TheoryData<...>` for non-constant values |
| `@BeforeEach`                                   | The test class constructor: xUnit creates a new instance for every test                          |
| `@AfterEach`                                    | `IDisposable.Dispose`, or `IAsyncLifetime.DisposeAsync`                                          |
| `@Disabled("reason")`                           | `[Fact(Skip = "reason")]`                                                                        |
| `assertThat(actual).isEqualTo(expected)`        | `Assert.Equal(expected, actual)`                                                                 |
| `assertThatThrownBy(...).isInstanceOf(T.class)` | `Assert.Throws<T>(...)`, exact type; `Assert.ThrowsAny<T>(...)` also accepts derived types       |
| `extracting(...).containsOnly(...)`             | `Assert.All(items, item => ...)`                                                                 |
| `hasSize(1)`                                    | `Assert.Single(items)`, which also returns the element                                           |
| `@Nullable`, `Optional`                         | A `T?` parameter under `<Nullable>enable</Nullable>`                                             |
| Mockito mock, `ArgumentCaptor`, `any()`         | A double that records its calls: a hand-written fake, or the mocking library's capture (`argument-matching.md`) |
| `Clock.fixed(...)`                              | A fake of the clock abstraction the SUT receives                                                 |

### NUnit and MSTest

The same concepts in the other two frameworks:

| Concept                      | xUnit                                          | NUnit                                       | MSTest                                                                     |
| ---------------------------- | ---------------------------------------------- | ------------------------------------------- | -------------------------------------------------------------------------- |
| Test method                  | `[Fact]`                                       | `[Test]`                                    | `[TestMethod]`, in a class marked `[TestClass]`                            |
| Parameterized test           | `[Theory]` with `[InlineData]` / `[MemberData]` | `[TestCase]` / `[TestCaseSource]`           | `[DataRow]` / `[DynamicData]`                                              |
| Per-test setup               | Constructor                                    | `[SetUp]`                                   | `[TestInitialize]`                                                         |
| Per-test cleanup             | `Dispose`                                      | `[TearDown]`                                | `[TestCleanup]`                                                            |
| Skipped test                 | `[Fact(Skip = "reason")]`                      | `[Ignore("reason")]`                        | `[Ignore("reason")]`                                                       |
| Equality                     | `Assert.Equal(expected, actual)`               | `Assert.That(actual, Is.EqualTo(expected))` | `Assert.AreEqual(expected, actual)`                                        |
| Exception of the exact type  | `Assert.Throws<T>(...)`                        | `Assert.Throws<T>(...)`                     | `Assert.ThrowsExactly<T>(...)` (MSTest 3.8+) or `Assert.ThrowsException<T>(...)` |

### Records and Value Equality

Records compare by value, so one `Assert.Equal` on a whole record checks every field:

```csharp
Assert.Equal(new Session(userId, "Carlos", Role.Manager), actualSession);
```

Use it when every field belongs to the behavior under test (`keep-tests-focused.md`). When only some do, assert those properties, or compare a tuple of them:

```csharp
Assert.Equal((Reason.UserDisabled, userId), (actualRejection.Reason, actualRejection.UserId));
```

### Nullable Reference Types

With `<Nullable>enable</Nullable>`, a non-nullable parameter already states that null is not accepted. Test a null argument only when the parameter is `T?`, or when the code has an explicit guard (`ArgumentNullException.ThrowIfNull`), which is a code branch of its own.

### Async Code

- Use `public async Task` test methods and `await Assert.ThrowsAsync<T>(...)`
- Never block with `.Result` or `.Wait()`
- In xUnit v3, pass `TestContext.Current.CancellationToken` where the API takes a cancellation token
