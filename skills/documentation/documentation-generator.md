# Documentation Generator

## Skill Name
Documentation Generator

## Description
An agent skill that automatically generates comprehensive documentation for code, APIs, and projects. It creates clear, well-structured documentation that follows best practices and includes examples.

## Agent Type
general-purpose

## Use Cases
- Generating API documentation
- Creating README files
- Writing inline code comments
- Producing user guides
- Building developer onboarding docs

## Prerequisites
- Access to the codebase
- Understanding of the project's purpose

## Prompt Template

```
Generate comprehensive documentation for [component/API/project]:

**Requirements:**
1. **Overview**: Brief description and purpose
2. **Installation**: Setup and installation steps
3. **Usage**: How to use with clear examples
4. **API Reference**: All public methods/endpoints with parameters and returns
5. **Examples**: Real-world usage examples
6. **Configuration**: Available options and settings
7. **Troubleshooting**: Common issues and solutions
8. **Contributing**: How to contribute (if applicable)

**Format:**
- Use clear, concise language
- Include code examples with syntax highlighting
- Add diagrams where helpful (using ASCII or mermaid)
- Follow markdown best practices
- Include table of contents for longer docs

**Style:**
- Write for the target audience: [developers/users/both]
- Use active voice
- Be specific and actionable
- Include "why" not just "how"
```

## Example Usage

### Input
```
Generate API documentation for the UserService class in src/services/user.js
```

### Expected Output
```
# UserService API Documentation

## Overview
UserService handles all user-related operations including authentication, profile management, and user data persistence.

## Installation
```javascript
import { UserService } from './services/user';

const userService = new UserService(database);
```

## API Reference

### `createUser(userData)`
Creates a new user account.

**Parameters:**
- `userData` (Object): User information
  - `email` (string, required): User's email address
  - `password` (string, required): User's password (will be hashed)
  - `name` (string, optional): User's display name

**Returns:**
- Promise<User>: Newly created user object (password excluded)

**Throws:**
- `ValidationError`: If email is invalid or already exists
- `DatabaseError`: If database operation fails

**Example:**
```javascript
const user = await userService.createUser({
  email: 'user@example.com',
  password: 'SecurePass123!',
  name: 'John Doe'
});
```

### `authenticate(email, password)`
Authenticates a user and returns a JWT token.

**Parameters:**
- `email` (string): User's email
- `password` (string): User's password

**Returns:**
- Promise<AuthResult>: Object containing token and user info

**Example:**
```javascript
const result = await userService.authenticate(
  'user@example.com',
  'SecurePass123!'
);
console.log(result.token); // JWT token
```

## Error Handling
All methods throw typed errors that can be caught and handled:

```javascript
try {
  await userService.createUser(userData);
} catch (error) {
  if (error instanceof ValidationError) {
    // Handle validation error
  }
}
```

## Best Practices
- Always validate input before calling UserService methods
- Store tokens securely (httpOnly cookies recommended)
- Implement rate limiting for authentication endpoints
- Use HTTPS in production
```

## Tips and Best Practices
- Generate docs incrementally as you write code
- Update docs when code changes
- Include both happy path and error scenarios in examples
- Use consistent formatting throughout
- Add version numbers to track documentation changes

## Related Skills
- Code Review Assistant
- API Schema Generator
- Tutorial Writer

## Version History
- v1.0 - Initial version (2026-02-04)

## Author
ManzarIMalik

## Tags
`documentation` `api-docs` `code-comments` `developer-experience`
