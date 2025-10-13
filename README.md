# claude_files

Configuration files and prompts for working with Claude Code.

## Quick Start

1. **Fork this repository**
   ```bash
   # Click the "Fork" button on GitHub, then clone your fork
   git clone https://github.com/YOUR_USERNAME/claude_files.git
   cd claude_files
   ```

2. **Customize CLAUDE.md for your workflow**
   - Edit `CLAUDE.md` to match your coding style and preferences
   - Update the partner name references (currently "Taariq") to your own name
   - Adjust rules to fit your development practices

3. **Copy CLAUDE.md to your projects**
   ```bash
   # Copy to any project where you want Claude Code to follow these rules
   cp CLAUDE.md /path/to/your/project/
   ```

4. **Start using Claude Code**
   - Claude Code automatically reads CLAUDE.md from your project root
   - The rules will apply to all interactions in that project

## What's Included

- **CLAUDE.md** - Core instructions for Claude Code covering:
  - Code style and conventions
  - Testing requirements (TDD)
  - Git workflow and commit practices
  - Security rules for secrets and sensitive data
  - Debugging methodology

- **brainstorm.md** - Prompt template for iterative design sessions
- **write_plan_prompt.md** - Template for generating implementation plans

## Customization Tips

- Keep rules concise and actionable
- Use "MUST" for hard requirements, "should" for preferences
- Add project-specific patterns and conventions
- Include examples of good vs. bad practices
- Update rules based on what works for your workflow
