---
title: Domain and Service Unit Test Rules
impact: HIGH
impactDescription: ensures fast, isolated unit tests for business logic
tags: csharp, tests, unit, domain, service, use-case, fakes
---

## Domain and Service Unit Test Rules

Use test doubles for unit testing services, use cases and domain logic: the mocking library the project already uses, or hand-written fakes when it has none. The examples use fakes. Keep tests fast and isolated.

### Rules

- Construct the SUT with `new`, passing a test double for each port (interface) it receives
- Do NOT start a host, a DI container, a database or any other I/O for unit tests
- Fake external dependencies, not the system under test
- Never fake simple value objects: records and value types are used as they are
- Assert on what the fakes recorded after the call (`argument-matching.md`) — a fake that
  checks its own arguments fails inside the SUT, far from the assertion that should have
  named the expected value
- Fake the clock and the ID generator the SUT receives: reading the real time
  (`DateTimeOffset.Now`) or creating `Guid.NewGuid()` in the code path makes the test
  non-deterministic (`general-principles.md`)

**Incorrect:**

```csharp
// Starting the DI container for unit test - slow!
public sealed class OrderServiceTests
{
    [Fact]
    public void CalculateTotal_ValidOrder_ReturnsSum()
    {
        using var provider = new ServiceCollection().AddOrders().BuildServiceProvider();
        var orderService = provider.GetRequiredService<OrderService>();
        // ...
    }
}

// Faking value objects - unnecessary
[Fact]
public void ProcessOrder_ValidOrder_CalculatesCorrectly()
{
    IProduct product = new FakeProduct { Price = 10_000, Name = "Test" };
    // ...
}
```

**Correct:**

```csharp
namespace Shop.Orders.Tests;

public sealed class OrderServiceTests
{
    private readonly FakeOrderRepository _orderRepository = new();
    private readonly FakePaymentGateway _paymentGateway = new();
    private readonly FakeNotifier _notifier = new();
    private readonly OrderService _orderService;

    public OrderServiceTests() =>
        _orderService = new OrderService(_orderRepository, _paymentGateway, _notifier);

    [Fact]
    public void CreateOrder_ValidRequest_SavesAndReturnsOrder()
    {
        // Given
        var request = new OrderRequest("product-1", 5);
        _orderRepository.NextId = "order-123";

        // When
        var actualOrder = _orderService.CreateOrder(request);

        // Then
        Assert.Equal("order-123", actualOrder.Id);

        var savedOrder = Assert.Single(_orderRepository.Saved);
        Assert.Equal("product-1", savedOrder.ProductId);
        Assert.Equal(5, savedOrder.Quantity);
    }

    [Fact]
    public void ProcessPayment_ValidOrder_ChargesTheOrderTotal()
    {
        // Given
        var order = new Order("order-123", "product-1", 5) { TotalInCents = 50_000 };
        _paymentGateway.Approves = true;

        // When
        var actualResult = _orderService.ProcessPayment(order);

        // Then
        Assert.True(actualResult);
        Assert.Equal([new Payment("order-123", 50_000)], _paymentGateway.Payments);
    }

    [Fact]
    public void ProcessPayment_PaymentFails_ThrowsPaymentException()
    {
        // Given
        var order = new Order("order-123", "product-1", 5) { TotalInCents = 50_000 };
        _paymentGateway.Approves = false;

        // When-Then
        var actualException = Assert.Throws<PaymentException>(() => _orderService.ProcessPayment(order));
        Assert.Contains("Payment failed", actualException.Message);
    }

    [Fact]
    public void CalculateTotal_MultipleProducts_ReturnsSumOfPrices()
    {
        // Given - use real value objects
        var order = new Order([new Product("A", 5_000), new Product("B", 10_000)]);

        // When
        var actualTotal = _orderService.CalculateTotal(order);

        // Then
        Assert.Equal(15_000, actualTotal);
    }
}
```

### What to Fake vs What to Use Real Objects

**Fake:**
- Repositories / DAOs (the port, not the database)
- External service clients
- Message and outbox writers
- Clock and ID generator
- Any I/O operation

**Use Real Objects:**
- DTOs / records / value types
- Domain entities (in most cases)
- Static helpers and utility classes
- Mappers (usually)

### Verification Patterns

```csharp
// Verify method was called with the data that matters
var savedOrder = Assert.Single(_orderRepository.Saved);
Assert.Equal("product-1", savedOrder.ProductId);

// Verify method was NOT called
Assert.Empty(_notifier.Sent);

// Verify call count
Assert.Equal(2, _orderRepository.FindByIdCalls.Count);

// Verify no more interactions: compare the whole recording
Assert.Equal([new Payment("order-123", 50_000)], _paymentGateway.Payments);
```
