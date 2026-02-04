# Security Vulnerability Scanner

## Skill Name
Security Vulnerability Scanner

## Description
An agent skill specialized in identifying security vulnerabilities in code, including common issues like SQL injection, XSS, authentication flaws, and insecure dependencies. Provides actionable remediation steps.

## Agent Type
code-review

## Use Cases
- Security audits before deployment
- Identifying OWASP Top 10 vulnerabilities
- Reviewing authentication/authorization code
- Checking for insecure dependencies
- Validating input sanitization
- Finding hardcoded secrets

## Prerequisites
- Access to the codebase
- Understanding of the application's security context
- Knowledge of dependencies and frameworks used

## Prompt Template

```
Perform a security vulnerability scan on [component/file/codebase]:

**Scan Focus:**
1. **OWASP Top 10**:
   - Injection flaws (SQL, NoSQL, Command)
   - Broken authentication
   - Sensitive data exposure
   - XML external entities (XXE)
   - Broken access control
   - Security misconfiguration
   - Cross-Site Scripting (XSS)
   - Insecure deserialization
   - Using components with known vulnerabilities
   - Insufficient logging & monitoring

2. **Additional Checks**:
   - Hardcoded secrets/credentials
   - Insecure cryptography
   - Path traversal vulnerabilities
   - CSRF vulnerabilities
   - Insecure direct object references
   - Race conditions
   - Memory leaks

**For Each Vulnerability Found:**
- **Severity**: Critical/High/Medium/Low
- **Location**: File and line number
- **Description**: What the vulnerability is
- **Impact**: Potential security risk
- **Remediation**: Specific fix with code example
- **References**: CWE/CVE numbers if applicable

**Output Priority:**
1. Critical issues first
2. Group by vulnerability type
3. Provide exploitability assessment
```

## Example Usage

### Input
```
Scan this API endpoint for security vulnerabilities:

```javascript
app.post('/api/user/update', (req, res) => {
  const userId = req.body.userId;
  const email = req.body.email;
  const role = req.body.role;
  
  const query = `UPDATE users SET email='${email}', role='${role}' WHERE id=${userId}`;
  
  db.query(query, (err, result) => {
    if (err) {
      res.status(500).send(err.message);
    }
    res.json({ success: true });
  });
});
```
```

