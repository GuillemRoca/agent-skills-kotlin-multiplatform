# Cursor Setup

## Rules Directory (Recommended)

Create a `.cursor/rules/` directory in your project and copy skills into it:

```bash
# Clone the repo
git clone https://github.com/GuillemRoca/agent-skills-kotlin-multiplatform.git

# Copy core skills into your project
mkdir -p your-project/.cursor/rules
cp agent-skills-kotlin-multiplatform/skills/test-driven-development/SKILL.md your-project/.cursor/rules/test-driven-development.md
cp agent-skills-kotlin-multiplatform/skills/code-review-and-quality/SKILL.md your-project/.cursor/rules/code-review-and-quality.md
cp agent-skills-kotlin-multiplatform/skills/incremental-implementation/SKILL.md your-project/.cursor/rules/incremental-implementation.md
```

Rules in `.cursor/rules/` are automatically loaded into Cursor's context.

## Single Rules File

Alternatively, combine skills into a single `.cursorrules` file at your project root:

```bash
# Combine essential skills into one file
cat agent-skills-kotlin-multiplatform/skills/test-driven-development/SKILL.md > your-project/.cursorrules
echo -e "\n---\n" >> your-project/.cursorrules
cat agent-skills-kotlin-multiplatform/skills/incremental-implementation/SKILL.md >> your-project/.cursorrules
echo -e "\n---\n" >> your-project/.cursorrules
cat agent-skills-kotlin-multiplatform/skills/code-review-and-quality/SKILL.md >> your-project/.cursorrules
```

## Notepads

Store skills as reusable Notepads in Cursor's settings. Reference them with `@notepad` when needed:

- **Always-load essentials:** `test-driven-development`, `code-review-and-quality`, `incremental-implementation`
- **Phase-specific notepads:** `compose-multiplatform-ui` (shared UI work), `security-and-hardening` (security reviews), `performance-optimization` (performance work), `multiplatform-architecture` (architecture decisions)

## Best Practices

- Load 2–3 core skills as permanent rules
- Keep specialized skills as on-demand notepads
- Reference agent personas (`agents/code-reviewer.md`) for targeted reviews
- Avoid overloading context — load skills relevant to the current task
