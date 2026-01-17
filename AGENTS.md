# AGENTS.md

## Build/Lint/Test Commands
- **Build site**: `hugo`
- **Dev server**: `hugo server --buildDrafts`
- **Theme assets**: `cd themes/blowfish && npm run build`
- **Format code**: `cd themes/blowfish && npx prettier --write .`
- **No tests**: This is a static site with no test suite

## Code Style Guidelines
- **Formatting**: Prettier with custom config (2 spaces, double quotes, semicolons)
- **Hugo templates**: Go template syntax with bracket spacing
- **Markdown**: Standard with TOML front matter (+++ delimiters)
- **Naming**: kebab-case for files, camelCase for JS variables
- **Imports**: Group by type, standard library first
- **Error handling**: Graceful degradation in templates
- **Comments**: Minimal, self-documenting code preferred

## Creating Blog Posts

### Important: Post Date Handling
- Hugo config has `buildFuture = false`, which means posts with future dates won't be displayed
- Always set the `date` field in frontmatter to current or past time
- If a post isn't showing up in `hugo server`, check if the date is in the future
- Example: If current time is 16:11, don't use 18:00 on the same day
- After creating a post, if it doesn't appear, try clearing cache with `rm -rf public resources` and restart Hugo server