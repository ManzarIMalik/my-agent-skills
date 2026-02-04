# Bug Hunter

## Skill Name
Bug Hunter

## Description
An agent skill specialized in identifying, analyzing, and suggesting fixes for bugs in code. It performs systematic debugging by analyzing error messages, stack traces, and code logic to pinpoint root causes.

## Agent Type
general-purpose

## Use Cases
- Debugging runtime errors
- Analyzing stack traces
- Finding logic errors
- Identifying race conditions
- Resolving integration issues
- Tracking down memory leaks

## Prerequisites
- Access to error logs and stack traces
- Access to relevant source code
- Understanding of the expected behavior

## Prompt Template

```
Debug the following issue:

**Problem Description:**
[Describe the bug, error message, or unexpected behavior]

**Steps to Reproduce:**
1. [Step 1]
2. [Step 2]
3. [Step 3]

**Expected Behavior:**
[What should happen]

**Actual Behavior:**
[What actually happens]

**Error Messages/Stack Trace:**
```
[Paste error messages or stack traces]
```

**Investigation Requirements:**
1. Analyze the error message and stack trace
2. Identify the root cause (not just symptoms)
3. Trace the execution flow leading to the error
4. Check for common bug patterns (null refs, race conditions, etc.)
5. Review related code that might contribute to the issue
6. Suggest specific fixes with code examples
7. Recommend preventive measures

**Debugging Approach:**
- Start with the error message and work backwards
- Check assumptions and preconditions
- Look for edge cases
- Verify data types and formats
- Check for timing issues
- Review recent changes
```

## Example Usage

### Input
```
Debug this error:

Error: Cannot read property 'name' of undefined
    at UserProfile.render (UserProfile.js:45)
    at ReactDOM.render (react-dom.js:1234)

The error occurs when navigating to /profile after login.
Expected: User profile displays with name
Actual: Application crashes with the error above
```

### Expected Output
```
**Bug Analysis: Undefined User Object**

**Root Cause:**
The `user` object is undefined when `UserProfile` component renders. This is a race condition where the component renders before user data is fetched.

**Execution Flow:**
1. User logs in → redirected to /profile
2. UserProfile component mounts
3. Component attempts to access user.name (line 45)
4. User data fetch is still in progress (async)
5. Error: user is undefined

**Problem Location:**
UserProfile.js, line 45:
```javascript
// Current code (problematic)
return <h1>Welcome, {user.name}</h1>;
```

**Root Cause Details:**
- Component doesn't handle loading state
- No check for undefined user before accessing properties
- Async data fetch completes after initial render

**Recommended Fix:**

**Option 1: Add null check (quick fix)**
```javascript
// UserProfile.js, line 45
return <h1>Welcome, {user?.name || 'Guest'}</h1>;
```

**Option 2: Add loading state (better)**
```javascript
function UserProfile() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUserData()
      .then(setUser)
      .finally(() => setLoading(false));
  }, []);

  if (loading) {
    return <div>Loading...</div>;
  }

  if (!user) {
    return <div>User not found</div>;
  }

  return <h1>Welcome, {user.name}</h1>;
}
```

**Option 3: Redirect if no user (most robust)**
```javascript
function UserProfile() {
  const [user, setUser] = useState(null);
  const navigate = useNavigate();

  useEffect(() => {
    const storedUser = localStorage.getItem('user');
    if (!storedUser) {
      navigate('/login');
      return;
    }
    setUser(JSON.parse(storedUser));
  }, [navigate]);

  if (!user) {
    return null; // or loading spinner
  }

  return <h1>Welcome, {user.name}</h1>;
}
```

**Preventive Measures:**
1. Always check for null/undefined before accessing properties
2. Implement loading states for async operations
3. Add PropTypes or TypeScript for type safety
4. Use optional chaining (?.) for nested properties
5. Add error boundaries to catch rendering errors
6. Write tests for edge cases (missing data, slow network)

**Testing Recommendation:**
```javascript
it('should show loading state while fetching user', () => {
  render(<UserProfile />);
  expect(screen.getByText('Loading...')).toBeInTheDocument();
});

it('should handle missing user data gracefully', () => {
  mockFetchUserData.mockResolvedValue(null);
  render(<UserProfile />);
  expect(screen.getByText('User not found')).toBeInTheDocument();
});
```

**Priority:** High - Application crash
**Estimated Fix Time:** 15 minutes
```

## Tips and Best Practices
- Start by reproducing the bug in a minimal environment
- Read error messages carefully - they often point to the exact issue
- Use debugger breakpoints to inspect state at critical points
- Check recent commits that might have introduced the bug
- Look for similar issues in issue trackers or Stack Overflow
- Test the fix thoroughly before committing
- Add regression tests to prevent the bug from reoccurring

## Related Skills
- Code Review Assistant
- Test Generator
- Error Log Analyzer

## Version History
- v1.0 - Initial version (2026-02-04)

## Author
ManzarIMalik

## Tags
`debugging` `bug-fixing` `error-analysis` `troubleshooting`
