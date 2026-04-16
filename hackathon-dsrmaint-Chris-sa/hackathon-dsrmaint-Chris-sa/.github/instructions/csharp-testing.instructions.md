---
applyTo: "**/*Tests*/**/*.cs, **/*Test*/**/*.cs"
---

# Shamrock Foods C# Testing Conventions

## Unit Tests

- Do **not** emit `// Arrange`, `// Act`, or `// Assert` comments
- Copy existing style in nearby test files for method names and capitalization
- Include test cases for all critical paths of the application — cover both happy-path (expected success) and unhappy-path (validation failures, not found, unauthorized) scenarios
- Mock dependencies via interfaces — use constructor injection for testability
- Test authentication and authorization logic (verify role policies, unauthorized access)

## Integration Tests

- Use integration tests for API endpoint verification
- Test the full request/response pipeline including model binding and validation

## Cross-References

- **Interfaces:** All services and data access layers must have interfaces. See `csharp-conventions` for the interface convention.
