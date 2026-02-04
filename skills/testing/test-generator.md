# Test Generator

## Skill Name
Test Generator

## Description
An agent skill that creates comprehensive test suites for code, including unit tests, integration tests, and edge case coverage. Follows testing best practices and matches the project's existing test patterns.

## Agent Type
task

## Use Cases
- Generating unit tests for new code
- Adding missing test coverage
- Creating integration tests
- Testing edge cases
- Generating test fixtures and mocks

## Prerequisites
- Access to the code to be tested
- Understanding of the testing framework used (Jest, pytest, etc.)
- Knowledge of the code's expected behavior

## Prompt Template

```
Generate comprehensive tests for [component/function/class]:

**Test Requirements:**
1. **Unit Tests**: Test individual functions/methods in isolation
2. **Integration Tests**: Test component interactions
3. **Edge Cases**: Boundary conditions, null/undefined, empty inputs
4. **Error Cases**: Invalid inputs, error handling, exceptions
5. **Happy Path**: Normal usage scenarios
6. **Mocking**: Mock external dependencies appropriately

**Coverage Goals:**
- Aim for >80% code coverage
- Cover all public methods/functions
- Test all conditional branches
- Validate error handling

**Test Structure:**
- Follow existing test patterns in the project
- Use descriptive test names: "should [expected behavior] when [condition]"
- Use AAA pattern: Arrange, Act, Assert
- Keep tests independent and isolated
- Use appropriate assertions

**Include:**
- Setup/teardown when needed
- Test fixtures and mock data
- Clear comments for complex test scenarios
```

## Example Usage

### Input
```
Generate tests for the calculateDiscount function in src/utils/pricing.js

Function signature:
calculateDiscount(price: number, discountPercent: number): number
```

### Expected Output
```javascript
import { calculateDiscount } from '../src/utils/pricing';

describe('calculateDiscount', () => {
  describe('happy path', () => {
    it('should calculate 10% discount correctly', () => {
      const result = calculateDiscount(100, 10);
      expect(result).toBe(90);
    });

    it('should calculate 50% discount correctly', () => {
      const result = calculateDiscount(200, 50);
      expect(result).toBe(100);
    });

    it('should handle decimal prices', () => {
      const result = calculateDiscount(99.99, 20);
      expect(result).toBeCloseTo(79.99, 2);
    });
  });

  describe('edge cases', () => {
    it('should return original price when discount is 0', () => {
      const result = calculateDiscount(100, 0);
      expect(result).toBe(100);
    });

    it('should return 0 when discount is 100%', () => {
      const result = calculateDiscount(100, 100);
      expect(result).toBe(0);
    });

    it('should handle very small prices', () => {
      const result = calculateDiscount(0.01, 50);
      expect(result).toBeCloseTo(0.005, 3);
    });

    it('should handle very large prices', () => {
      const result = calculateDiscount(1000000, 25);
      expect(result).toBe(750000);
    });
  });

  describe('error cases', () => {
    it('should throw error for negative price', () => {
      expect(() => calculateDiscount(-100, 10))
        .toThrow('Price cannot be negative');
    });

    it('should throw error for negative discount', () => {
      expect(() => calculateDiscount(100, -10))
        .toThrow('Discount percent cannot be negative');
    });

    it('should throw error for discount over 100%', () => {
      expect(() => calculateDiscount(100, 150))
        .toThrow('Discount percent cannot exceed 100');
    });

    it('should throw error for non-number price', () => {
      expect(() => calculateDiscount('100', 10))
        .toThrow('Price must be a number');
    });

    it('should throw error for NaN inputs', () => {
      expect(() => calculateDiscount(NaN, 10))
        .toThrow('Invalid price value');
    });
  });

  describe('precision', () => {
    it('should round to 2 decimal places', () => {
      const result = calculateDiscount(99.99, 33.33);
      expect(result).toBeCloseTo(66.66, 2);
    });

    it('should not have floating point precision issues', () => {
      const result = calculateDiscount(0.1, 10);
      expect(result).toBeCloseTo(0.09, 2);
    });
  });
});
```

## Tips and Best Practices
- Generate tests immediately after writing code
- Use test-driven development (TDD) when possible
- Mock external dependencies (APIs, databases, file system)
- Test one thing per test
- Use factory functions for test data generation
- Keep tests fast and independent
- Use meaningful test descriptions
- Group related tests with describe blocks

## Related Skills
- Code Coverage Analyzer
- Mock Data Generator
- Test Refactoring Assistant

## Version History
- v1.0 - Initial version (2026-02-04)

## Author
ManzarIMalik

## Tags
`testing` `unit-tests` `tdd` `quality-assurance` `automation`
