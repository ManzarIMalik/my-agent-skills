# Codebase Explorer

## Skill Name
Codebase Explorer

## Description
An agent skill designed to quickly explore and understand unfamiliar codebases. It analyzes project structure, identifies key components, maps dependencies, and provides a comprehensive overview of how the code is organized.

## Agent Type
explore

## Use Cases
- Onboarding to new projects
- Understanding legacy code
- Finding specific functionality
- Mapping code architecture
- Identifying entry points and main flows

## Prerequisites
- Access to the repository
- Basic understanding of the technology stack

## Prompt Template

```
Explore this codebase and provide a comprehensive overview:

1. **Project Structure**: Identify main directories and their purposes
2. **Technology Stack**: List frameworks, libraries, and tools used
3. **Entry Points**: Find main application entry points (main.js, app.py, etc.)
4. **Key Components**: Identify core modules and their responsibilities
5. **Dependencies**: Map external dependencies and internal module relationships
6. **Configuration**: Locate configuration files and environment setup
7. **Build/Test**: Identify build scripts, test files, and CI/CD setup
8. **Documentation**: Find README, docs, and inline documentation

Focus on understanding the overall architecture and data flow.

Provide:
- High-level architecture diagram (textual)
- Component relationship map
- Quick start guide for developers
```

## Example Usage

### Input
```
Explore the repository at /home/runner/work/project and explain its structure and main components.
```

### Expected Output
```
**Codebase Overview**

Project Structure:
├── src/
│   ├── components/    # React UI components
│   ├── services/      # API and business logic
│   ├── utils/         # Helper functions
│   └── App.js         # Main application entry point
├── tests/             # Test suites
├── public/            # Static assets
└── config/            # Configuration files

Technology Stack:
- React 18.2 (UI framework)
- Express 4.x (Backend API)
- PostgreSQL (Database)
- Jest (Testing)

Architecture:
- Frontend: Component-based React SPA
- Backend: RESTful API with Express
- Database: PostgreSQL with Sequelize ORM

Key Components:
1. UserService (src/services/user.js): User authentication and management
2. AuthMiddleware (src/middleware/auth.js): JWT token validation
3. DataProvider (src/components/DataProvider.js): Global state management

Entry Points:
- Frontend: src/App.js
- Backend: src/server.js

Next Steps:
1. Install dependencies: npm install
2. Configure .env file (see .env.example)
3. Run migrations: npm run migrate
4. Start dev server: npm run dev
```

## Tips and Best Practices
- Start with high-level structure before diving into specific files
- Use glob patterns to find file types: `**/*.test.js`
- Search for common patterns like "config", "main", "app"
- Look for package.json, requirements.txt, or similar dependency files first
- Check for CI/CD configs to understand build/deploy process

## Related Skills
- Dependency Analyzer
- Architecture Documenter
- Code Flow Tracer

## Version History
- v1.0 - Initial version (2026-02-04)

## Author
ManzarIMalik

## Tags
`exploration` `codebase-analysis` `onboarding` `architecture`
