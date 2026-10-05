---
title: Argument Matching with Test Doubles
impact: HIGH
impactDescription: ensures meaningful verification of method arguments
tags: csharp, tests, fakes, moq, nsubstitute, verification
---

## Argument Matching with Test Doubles

Assert on the arguments a test double actually received instead of only checking that it was called. The examples use hand-written fakes; "With a Mocking Library" below applies the same rule to Moq and NSubstitute.

### Rules

- A fake records every call it receives, as a list of the arguments or of a small record per
  call. In the Then section, assert the fields that matter on the recorded value — a call
  count or a `WasCalled` flag asserts nothing about the data
- A fake's canned answer stays loose: it returns the configured value for any input. It
  decides what the SUT receives, it makes no assertion. See "Canned Answers and Recorded
  Calls Are Different Positions" below

**Incorrect:**

```csharp
[Fact]
public void CreateOrder_ValidRequest_CallsRepository()
{
    // Counting calls - doesn't verify actual data passed
    _orderService.CreateOrder(new OrderRequest("product-1", 5));

    Assert.Equal(1, _orderRepository.SaveCount);
}

[Fact]
public void NotifyUser_ValidUser_SendsEmail()
{
    // A flag hides what's actually being sent
    _userService.NotifyUser(user);

    Assert.True(_emailSender.WasCalled);
}
```

**Correct:**

```csharp
[Fact]
public void CreateOrder_ValidRequest_SavesCorrectOrder()
{
    // Given
    var request = new OrderRequest("product-1", 5);

    // When
    _orderService.CreateOrder(request);

    // Then
    var actualOrder = Assert.Single(_orderRepository.Saved);
    Assert.Equal("product-1", actualOrder.ProductId);
    Assert.Equal(5, actualOrder.Quantity);
}

[Fact]
public void NotifyUser_ValidUser_SendsCorrectEmail()
{
    // Given
    var user = new User("john@test.com", "John");

    // When
    _userService.NotifyUser(user);

    // Then
    var actualMessage = Assert.Single(_emailSender.Sent);
    Assert.Equal("john@test.com", actualMessage.To);
    Assert.Contains("John", actualMessage.Subject);
}
```

### Canned Answers and Recorded Calls Are Different Positions

This rule constrains **verification**, not the canned answer. A canned answer answers "what
should the fake return"; a recorded call answers "what did the code actually pass". Only
the second is an assertion, so only the second has to name real values.

A fake that returns the same value for any input, paired with an assertion on what it
recorded, is the correct combination — the answer stays loose so the call reaches the code
under test, and the recorded value does the checking:

```csharp
// Given
_orderRepository.NextId = "order-123";   // returned for any order saved

// When
_orderService.CreateOrder(new OrderRequest("product-1", 5));

// Then — the assertion lives here, on the real recorded value
Assert.Equal("product-1", Assert.Single(_orderRepository.Saved).ProductId);
```

Keep the check out of the fake. A fake that answers only when the argument matches returns
the default when the code passes something else, so the test fails late — at a
`NullReferenceException` or a wrong result — with an error that points away from the call
that went wrong. A fake that asserts inside its method fails inside the SUT, and the Then
section no longer says what was expected. The assertion on the recorded value fails at the
right line and names the expected value.

### When a Call Count Is Acceptable

Check only that a call happened, or how many times, without asserting its fields, for:
- Primitive arguments where the exact value doesn't matter
- Simple types (`string`, `int`) when focus is on other behavior
- Verify that method was called at all (existence check), or never

```csharp
// OK - verifying call count, not data
Assert.Equal(3, _auditLog.Entries.Count);

// OK - never called
Assert.Empty(_notifier.Sent);
```

### Writing Fakes

```csharp
// Records each call
public sealed class FakeEmailSender : IEmailSender
{
    public List<EmailMessage> Sent { get; } = [];

    public void Send(EmailMessage message) => Sent.Add(message);
}

// Several arguments per call: record them as a record; the canned answer is a property
public sealed class FakePaymentGateway : IPaymentGateway
{
    public bool Approves { get; set; } = true;

    public List<Payment> Payments { get; } = [];

    public bool Charge(string orderId, long amountInCents)
    {
        Payments.Add(new Payment(orderId, amountInCents));
        return Approves;
    }
}

public sealed record Payment(string OrderId, long AmountInCents);

// Verify multiple calls
Assert.Equal(
    [new Payment("order-1", 1_000), new Payment("order-2", 2_500)],
    _paymentGateway.Payments);
```

- Search the test project for an existing fake of the interface before writing one, and reuse it
- Keep fakes free of logic beyond recording the call and returning the canned answer (`no-logic-in-tests.md`)
- Put a new fake where the test project keeps its shared fakes; write it in the test file only when no such place exists
- Do not add a mocking library (Moq, NSubstitute, FakeItEasy) to a project that has none. If the project already uses one, follow it (`existing-test-awareness.md`)

### With a Mocking Library

When the project's tests already use a mocking library, the same rule applies: capture what
the double received, then assert its fields in the Then section. Matching the argument inside
the verification — `Verify(s => s.Send(It.Is<EmailMessage>(m => m.To == "john@test.com")))`
or `Received().Send(Arg.Is<EmailMessage>(...))` — is the C# form of `any()` with a predicate:
when it fails, the message names the predicate, not the value that arrived.

```csharp
// Moq: a loose setup that captures every call
var sent = new List<EmailMessage>();
var emailSender = new Mock<IEmailSender>();
emailSender.Setup(s => s.Send(It.IsAny<EmailMessage>())).Callback<EmailMessage>(sent.Add);

// NSubstitute: Arg.Do captures every call
var sent = new List<EmailMessage>();
var emailSender = Substitute.For<IEmailSender>();
emailSender.Send(Arg.Do<EmailMessage>(sent.Add));

// Then — identical for both
var actualMessage = Assert.Single(sent);
Assert.Equal("john@test.com", actualMessage.To);
```

The rules on canned answers and call counts above hold unchanged: keep setups loose
(`It.IsAny`, `Arg.Any`), and use `Verify(..., Times.Never)` or `DidNotReceive()` only for
the existence checks listed in "When a Call Count Is Acceptable".
