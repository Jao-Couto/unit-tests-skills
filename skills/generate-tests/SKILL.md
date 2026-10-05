---
name: generate-tests
description: "Use when the user asks to generate, create, write, or add unit tests for existing code, or to cover a class, method, or file with tests — including Java (JUnit 5, Mockito, AssertJ) and C# (xUnit, NUnit, MSTest) targets, and other languages, for which it first drafts rules for approval. Not for analysis-only requests that stop at listing test cases."
allowed-tools: Read, Write, Glob, Grep, Bash
argument-hint: "<file-path-or-class>"
---

# Generate Tests Skill

You will analyze code and generate high-quality unit tests for a given target.

**Target to test:** $ARGUMENTS

## Quality Standards

- Take your time to analyze the code thoroughly before generating test cases.
- Quality is more important than speed — read all relevant source files and rules carefully.
- Do not skip any step in the workflow below. Every step exists for a reason.
- Do not take shortcuts with test data — read the actual classes to use correct constructors and fields.

---

## Instructions

### Step 1: Read Rules and Analyze Context

1. **Read the relevant rules** from `./rules/tests/` based on the target's language and code type (see Rules Reference below). `./rules/RULES-INDEX.md` maps each language's code types to its rule files; if it has no section for the target's language, go to "Adding Rules for a New Language" below before anything else
2. **Read the target** source file/class/method
3. **Read dependencies**: Follow imports to read DTOs, entities, enums, custom exceptions, and other types referenced by the target (as specified in `code-context-analysis` rule)
4. **Check for existing tests**: Search for `{ClassName}Test` or `{ClassName}Tests` in the test directory (as specified in `existing-test-awareness` rule)
   - If found, read fully — you will add missing tests to it, not create a new file
   - If not found, scan 2-3 neighboring test classes to learn project conventions

### Step 2: Establish the Test Case List

**If a test case list for this target is already present** — `generate-test-cases` ran
first, whether the user invoked it or the agent did — that list is the plan. Generate
from it. Do not re-analyze the target from scratch; where your reading of the code
differs, name the cases you add or drop and why, so the change to the plan is visible.

**If there is no such list**, produce one here before writing any test code:

1. Analyze ALL code branches, including:
   - Success paths
   - Error/exception paths
   - Validation logic
   - Private/protected methods called by the target
   - Security annotations (if present)
2. Apply the INCLUDE/EXCLUDE rules strictly
3. Output the list in the format below

Either way, continue straight to Step 3. The list is printed so the user can audit the
plan against the generated tests afterwards; the run stays unattended end to end.

#### Test Case Output Format

```
## Test Cases for {ClassName}.{methodName}

### 1. {testMethodName}
- **Given:** {preconditions/input state}
- **When:** {action being tested}
- **Then:** {expected outcome}
- **Code branch:** {which code path this covers}

### 2. {testMethodName}
...
```

#### Naming Convention
Test method name format: `{testedMethod}_{givenState}_{expectedOutcome}`

Examples:
- `calculateTotal_validProducts_returnsSum`
- `calculateTotal_emptyList_throwsIllegalArgumentException`
- `getUser_unauthorized_returns401`

