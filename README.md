# Agent Skills

A collection of modular skills that extend Claude Code's capabilities with specialized knowledge, workflows, and tools.

## What are Skills?

Skills are "onboarding guides" for specific domains that transform Claude from a general-purpose agent into a specialized one. Each skill provides:

- **Specialized workflows** - Multi-step procedures for specific domains
- **Domain expertise** - Best practices, patterns, and conventions
- **Bundled resources** - Reference documentation loaded as needed

## Available Skills

| Skill                               | Description                                                                                                                                                                                            |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [spring-boot](plugins/spring-boot/) | Spring Boot development patterns, best practices, and Spring Boot 4 features. Covers REST APIs, dependency injection, testing strategies, error handling with ProblemDetail, and modern Java patterns. |

## Installation

### Claude Code (Plugin Marketplace)

```bash
/plugin marketplace add uuhnaut69/agent-skills
/plugin enable uuhnaut69/agent-skills
/plugin install spring-boot@agent-skills
```

### Claude Code (Manual Installation)

```bash
# Clone or download this repository
git clone https://github.com/uuhnaut69/agent-skills.git

# Copy skills to Claude Code skills directory
cp -r agent-skills/plugins/* ~/.claude/skills/
```

## Usage

Once installed, Claude Code will automatically detect and use the appropriate skill based on your queries:

- **Spring Boot queries**: Ask about Spring Boot patterns, REST API design, testing strategies, or Spring Boot 4 features
  - "How should I structure my Spring Boot REST controller?"
  - "What's the best way to handle exceptions with ProblemDetail?"
  - "Show me how to use RestTestClient in Spring Boot 4"

Skills use progressive disclosure - only loading detailed reference material when needed to keep context efficient.

## Skill Structure

Each skill follows this structure:

```
skill-name/
├── SKILL.md           # Core instructions and guidance
└── references/        # Detailed documentation loaded as needed
```

## License

[MIT](LICENSE)
