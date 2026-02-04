# Code Review Assistant

## Skill Name
Code Review Assistant

## Description
An agent skill that performs comprehensive code reviews, focusing on code quality, best practices, security vulnerabilities, and potential bugs. This skill provides actionable feedback with specific line references and improvement suggestions.

## Agent Type
code-review

## Use Cases
- Pre-commit code review
- Pull request review automation
- Code quality assessment
- Security vulnerability detection
- Best practices enforcement

## Prerequisites
- Access to the codebase being reviewed
- Understanding of the programming language(s) used

## Prompt Template

```
Please perform a comprehensive code review of the changes in this [pull request/commit/file].

Focus on:
1. **Code Quality**: Clean code principles, readability, maintainability
2. **Security**: Potential vulnerabilities, input validation, authentication issues
3. **Performance**: Inefficient algorithms, resource leaks, optimization opportunities
4. **Best Practices**: Language-specific conventions, design patterns
5. **Testing**: Test coverage, edge cases, test quality
6. **Documentation**: Comments, API documentation, inline explanations

For each issue found:
- Specify the file and line number
- Explain the problem clearly
- Provide a specific solution or improvement
- Rate severity: Critical, High, Medium, Low

Only report issues that genuinely matter. Skip style/formatting unless it impacts readability.
```

## Example Usage

### Input
```
Review the authentication middleware in src/auth/middleware.js
```

### Expected Output
```
**Code Review Results**

Critical Issues:
1. Line 45: Password comparison using == instead of secure comparison
   - Use crypto.timingSafeEqual() to prevent timing attacks
   
High Priority:
1. Line 78: JWT token not validated for expiration
   - Add expiration check before processing token

Medium Priority:
1. Line 23: Error messages leak sensitive information
   - Use generic error messages for authentication failures

Low Priority:
1. Line 10: Missing JSDoc documentation
   - Add function documentation for public API
```

## Tips and Best Practices
- Run the code review agent before requesting human reviews
- Combine with automated testing for comprehensive quality checks
- Use for educational purposes to learn best practices
- Re-run after making significant changes
- Focus on high-impact issues first

## Related Skills
- Security Scanner
- Test Coverage Analyzer
- Documentation Generator

## Version History
- v1.0 - Initial version (2026-02-04)

## Author
ManzarIMalik

## Tags
`code-review` `quality-assurance` `security` `best-practices`
