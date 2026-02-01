# Agent Skills

A collection of modular skills that extend Claude Code's capabilities with specialized knowledge, workflows, and tools.

## What are Skills?

Skills are "onboarding guides" for specific domains that transform Claude from a general-purpose agent into a specialized one. Each skill provides:

- **Specialized workflows** - Multi-step procedures for specific domains
- **Domain expertise** - Best practices, patterns, and conventions
- **Bundled resources** - Reference documentation loaded as needed

## Available Skills

| Skill                                      | Description                                                                                                                                                                           |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [java-spring](skills/java-spring/)         | Best practices for Java/Spring projects. Covers DDD principles, explicit code (no Lombok), Spring Boot 4 features, REST APIs, testing with Testcontainers, and observability.         |
| [spring-modulith](skills/spring-modulith/) | Expert guidance on building modular monoliths with Spring Modulith 2.0.2+ (targeting Spring Boot 3.4+). Covers module structure, verification, testing, and documentation generation. |

## Installation

### Claude Code (Plugin Marketplace)

```bash
/plugin marketplace add uuhnaut69/skills
/plugin install java-spring@skills
```

### Claude Code (Manual Installation)

```bash
# Clone or download this repository
git clone git@github.com:uuhnaut69/skills.git

# Copy skills to Claude Code skills directory
cp -r skills/skills/* ~/.claude/skills/
```

## Usage

Once installed, Claude Code will automatically detect and use the appropriate skill based on your queries:

- **Java/Spring queries**: Ask about Spring Boot patterns, DDD structure, REST API design, testing strategies, or Spring Boot 4 features
  - "How should I structure my Spring Boot REST controller?"
  - "What's the best way to handle exceptions with ProblemDetail?"
  - "Show me how to use RestTestClient in Spring Boot 4"

- **Spring Modulith queries**: Ask about modular monolith architecture, module boundaries, or event-driven design
  - "How do I define modules in Spring Modulith?"
  - "How can I test individual application modules?"
  - "Show me how to generate module documentation"

## Skill Structure

Each skill follows this structure:

```
skill-name/
├── SKILL.md           # Core instructions and guidance
└── references/        # Detailed documentation loaded as needed
```

## License

[MIT](LICENSE)
