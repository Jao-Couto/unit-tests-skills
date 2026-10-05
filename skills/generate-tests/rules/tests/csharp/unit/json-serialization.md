---
title: JSON Serialization in Tests
impact: HIGH
impactDescription: prevents test fragility and ensures explicit test data
tags: csharp, tests, json, serialization, system-text-json, contracts
---

## JSON Serialization in Tests

Use explicit JSON — a string literal in the test, or a reference file reviewed by a person — instead of runtime serializers to build the expected data.

### Rules

- **DO NOT** call the serializer to produce the expected JSON or the input JSON (`JsonSerializer.Serialize(expectedObject)`, `JsonConvert.SerializeObject(...)`)
- You **MUST** use explicit JSON: a raw string literal (`"""`) in the test, or a reference file in the test project
- Serializing the SUT's own output is fine — it is the thing under test. The side that must be explicit is the expected one

**Incorrect:**

```csharp
[Fact]
public void Serialize_OrderCreated_WritesOrderFields()
{
    var orderCreated = new OrderCreatedV1("order-123", 5);

    var actualJson = JsonSerializer.Serialize(orderCreated, ContractJson.Options);

    // Expected built by the same serializer - passes whatever it writes
    Assert.Equal(JsonSerializer.Serialize(new OrderCreatedV1("order-123", 5), ContractJson.Options), actualJson);
}

[Fact]
public void Deserialize_OrderCreated_ReadsOrderFields()
{
    // Input produced by the serializer - hides the format being read
    var json = JsonSerializer.Serialize(new OrderCreatedV1("order-123", 5), ContractJson.Options);

    var actualOrder = JsonSerializer.Deserialize<OrderCreatedV1>(json, ContractJson.Options);
    // ...
}
```

**Correct:**

```csharp
[Fact]
public void Serialize_OrderCreated_WritesOrderFields()
{
    // Given
    var orderCreated = new OrderCreatedV1("order-123", 5);

    // When
    var actualJson = JsonNode.Parse(JsonSerializer.Serialize(orderCreated, ContractJson.Options))!;

    // Then - explicit field names, independent of order and whitespace
    Assert.Equal("order-123", (string?)actualJson["order_id"]);
    Assert.Equal(5, (int?)actualJson["quantity"]);
}

[Fact]
public void Deserialize_OrderCreated_ReadsOrderFields()
{
    // Given - explicit JSON literal
    var json = """
        {
            "order_id": "order-123",
            "quantity": 5
        }
        """;

    // When
    var actualOrder = JsonSerializer.Deserialize<OrderCreatedV1>(json, ContractJson.Options);

    // Then
    Assert.Equal(new OrderCreatedV1("order-123", 5), actualOrder);
}
```

### Published Contracts: Reference Files

When the JSON is a published contract — other processes read it, and a change breaks them —
the exact text is the behavior under test. Compare the whole document with a reference file
in the test project instead of field by field; `what-makes-good-test.md` ("Resilience")
does not apply to a format that is not allowed to change.

- The reference file is the explicit JSON this rule requires. A person writes or reviews it
- Look for the project's reference-file helper before writing a contract test, and use it
- When the helper creates a missing reference file and fails the test, the generated file
  is **not** a pass. Show its content to the user for review; never re-run the test to
  green a file the test itself just wrote
- Never edit an existing reference file to make a failing test pass. The failure means the
  contract changed: report it

### Benefits

1. **Readability** - Expected data is visible directly in test
2. **Determinism** - No dependency on serializer configuration
3. **Debugging** - Easy to see what's being tested
4. **Maintenance** - Changes to serializer settings don't break tests silently
