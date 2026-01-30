# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository manages **Agent Skills**—modular packages that extend Claude's capabilities with specialized knowledge, workflows, and tools. Skills are "onboarding guides" for specific domains that transform Claude from a general-purpose agent into a specialized one.

## Commands

### Initialize a new skill
```bash
python plugins/skill-creator/scripts/init_skill.py <skill-name> --path <output-directory>
```
Creates a skill directory with template SKILL.md and example resource directories (scripts/, references/, assets/).

### Validate a skill
```bash
python plugins/skill-creator/scripts/quick_validate.py <skill-directory>
```
Checks YAML frontmatter format, naming conventions, and required fields.

### Package a skill for distribution
```bash
python plugins/skill-creator/scripts/package_skill.py <skill-directory> [output-directory]
```
Validates and packages a skill into a `.skill` file (ZIP archive with .skill extension).

## Architecture

### Skill Structure
```
skill-name/
├── SKILL.md           # Required: YAML frontmatter (name, description) + markdown instructions
├── scripts/           # Optional: Executable Python/Bash for deterministic tasks
├── references/        # Optional: Documentation loaded into context as needed
└── assets/            # Optional: Files used in output (templates, images, fonts)
```

### Progressive Disclosure Design
Skills use three-level loading to manage context efficiently:
1. **Metadata** (name + description) - Always in context (~100 words)
2. **SKILL.md body** - Loaded when skill triggers (<5k words)
3. **Bundled resources** - Loaded/executed as needed

### Key Files
- `plugins/skill-creator/` - Core skill for creating other skills
- `plugins/spring-boot/` - Spring Boot development patterns and best practices
- `.claude-plugin/marketplace.json` - Plugin registry metadata

### Naming Conventions
- Skill names must be hyphen-case: lowercase letters, digits, and hyphens only
- Maximum 64 characters for name, 1024 for description
- Description cannot contain angle brackets
