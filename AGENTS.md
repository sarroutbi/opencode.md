# Global OpenCode Instructions

## Commits and branches

- When asked to commit, create a kebab-case branch if not already on one.
- Avoid branch names starting with "up", "or", or "ma"; use alternatives
  such as "modify" instead of "update".
- Sign every commit (`git commit -s`).
- Do not include issue or PR numbers (e.g. `(#40)`) in commit messages.
  GitHub appends these automatically when a PR is merged.
- Never push changes unless explicitly asked.
- When there is code that has been vibe coded, add the corresponding Co-Authored-By

## Programming

- Follow Robert C. Martin's Clean Code guidance and SOLID principles.
- Use the standard tooling for each language (pylint, cargo fmt, etc.).
- Keep mocks in sync with production code changes.
- Update README.md or AGENTS.md when changes require it.
- When there is code that has been vibe coded, add the corresponding Co-Authored-By
header when committing

## Repository scope

- Only commit changes to the primary working directory repository.
  Other repositories may be read but must not be committed to from this session.

## Skills

- Before writing new code or building a feature, load the `clean-code-builder` skill.
- Before refactoring existing code, load the `fowler-beck-refactoring` skill.