### Expected Output
```
**Security Vulnerability Report**

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🚨 CRITICAL VULNERABILITIES

1. **SQL Injection** (CWE-89)
   **Severity:** Critical
   **Location:** Line 6 - Query construction
   **CVSS Score:** 9.8
   
   **Vulnerability:**
   User input (email, role, userId) is directly concatenated into SQL query without sanitization.
   
   **Impact:**
   - Attackers can execute arbitrary SQL commands
   - Database can be read, modified, or deleted
   - Potential data breach affecting all users
   - Possible server compromise
   
   **Exploit Example:**
   ```
   POST /api/user/update
   {
     "userId": "1; DROP TABLE users; --",
     "email": "test@example.com",
     "role": "admin"
   }
   ```
   
   **Remediation:**
   Use parameterized queries (prepared statements):
   
   ```javascript
   app.post('/api/user/update', authenticateUser, (req, res) => {
     const { userId, email, role } = req.body;
     
     // Use parameterized query
     const query = 'UPDATE users SET email = ?, role = ? WHERE id = ?';
     const params = [email, role, userId];
     
     db.query(query, params, (err, result) => {
       if (err) {
         logger.error('Database error', { error: err });
         res.status(500).json({ error: 'Internal server error' });
         return;
       }
       res.json({ success: true });
     });
   });
   ```

2. **Broken Access Control** (CWE-284)
   **Severity:** Critical
   **Location:** Line 1 - Missing authentication
   
   **Vulnerability:**
   No authentication or authorization checks before allowing user updates.
   
   **Impact:**
   - Anyone can update any user's data
   - Privilege escalation (changing role to admin)
   - Unauthorized access to user accounts
   
   **Remediation:**
   Add authentication and authorization:
   
   ```javascript
   // Middleware to verify user is authenticated
   function authenticateUser(req, res, next) {
     const token = req.headers.authorization?.split(' ')[1];
     if (!token) {
       return res.status(401).json({ error: 'Authentication required' });
     }
     
     try {
       const decoded = jwt.verify(token, process.env.JWT_SECRET);
       req.user = decoded;
       next();
     } catch (err) {
       return res.status(401).json({ error: 'Invalid token' });
     }
   }
   
   // Middleware to verify user can only update their own data
   function authorizeUserUpdate(req, res, next) {
     const requestedUserId = parseInt(req.body.userId);
     const currentUserId = req.user.userId;
     
     if (requestedUserId !== currentUserId && req.user.role !== 'admin') {
       return res.status(403).json({ error: 'Forbidden' });
     }
     next();
   }
   
   app.post('/api/user/update', 
     authenticateUser, 
     authorizeUserUpdate, 
     updateUserHandler
   );
   ```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  HIGH SEVERITY VULNERABILITIES

3. **Privilege Escalation** (CWE-269)
   **Severity:** High
   **Location:** Line 4 - Role modification
   
   **Vulnerability:**
   Users can modify their own role, including escalating to admin.
   
   **Impact:**
   - Users can grant themselves admin privileges
   - Bypass authorization controls
   
   **Remediation:**
   Restrict role changes to admin users only:
   
   ```javascript
   function updateUserHandler(req, res) {
     const { userId, email, role } = req.body;
     
     // Only admins can change roles
     if (role && req.user.role !== 'admin') {
       return res.status(403).json({ 
         error: 'Only administrators can change user roles' 
       });
     }
     
     // Build query based on what user can update
     let query, params;
     if (role && req.user.role === 'admin') {
       query = 'UPDATE users SET email = ?, role = ? WHERE id = ?';
       params = [email, role, userId];
     } else {
       query = 'UPDATE users SET email = ? WHERE id = ?';
       params = [email, userId];
     }
     
     db.query(query, params, (err, result) => {
       // ... handle response
     });
   }
   ```

4. **Information Disclosure** (CWE-209)
   **Severity:** High
   **Location:** Line 9 - Error handling
   
   **Vulnerability:**
   Database error messages exposed to client.
   
   **Impact:**
   - Reveals database structure
   - Aids attackers in crafting attacks
   - May expose sensitive paths or data
   
   **Remediation:**
   Log detailed errors server-side, return generic message to client.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 MEDIUM SEVERITY ISSUES

5. **Missing Input Validation** (CWE-20)
   **Severity:** Medium
   **Location:** Lines 2-4
   
   **Remediation:**
   ```javascript
   const { body } = req;
   
   // Validate email
   if (!body.email || !isValidEmail(body.email)) {
     return res.status(400).json({ error: 'Invalid email format' });
   }
   
   // Validate role
   const validRoles = ['user', 'admin', 'moderator'];
   if (body.role && !validRoles.includes(body.role)) {
     return res.status(400).json({ error: 'Invalid role' });
   }
   ```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Summary:**
- Critical: 2
- High: 2
- Medium: 1
- Total: 5 vulnerabilities

**Recommended Actions:**
1. ✅ IMMEDIATE: Fix SQL injection (use parameterized queries)
2. ✅ IMMEDIATE: Add authentication and authorization
3. ✅ HIGH PRIORITY: Restrict role changes
4. ✅ HIGH PRIORITY: Improve error handling
5. ⚠️  MEDIUM: Add input validation

**Additional Security Measures:**
- Implement rate limiting to prevent brute force
- Add CSRF protection
- Enable security headers (helmet.js)
- Implement audit logging for all user updates
- Add input sanitization
- Use HTTPS only
- Implement account lockout after failed attempts
```

## Tips and Best Practices
- Run security scans before every deployment
- Focus on critical vulnerabilities first
- Test fixes in staging environment
- Use security linting tools (eslint-plugin-security)
- Keep dependencies updated
- Follow OWASP guidelines
- Implement defense in depth
- Regular security training for team

## Related Skills
- Code Review Assistant
- Dependency Vulnerability Checker
- Authentication Pattern Validator

## Version History
- v1.0 - Initial version (2026-02-04)

## Author
ManzarIMalik

## Tags
`security` `vulnerability-scanning` `owasp` `penetration-testing` `code-security`
