# Agent Guidelines for Zenn Documentation Repository

## Repository Purpose
This repository manages Zenn articles and books. It contains markdown files following Zenn's frontmatter format.

## Working with Articles

### Article Structure
- Articles are stored in the `articles/` directory
- Each article is a markdown file with YAML frontmatter
- Required frontmatter fields: `title`, `emoji`, `type`, `topics`, `published`

### Creating New Articles
1. Create a new markdown file in `articles/` with a descriptive filename (use kebab-case)
2. Include the required frontmatter:
   ```yaml
   ---
   title: "Article Title"
   emoji: "📝"
   type: "tech" # or "idea"
   topics: ["topic1", "topic2"]
   published: true
   ---
   ```
3. Write article content below the frontmatter

### Books
- Book manuscripts are stored in the `books/` directory
- Currently contains only a `.gitkeep` placeholder

## Git Workflow
- Standard git add/commit/push workflow applies
- No special build/test processes required
- Preview articles locally using Zenn CLI if available:
  ```bash
  # Install Zenn CLI (if needed)
  npm init -y
  npm install zenn-cli
  
  # Preview
  npx zenn preview
  ```

## File Naming Conventions
- Use kebab-case for article filenames (e.g., `my-article-title.md`)
- Keep filenames descriptive but concise
- Use .md extension for all markdown files

## Validation
- Ensure frontmatter is valid YAML
- Check that all required fields are present
- Verify topics match existing tags when possible