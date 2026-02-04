# Refactoring Assistant

## Skill Name
Refactoring Assistant

## Description
An agent skill that helps refactor code to improve maintainability, readability, and performance while preserving functionality. Suggests design patterns, removes code smells, and modernizes code.

## Agent Type
general-purpose

## Use Cases
- Removing code duplication
- Extracting reusable functions/components
- Applying design patterns
- Modernizing legacy code
- Improving code organization
- Reducing complexity

## Prerequisites
- Access to the code to be refactored
- Understanding of the code's intended functionality
- Test coverage (recommended)

## Prompt Template

```
Refactor the following code to improve [quality/readability/performance/maintainability]:

**Code Location:** [file path or code snippet]

**Refactoring Goals:**
1. **Primary Goal**: [What's the main improvement needed?]
2. **Constraints**: [What should NOT change? e.g., public API, behavior]
3. **Focus Areas**:
   - Remove duplication
   - Extract functions/methods
   - Apply design patterns
   - Improve naming
   - Reduce complexity
   - Enhance testability

**Requirements:**
- Preserve all existing functionality
- Maintain backward compatibility (unless specified otherwise)
- Add comments explaining complex refactoring
- Follow existing code style and conventions
- Keep changes minimal and focused

**Output:**
1. Refactored code with clear improvements
2. Explanation of changes made
3. Before/after comparison for key sections
4. Any potential risks or breaking changes
5. Suggested tests to verify refactoring
```

## Example Usage

### Input
```
Refactor this user authentication function to remove duplication and improve readability:

```javascript
function loginUser(email, password) {
  if (!email || email.trim() === '') {
    return { success: false, error: 'Email is required' };
  }
  if (!email.includes('@')) {
    return { success: false, error: 'Invalid email format' };
  }
  if (!password || password.trim() === '') {
    return { success: false, error: 'Password is required' };
  }
  if (password.length < 8) {
    return { success: false, error: 'Password too short' };
  }
  
  const user = database.findUser(email);
  if (!user) {
    return { success: false, error: 'User not found' };
  }
  
  const isValid = bcrypt.compare(password, user.passwordHash);
  if (!isValid) {
    return { success: false, error: 'Invalid password' };
  }
  
  const token = jwt.sign({ userId: user.id }, SECRET);
  return { success: true, token: token, user: user };
}
```
```

### Expected Output
```javascript
// Refactored code with extracted validation functions

class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = 'ValidationError';
  }
}

class AuthenticationError extends Error {
  constructor(message) {
    super(message);
    this.name = 'AuthenticationError';
  }
}

// Extracted validation functions
function validateEmail(email) {
  if (!email?.trim()) {
    throw new ValidationError('Email is required');
  }
  if (!email.includes('@')) {
    throw new ValidationError('Invalid email format');
  }
  return email.trim();
}

function validatePassword(password) {
  if (!password?.trim()) {
    throw new ValidationError('Password is required');
  }
  if (password.length < 8) {
    throw new ValidationError('Password must be at least 8 characters');
  }
  return password;
}

async function findUserByEmail(email) {
  const user = await database.findUser(email);
  if (!user) {
    throw new AuthenticationError('User not found');
  }
  return user;
}

async function verifyPassword(password, passwordHash) {
  const isValid = await bcrypt.compare(password, passwordHash);
  if (!isValid) {
    throw new AuthenticationError('Invalid password');
  }
}

function generateAuthToken(userId) {
  return jwt.sign({ userId }, SECRET, { expiresIn: '24h' });
}

// Main refactored function
async function loginUser(email, password) {
  try {
    // Validate inputs
    const validEmail = validateEmail(email);
    const validPassword = validatePassword(password);
    
    // Authenticate user
    const user = await findUserByEmail(validEmail);
    await verifyPassword(validPassword, user.passwordHash);
    
    // Generate token
    const token = generateAuthToken(user.id);
    
    return {
      success: true,
      token,
      user: {
        id: user.id,
        email: user.email,
        name: user.name
      }
    };
  } catch (error) {
    return {
      success: false,
      error: error.message
    };
  }
}

/**
 * Key Improvements:
 * 
 * 1. **Extracted Functions**: Each validation is now a separate, testable function
 * 2. **Error Handling**: Custom error classes for different error types
 * 3. **Single Responsibility**: Each function has one clear purpose
 * 4. **Async/Await**: Proper handling of async database operations
 * 5. **Security**: Token expiration added, sensitive user data filtered
 * 6. **Maintainability**: Easy to add new validations or modify existing ones
 * 7. **Testability**: Each function can be unit tested independently
 * 
 * Potential Risks:
 * - Error handling changed from inline returns to try/catch
 * - Made functions async (ensure callers handle promises)
 * - Token now includes expiration (verify this doesn't break existing clients)
 * 
 * Recommended Tests:
 * - Test each validation function independently
 * - Test authentication success flow
 * - Test each error scenario
 * - Test token generation and expiration
 */
```

## Tips and Best Practices
- Always have tests before refactoring
- Refactor in small, incremental steps
- Run tests after each change
- Use version control to track changes
- Focus on one improvement at a time
- Don't mix refactoring with new features
- Review refactored code with team
- Document significant changes

## Related Skills
- Code Review Assistant
- Test Generator
- Documentation Generator

## Version History
- v1.0 - Initial version (2026-02-04)

## Author
ManzarIMalik

## Tags
`refactoring` `code-quality` `clean-code` `design-patterns`
