# AGENTS — AISkillsForMe

This file tells AI agents how to work with this project.

## Project Overview

A curated collection of development skills/rules organized by tech stack. Each file is a reusable skill document following the SKILL.md format — designed to be loaded by AI coding assistants (Hermes Agent, Claude Code, Cursor, etc.) to follow best practices for specific technologies.

## Tech Stack Categories

| Category | Sub-topics |
|---|---|
| **spring-cloud** | Nacos, Gateway, Sentinel, Feign, Seata |
| **vue3** | Composition API, Pinia, Router, Vite |
| **postgresql** | SQL, indexing, migration, connection pool |
| **redis** | Cache, distributed lock, Redisson, pub/sub |
| **minio** | Object storage, S3 API, presigned URL |
| **onlyoffice** | Document server, online editing, callback |
| **rabbitmq** | AMQP, exchanges, DLQ, publisher confirm |
| **rocketmq** | Transaction message, ordered message, delayed |
| **mybatis-plus** | Lambda query, pagination, auto-fill |
| **tools** | Git, Docker, Maven, Gradle |
| **patterns** | Microservice, DDD, design patterns |

## How to Use

When working on a specific technology, the AI agent should:

1. Check if a skill file exists under `skills/<category>/`
2. Load the relevant skill(s) to follow best practices
3. Cross-reference related categories when applicable

## File Format Convention

Every `.md` file in skills/ follows this frontmatter format:

```yaml
---
name: short-hyphenated-name
description: One-line description
tags: [tag1, tag2, tag3]
---
```

## Contribution Guidelines

- One skill file per focused topic
- Frontmatter metadata required (name, description, tags)
- Keep content practical — best practices, code snippets, pitfalls
- Include real configuration examples where applicable

## Build / Validate

```bash
# Count skills by category
for d in skills/*/; do echo "$(find $d -name '*.md' | wc -l) $d"; done

# Validate all files have frontmatter
for f in $(find skills -name '*.md'); do head -3 "$f" | grep -q "^---$" && echo "✓ $f" || echo "✗ missing frontmatter: $f"; done
```
