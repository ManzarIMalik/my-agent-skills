# Contributing to My Agent Skills

Thank you for your interest in contributing to this collection of agent skills! This guide will help you contribute effectively.

## 🎯 Types of Contributions

We welcome:

1. **New Agent Skills** - Share skills you've developed and tested
2. **Skill Improvements** - Enhance existing skills with better prompts or examples
3. **Documentation** - Improve README, guides, or skill descriptions
4. **Bug Reports** - Report issues with existing skills
5. **Use Cases** - Share successful applications of skills

## 📋 Contribution Process

### 1. Before You Start

- Check existing skills to avoid duplicates
- Read through the [skill template](templates/skill-template.md)
- Ensure your skill is tested and works as expected
- Consider if your skill fits an existing category or needs a new one

### 2. Adding a New Skill

**Step 1: Fork and Clone**
```bash
git clone https://github.com/YOUR_USERNAME/my-agent-skills.git
cd my-agent-skills
git checkout -b add-skill-name
```

**Step 2: Create Your Skill**
```bash
# Copy the template
cp templates/skill-template.md skills/CATEGORY/your-skill-name.md

# Edit the file with your skill details
```

**Step 3: Fill Out the Template**

Required sections:
- ✅ Skill Name
- ✅ Description (what it does and when to use it)
- ✅ Agent Type
- ✅ Use Cases (at least 3)
- ✅ Prompt Template
- ✅ Example Usage with Input/Output
- ✅ Tips and Best Practices (at least 3)
- ✅ Tags (at least 2)

Optional but recommended:
- Prerequisites
- Related Skills
- Version History
- Author

**Step 4: Test Your Skill**

Before submitting:
1. Test the prompt with an AI assistant (Copilot, Claude, etc.)
2. Verify the output matches your example
3. Try 2-3 variations to ensure robustness
4. Check for any unclear instructions

**Step 5: Update Documentation**

Add your skill to README.md:
```markdown
### Category Name
- **[Your Skill Name](skills/category/your-skill.md)** - Brief description
```

**Step 6: Submit Pull Request**

```bash
git add .
git commit -m "Add [Skill Name] agent skill"
git push origin add-skill-name
```

Create a pull request with:
- **Title**: `Add [Skill Name] agent skill`
- **Description**: 
  - What the skill does
  - Which category it belongs to
  - How you tested it
  - Any special considerations

### 3. Improving Existing Skills

**What to Improve:**
- Clearer prompt instructions
- Better examples
- Additional use cases
- Fixed errors or outdated information
- Enhanced best practices

**Process:**
1. Fork and create a branch: `improve-skill-name`
2. Make your changes
3. Test the improved version
4. Submit PR with clear explanation of improvements

### 4. Reporting Issues

If you find a problem with a skill:

1. Check if it's already reported in [Issues](https://github.com/ManzarIMalik/my-agent-skills/issues)
2. If not, create a new issue with:
   - **Title**: `[Skill Name] - Brief issue description`
   - **Description**:
     - What you expected
     - What actually happened
     - Steps to reproduce
     - Your AI assistant version/platform
     - Any error messages

## 📏 Quality Standards

### Prompt Quality

**Good Prompts:**
```markdown
✅ Clear and specific
✅ Include context and requirements
✅ Specify expected output format
✅ Cover edge cases
✅ Provide examples
✅ Use structured formatting
```

**Avoid:**
```markdown
❌ Vague or ambiguous instructions
❌ Missing context
❌ No examples
❌ Overly complex or nested instructions
❌ Untested assumptions
```

### Example Quality

**Good Examples:**
- Realistic scenarios
- Complete input and output
- Show both simple and complex cases
- Include code blocks with proper syntax highlighting
- Demonstrate the value of the skill

### Documentation Quality

- Use clear, concise language
- Include code examples where relevant
- Maintain consistent formatting
- Add links to related resources
- Keep categories organized

## 🏷️ Categorization

### Existing Categories

- `code-analysis/` - Code review, refactoring, architecture
- `documentation/` - Docs generation, comments, guides
- `testing/` - Test creation, coverage, quality assurance
- `debugging/` - Bug hunting, error analysis, troubleshooting
- `deployment/` - CI/CD, containerization, infrastructure

### Creating New Categories

If your skill doesn't fit existing categories:

1. Propose the new category in your PR description
2. Explain why it's needed
3. Suggest at least 2-3 skills that would fit
4. Create the directory: `skills/new-category/`

## 🎨 Formatting Guidelines

### File Naming

- Use lowercase with hyphens: `my-skill-name.md`
- Be descriptive but concise
- Match the skill name

### Markdown Style

```markdown
# Main headings (H1) for skill name only
## Section headings (H2)
### Subsections (H3)

**Bold** for emphasis
`code` for inline code
```code blocks``` for multi-line code

- Bullet points for lists
1. Numbered lists for sequential steps
```

### Code Blocks

Always specify the language:

````markdown
```javascript
// JavaScript code
```

```python
# Python code
```

```bash
# Bash commands
```
````

## 🔍 Review Process

### What We Look For

1. **Completeness** - All required sections filled
2. **Clarity** - Easy to understand and follow
3. **Tested** - Skill works as described
4. **Value** - Provides genuine utility
5. **Quality** - Follows formatting and style guidelines

### Timeline

- Initial review: Within 3-5 days
- Feedback incorporated: 1-2 days
- Merge: After approval from maintainer

## 🤝 Code of Conduct

### Be Respectful

- Welcome newcomers
- Provide constructive feedback
- Assume good intentions
- Focus on ideas, not people

### Be Collaborative

- Share knowledge
- Help others improve their skills
- Credit original authors
- Build on existing work

### Be Professional

- Keep discussions on-topic
- Use appropriate language
- Respect different perspectives
- Follow GitHub's community guidelines

## 💬 Getting Help

Need assistance?

- **Questions about contributing**: Open a Discussion
- **Technical issues**: Open an Issue
- **General questions**: Comment on relevant PRs or Issues

## 🎓 Learning Resources

Helpful resources for creating great agent skills:

- [Prompt Engineering Guide](https://www.promptingguide.ai/)
- [GitHub Markdown Guide](https://guides.github.com/features/mastering-markdown/)
- [Writing Great Prompts](https://github.blog/2023-06-20-how-to-write-better-prompts-for-github-copilot/)

## 📜 License

By contributing, you agree that your contributions will be licensed under the MIT License.

## 🙏 Recognition

Contributors will be:
- Listed in the skill's Author section
- Acknowledged in release notes
- Featured in the README (for significant contributions)

---

Thank you for helping make AI-assisted development better for everyone! 🚀
