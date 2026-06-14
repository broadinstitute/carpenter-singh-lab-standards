# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Repository Context

Documentation repository for Carpenter-Singh Lab at the Broad Institute.

## Key Tasks When Working Here

### 1. Documentation Improvements

- Check for consistency in command examples across documents
- Ensure all code blocks are properly formatted and executable
- Verify that file paths and project names use consistent placeholder syntax (e.g., `{PROJECT_NAME}`)
- Look for outdated dependencies or version numbers that may need updating

### 2. Common Updates

When asked to update documentation:

- Follow the Documentation Guidelines below
- Focus on executable commands over explanations

### 3. Adding New Sections

When adding new documentation:

- Follow the existing markdown structure and heading hierarchy
- Place new content in the appropriate file; the data-analysis method lives in the [catalog-skills](https://github.com/carpenter-singh-lab/catalog-skills) repo, not here
- Use real-world examples from the referenced repositories when possible

### 4. Quality Checks

Before finalizing any changes:

- Ensure all bash commands use proper escaping and line continuations
- Verify that Python/YAML/TOML code blocks have correct syntax
- Check that any new commands follow the established patterns in the document
- Check for broken links: `nix shell nixpkgs#lychee` then `lychee <filename>.md` (403 errors from Udacity are false positives)

## Repository Structure

```text
carpenter-singh-lab-standards/
├── README.md          # Main entry point, links to other docs
├── infrastructure.md  # Computing resources and dev environment
└── CLAUDE.md         # This file
```

## When Making Changes

- New documentation must complement, not duplicate, existing content
- Make processes clear and executable, not flexible

## Documentation Guidelines

When updating documentation in this repository, follow these principles:

### Target Audience

- **Expert data scientists** who are new to the lab
- They will be instructed to read documentation thoroughly
- They want to understand THE way this lab operates, not explore options
- Assume technical competence - skip basic explanations

### Writing Style

- **Prescriptive over descriptive**: "Do X" not "You might consider X"
- **Dense over verbose**: One clear statement beats three explanatory paragraphs
- **Executable over abstract**: Every process needs exact, working commands
- **Real over generic**: Use actual filenames from real projects, not placeholders

### Content Organization

- **Role-based sections**: Clearly separate what analysts vs maintainers need
- **Most-used first**: Daily workflow before one-time setup
- **Progressive disclosure**: Core workflow in main text, edge cases in appendix
- **Decision trees**: "If X → do Y" for any branching logic

### Critical Elements to Include

- **Coordination requirements**: Bold warnings when team sync is needed
- **Order dependencies**: Explicit when sequence matters (e.g., hook installation)
- **Known gotchas**: Document issues with specific workarounds
- **The "why" sparingly**: Only when it affects implementation choices

### What to Avoid

- Beginner explanations ("Git is a version control system...")
- Multiple options without clear guidance
- Philosophical discussions about methodology
- Untested command sequences

Remember: Documentation should be the authoritative reference that turns an expert data scientist into an expert lab member.
