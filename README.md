# 🤖 My Agent Skills

A curated collection of agent skills and prompts for AI-powered development assistants. These skills help automate common development tasks, improve code quality, and accelerate software development workflows.

## 📚 What Are Agent Skills?

Agent skills are specialized prompts and workflows designed for AI coding assistants (like GitHub Copilot, Cursor, etc.). Each skill provides:

- **Clear objectives** - What the agent should accomplish
- **Structured prompts** - Ready-to-use templates
- **Best practices** - Proven approaches and guidelines
- **Examples** - Real-world usage scenarios

## 🗂️ Repository Structure

```
my-agent-skills/
├── skills/
│   ├── code-analysis/       # Code review, exploration, refactoring
│   ├── documentation/        # API docs, README generation
│   ├── testing/             # Test generation, coverage analysis
│   ├── debugging/           # Bug hunting, error analysis
│   └── deployment/          # CI/CD, containerization, automation
├── templates/               # Templates for creating new skills
└── README.md               # This file
```

## 🚀 Available Skills

### Code Analysis
- **[Code Review Assistant](skills/code-analysis/code-review-assistant.md)** - Comprehensive code reviews with security focus
- **[Codebase Explorer](skills/code-analysis/codebase-explorer.md)** - Quick codebase understanding and architecture mapping
- **[Refactoring Assistant](skills/code-analysis/refactoring-assistant.md)** - Improve code quality and maintainability
- **[Security Vulnerability Scanner](skills/code-analysis/security-scanner.md)** - Identify and fix security vulnerabilities

### Documentation
- **[Documentation Generator](skills/documentation/documentation-generator.md)** - API docs, README files, inline comments

### Testing
- **[Test Generator](skills/testing/test-generator.md)** - Unit, integration, and edge case test creation

### Debugging
- **[Bug Hunter](skills/debugging/bug-hunter.md)** - Systematic debugging and root cause analysis

### Deployment
- **[Deployment Automation](skills/deployment/deployment-automation.md)** - CI/CD pipelines, Docker, Kubernetes configs

## 💡 How to Use

### 1. Browse the Skills
Navigate through the `skills/` directory to find a skill that matches your need.

### 2. Copy the Prompt Template
Open the skill file and copy the prompt template section.

### 3. Customize for Your Context
Replace placeholders with your specific requirements:
```
[component/function/class] → UserService
[platform] → AWS
[framework] → React
```

### 4. Use with Your AI Assistant
Paste the customized prompt to your AI coding assistant (GitHub Copilot Chat, Cursor, Claude, etc.).

### Example Workflow

```markdown
1. Need to review code? 
   → Use "Code Review Assistant" skill
   
2. New to a codebase? 
   → Use "Codebase Explorer" skill
   
3. Missing tests? 
   → Use "Test Generator" skill
   
4. Bug in production? 
   → Use "Bug Hunter" skill
   
5. Need to deploy? 
   → Use "Deployment Automation" skill
```

## ✨ Creating Your Own Skills

### Use the Template

1. Copy `templates/skill-template.md`
2. Fill in each section:
   - Name and description
   - Agent type
   - Use cases
   - Prompt template
   - Examples
   - Best practices

3. Save to appropriate category folder
4. Submit a pull request!

### Skill Quality Guidelines

**Good Skills Have:**
- ✅ Clear, specific objectives
- ✅ Actionable prompt templates
- ✅ Real-world examples
- ✅ Best practices and tips
- ✅ Proper categorization

**Avoid:**
- ❌ Vague or ambiguous instructions
- ❌ Overly complex prompts
- ❌ Missing examples
- ❌ Untested workflows

## 🤝 Contributing

Contributions are welcome! Here's how:

### Adding a New Skill

1. Fork this repository
2. Create a new branch: `git checkout -b add-my-skill`
3. Use the template in `templates/skill-template.md`
4. Add your skill to the appropriate category
5. Update this README's "Available Skills" section
6. Submit a pull request

### Improving Existing Skills

1. Fork this repository
2. Make your improvements
3. Test the skill with an AI assistant
4. Submit a pull request with details about improvements

### Guidelines

- Keep skills focused on a single task
- Provide working examples
- Test prompts before submitting
- Use clear, concise language
- Follow the existing format

## 📖 Best Practices

### When Using Agent Skills

1. **Be Specific** - Customize prompts with your exact requirements
2. **Iterate** - Refine the output by asking follow-up questions
3. **Validate** - Always review and test generated code
4. **Combine** - Use multiple skills together for complex tasks
5. **Share** - Contribute improvements back to the community

### Skill Organization Tips

- **Bookmark frequently used skills** for quick access
- **Create custom variations** for your specific tech stack
- **Document which skills work best** for your use cases
- **Share successful combinations** with your team

## 🏷️ Tags

Use tags to quickly find relevant skills:

- `code-review` - Code quality and security checks
- `testing` - Test generation and coverage
- `documentation` - Docs and comments
- `debugging` - Bug finding and fixing
- `deployment` - CI/CD and infrastructure
- `architecture` - Design and planning
- `refactoring` - Code improvements
- `security` - Security analysis

## 📝 License

This repository is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🌟 Why This Matters

Agent skills help developers:
- **Save time** on repetitive tasks
- **Maintain consistency** across projects
- **Learn best practices** through examples
- **Improve code quality** with systematic approaches
- **Onboard faster** to new technologies

## 🔗 Related Resources

- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
- [Prompt Engineering Guide](https://www.promptingguide.ai/)
- [AI-Assisted Development Best Practices](https://github.blog/2023-06-20-how-to-write-better-prompts-for-github-copilot/)

## 📬 Feedback

Have suggestions or questions? 
- Open an issue
- Start a discussion
- Submit a pull request

---

**Happy coding with AI! 🚀**

Made with ❤️ by developers, for developers.
