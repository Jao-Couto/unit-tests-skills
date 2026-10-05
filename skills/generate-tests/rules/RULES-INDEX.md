# Rules Index

Which language rules `generate-tests` applies, by language and code type. The general rules
(`general/`) and the post-generation rules (`post-generation/`) apply to every language;
`SKILL.md` lists them.

Detect the language with `general/technology-stack-detection.md`, then apply its section:
the rule file mapped to the code type, plus every file the section always applies. A
language with no section here has no rules yet — follow "Adding Rules for a New Language"
in `SKILL.md`.

## Java

JUnit 5, Mockito, AssertJ.

| Code type                            | Rule file                                                                                                        |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| Controller                           | `java/unit/controller-test-rules.md` (use `@WebMvcTest`, MockMvc patterns)                                       |
| Service / Domain logic               | `java/unit/domain-service-rules.md` (use `@ExtendWith(MockitoExtension.class)`, Mockito patterns)                |
| Repository / Messaging / Other types | `java/unit/domain-service-rules.md` as baseline; inform the user that type-specific rules are not yet available |

Always apply, regardless of code type:
- `java/unit/java-test-template.md` - Basic template, FORBIDDEN annotations
- `java/unit/argument-matching.md` - Use ArgumentCaptor, not any()
- `java/unit/json-serialization.md` - Use explicit JSON literals

When the test verifies log output:
- `java/unit/logging-rules.md` - OutputCaptureExtension for logs

## C#

xUnit, NUnit or MSTest; the project's mocking library, or hand-written fakes when it has none.

| Code type                                                      | Rule file                                                                                                          |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Repository / persistence adapter (issues SQL, runs migrations) | `csharp/unit/repository-test-rules.md` (real database created for the test)                                        |
| Service / Use case / Domain logic                              | `csharp/unit/domain-service-rules.md` (SUT built with `new`, test doubles for its ports)                           |
| Controller / Messaging / Other types                           | `csharp/unit/domain-service-rules.md` as baseline; inform the user that type-specific rules are not yet available |

Always apply, regardless of code type:
- `csharp/unit/csharp-test-template.md` - Basic template, framework equivalents, FORBIDDEN host and container
- `csharp/unit/argument-matching.md` - Assert what a double recorded, not only that it was called
- `csharp/unit/json-serialization.md` - Use explicit JSON literals and reviewed reference files

## Section Format

Each language section names its stack in one line, maps code types to rule files in a
table, and lists the files that always apply. A code type the table does not name takes the
section's baseline, and the user is told that type-specific rules are not yet available.