Write the name in the casing the target language gives method names; the language's test
template says which (C#: `CalculateTotal_ValidProducts_ReturnsSum`).

### Step 3: Generate Test Code

1. Determine code type and apply the matching rules from the language's section in `./rules/RULES-INDEX.md`:
   - The rule file the section maps to the code type; a code type it does not name takes the section's baseline, and you inform the user that type-specific rules are not yet available
   - Every file the section lists to always apply, regardless of code type
2. If an existing test class was found in Step 1, add new test methods to it (do not create a duplicate file)
3. Generate tests following all rules and the test cases from Step 2
4. Create or update the test file using the Write tool

### Step 4: Verify Compilation and Execution

1. Run compilation and fix any issues (max 5 attempts — see `compilation-verification.md`)
2. Run the generated test class to verify all tests pass (see `test-execution-verification.md`)
3. Fix any failing tests by changing the test — production code stays as it is
4. If a test resists fixing after 3 attempts, keep it in the file: mark it skipped with
   the reason (`@Disabled`, `[Fact(Skip = ...)]`, `[Ignore]`) and report it (see `test-execution-verification.md`)

---

## Adding Rules for a New Language

Run this when `./rules/RULES-INDEX.md` has no section for the target's language. The rules
you add live in this skill, so they apply to every project that uses it — not only to the
current one.

1. **Learn the stack** from the project: its build and package files
   (`technology-stack-detection.md`), the test framework, assertion library, mocking library
   and test runner its test projects reference, and 2-3 existing tests.
2. **Draft `rules/tests/{language}/unit/`** in this skill's own directory
   (`${CLAUDE_SKILL_DIR}/rules/tests/`), never inside the project under test. Use
   `java/unit/` and `csharp/unit/` as models and keep their shape:
   - A test template named after the language, as `csharp/unit/csharp-test-template.md` is:
     structure, FORBIDDEN setups, and a table translating the Java terms the general rules use
   - `domain-service-rules.md`: test doubles for services and domain logic
   - `argument-matching.md`: assert what a double received, not only that it was called
   - `json-serialization.md`: explicit JSON, when the ecosystem serializes JSON
   - One file per code type whose tests need the framework or real I/O, as
     `controller-test-rules.md` and `repository-test-rules.md` do — only for code types the
     project has
3. **Draft the language's section** for `./rules/RULES-INDEX.md`, and the language's compile
   and single-class test commands for the two `post-generation/` tables when they are missing.
4. **Describe the ecosystem, not the project.** A choice only this project makes — its own
   helpers, its domain rules, a library it forbids — belongs in the project's `.claude/rules/`,
   not here.
5. **Stop and show the drafts** — each file's path and full content — and ask the user to
   approve them. This is the one point where this skill waits for the user, because the rules
   change every project's tests. Write nothing until they approve; a reply asking for changes
   means a new draft.
6. **After approval**, write the files, then run `${CLAUDE_SKILL_DIR}/../../scripts/validate-rules.sh`
   when it exists and fix what it reports. If this skill's directory is a git checkout, tell
   the user the new files are there to commit. Then continue from Step 1.

---

## Troubleshooting

### Target file not found
If the specified target does not exist, inform the user with the exact path you searched and ask for clarification.

### Unsupported language
If the target code is in a language without a section in `./rules/RULES-INDEX.md`, follow "Adding Rules for a New Language". If the user declines the drafted rules, apply only the general rules and inform the user that language-specific conventions may need manual review.

### Compilation keeps failing
If compilation fails after 5 attempts:
1. Stop and show the user the remaining errors
2. Suggest possible causes (missing dependencies, incompatible versions)
3. Ask the user to resolve the build issue before continuing

### Tests fail due to production code behavior
If tests fail because the production code behaves differently than expected:
1. Do NOT modify production code
2. Fix the test to match actual behavior
3. If the behavior seems like a bug, add a comment: `// NOTE: current behavior may be a bug — {description}`

---

## Example

```
User says: "/generate-tests src/main/java/com/example/service/OrderService.java"

Step 1: Agent reads rules, reads OrderService.java, reads OrderRequest.java,
        Order.java, OrderRepository.java (dependencies), checks for
        existing OrderServiceTest.java

Step 2: Agent outputs 7 test cases covering:
        - createOrder success path
        - createOrder with invalid request (validation)
        - processPayment success
        - processPayment failure
        - calculateTotal with products
        - calculateTotal with empty list
        - cancelOrder for non-existent order

Step 3: Agent generates OrderServiceTest.java with @ExtendWith(MockitoExtension.class),
        mocked repository and payment service, 7 test methods.

Step 4: Agent runs `mvn test -Dtest=OrderServiceTest -q`, all tests pass.

Result: Complete test file delivered with 7 passing tests.
```

---

## Rules Reference

**CRITICAL: You MUST read and apply all relevant rules from the `./rules/tests/` directory.**

> **Maintenance note:** General rules in `./rules/tests/general/` are shared with the `generate-test-cases` skill (which has copies in `rules/general/`). When updating rules, keep both locations in sync.

### General Rules (Always Apply)
- `general/test-case-generation-strategy.md` - INCLUDE/EXCLUDE criteria
- `general/naming-conventions.md` - Test naming format
- `general/general-principles.md` - Core testing principles (Given-When-Then, actual/expected)
- `general/technology-stack-detection.md` - Detect language and framework
- `general/what-makes-good-test.md` - Clarity, Completeness, Conciseness, Resilience
- `general/cleanly-create-test-data.md` - Use helpers and builders for test data
- `general/keep-cause-effect-clear.md` - Effects follow causes immediately
- `general/no-logic-in-tests.md` - KISS > DRY, avoid logic in assertions
- `general/keep-tests-focused.md` - One scenario per test
- `general/test-behaviors-not-methods.md` - Separate tests for behaviors
- `general/verify-relevant-arguments-only.md` - Only verify relevant mock arguments
- `general/prefer-public-apis.md` - Test public APIs over private methods
- `general/existing-test-awareness.md` - Check for existing tests, match project conventions
- `general/code-context-analysis.md` - Read dependencies before writing tests

### Language Rules
- `./rules/RULES-INDEX.md` - Rule files per language and code type: Java (`java/unit/`) and C# (`csharp/unit/`)

### Post-Generation
- `post-generation/compilation-verification.md` - Verify compilation
- `post-generation/test-execution-verification.md` - Verify tests pass
