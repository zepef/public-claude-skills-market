# Public Claude Code Skills Market

A curated collection of Claude Code Skills created by Pierre-Emmanuel FEGA, assisted by Cline and Claude Code, designed to empower Claude Code users with enhanced capabilities and specialized workflows.

https://github.com/user-attachments/assets/claude-code-skills-animation.mp4

## What are Claude Code Skills?

Claude Code Skills are specialized capabilities that extend Claude's functionality through focused prompts and tool integrations. Skills enable Claude to perform domain-specific tasks more effectively by providing targeted expertise, workflows, and tool access patterns.

According to the [official documentation](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview#why-use-skills), skills help Claude:

- **Specialize**: Focus on specific domains or task types with dedicated expertise
- **Maintain Context**: Preserve relevant knowledge and patterns across conversations
- **Optimize Tool Usage**: Apply best practices for tool selection and coordination
- **Improve Consistency**: Deliver reliable, predictable results for common workflows

## Available Skills

Browse the skills directory to discover capabilities for:

- Software development workflows
- Code analysis and refactoring
- Testing and quality assurance
- Documentation generation
- DevOps and deployment automation
- And more...

## Installation

### Installing a Skill

Claude Code Skills are stored in the `.claude/skills/` directory. You can install skills either globally (available in all projects) or locally (specific to one project).

#### Global Installation (Recommended)

Global skills are available across all your projects:

```bash
# Navigate to your global Claude configuration directory
cd ~/.claude/skills/

# Clone or copy the desired skill
# For example, if the skill is in this repository:
git clone https://github.com/yourusername/public-claude-skills-market.git
# Or copy individual skill directories
cp -r /path/to/skill-name ~/.claude/skills/
```

#### Local Installation (Project-Specific)

Local skills are only available within a specific project:

```bash
# Navigate to your project directory
cd /path/to/your/project

# Create skills directory if it doesn't exist
mkdir -p .claude/skills/

# Copy the desired skill
cp -r /path/to/skill-name .claude/skills/
```

### Using a Skill

Once installed, invoke a skill in Claude Code using the `/skill` command:

```
/skill skill-name
```

Or reference skills in your prompts:

```
Use the [skill-name] skill to analyze this codebase
```

## Skill Structure

Each skill in this repository follows the standard Claude Code Skills structure:

```
skill-name/
├── skill.md          # Main skill prompt and instructions
├── README.md         # Skill-specific documentation
└── examples/         # Usage examples (optional)
```

## Creating Your Own Skills

Want to create custom skills? Check out the official guides:

- [Agent Skills Overview](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview#why-use-skills)
- [Claude Code Skills Documentation](https://docs.claude.com/en/docs/claude-code/skills)

## Contributing

Contributions are welcome! If you've created a useful Claude Code Skill:

1. Fork this repository
2. Add your skill to the appropriate category
3. Include clear documentation and examples
4. Submit a pull request

## References

- [Claude Agent Skills Overview](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview#why-use-skills)
- [Claude Code Skills Documentation](https://docs.claude.com/en/docs/claude-code/skills)

## License

See individual skill directories for specific licenses.

## Author

**Pierre-Emmanuel FEGA**

Created with assistance from:
- Cline
- Claude Code

## Support

For issues, questions, or suggestions:
- Open an issue in this repository
- Refer to the official [Claude Code documentation](https://docs.claude.com/en/docs/claude-code)

---

*Empowering Claude Code users, one skill at a time.*
