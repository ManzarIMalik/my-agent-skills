# Quick Reference Guide

Quick access to agent skills by common scenarios.

## 🎯 "I need to..."

### Understand New Code
→ **[Codebase Explorer](skills/code-analysis/codebase-explorer.md)**
- Map out project structure
- Identify key components
- Understand architecture

### Review My Changes
→ **[Code Review Assistant](skills/code-analysis/code-review-assistant.md)**
- Get automated code review
- Check for security issues
- Find bugs before commit

### Write Documentation
→ **[Documentation Generator](skills/documentation/documentation-generator.md)**
- Generate API docs
- Create README files
- Write inline comments

### Add Tests
→ **[Test Generator](skills/testing/test-generator.md)**
- Create unit tests
- Generate integration tests
- Cover edge cases

### Fix a Bug
→ **[Bug Hunter](skills/debugging/bug-hunter.md)**
- Analyze error messages
- Find root causes
- Get fix recommendations

### Deploy My App
→ **[Deployment Automation](skills/deployment/deployment-automation.md)**
- Set up CI/CD pipelines
- Create Docker configs
- Automate releases

## 🔧 By Technology

### JavaScript/TypeScript
- Code Review Assistant
- Test Generator (Jest)
- Documentation Generator

### Python
- Test Generator (pytest)
- Code Review Assistant
- Bug Hunter

### Docker/Kubernetes
- Deployment Automation

### Cloud Platforms
- Deployment Automation (AWS, Azure, GCP)

## ⚡ By Development Phase

### Planning
- Codebase Explorer

### Development
- Code Review Assistant
- Documentation Generator

### Testing
- Test Generator

### Debugging
- Bug Hunter

### Deployment
- Deployment Automation

## 🎓 Learning Path

### Beginner
1. Start with **Codebase Explorer** to understand code
2. Use **Documentation Generator** to create docs
3. Try **Test Generator** for basic testing

### Intermediate
1. **Code Review Assistant** for quality checks
2. **Bug Hunter** for debugging
3. **Deployment Automation** basics

### Advanced
1. Combine multiple skills
2. Customize prompts for your workflow
3. Create your own skills

## 💡 Pro Tips

### Skill Combinations

**Code Quality Workflow:**
1. Codebase Explorer → understand code
2. Code Review Assistant → review changes
3. Test Generator → add tests
4. Deployment Automation → deploy

**Debugging Workflow:**
1. Bug Hunter → find issue
2. Test Generator → create regression test
3. Code Review Assistant → verify fix

**New Project Onboarding:**
1. Codebase Explorer → learn structure
2. Documentation Generator → create/update docs
3. Test Generator → add missing tests

### Time Savers

- **Daily Code Review**: Use Code Review Assistant before every commit
- **TDD**: Use Test Generator to write tests first
- **Documentation**: Generate docs as you code
- **Debugging**: Start with Bug Hunter before manual debugging

## 📱 Quick Actions

Copy these commands to your clipboard manager:

```markdown
# Quick code review
Use the Code Review Assistant skill to review my changes

# Explore codebase
Use the Codebase Explorer skill to explain this repository

# Generate tests
Use the Test Generator skill to create tests for [file/function]

# Debug issue
Use the Bug Hunter skill to debug this error: [error message]

# Create docs
Use the Documentation Generator skill to document [component]

# Setup deployment
Use the Deployment Automation skill to create CI/CD for [platform]
```

## 🔍 Finding the Right Skill

### Ask Yourself:

**What's my goal?**
- Understand → Codebase Explorer
- Improve → Code Review Assistant
- Document → Documentation Generator
- Test → Test Generator
- Fix → Bug Hunter
- Deploy → Deployment Automation

**What's my context?**
- New to codebase → Codebase Explorer
- Before commit → Code Review Assistant
- After coding → Test Generator
- Production issue → Bug Hunter

**What's my priority?**
- Speed → Deployment Automation
- Quality → Code Review Assistant
- Coverage → Test Generator
- Clarity → Documentation Generator

---

**Not finding what you need?**
- Check [all skills](README.md#-available-skills)
- [Create your own](CONTRIBUTING.md)
- [Request a new skill](https://github.com/ManzarIMalik/my-agent-skills/issues)
