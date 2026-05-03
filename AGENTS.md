# AGENTS.md

## Commands

- `pnpm dev` - Start dev server (port 4321)
- `pnpm build` - Build to `./dist/`
- `pnpm preview` - Preview production build
- `pnpm astro` - Run Astro CLI commands

No test, lint, or typecheck scripts configured.

## Stack

- **Framework**: Astro 5 with Vue 3 components
- **Package manager**: pnpm (not npm)
- **Styling**: Tailwind CSS v4 + shadcn-vue (New York style)
- **Content**: Markdown + MDX with GFM, KaTeX math, Mermaid diagrams

## Project Structure

- `src/content/blog/` - Blog posts (.md/.mdx) with frontmatter: `title`, `description`, `pubDate`, `tags`, `categories`, `heroImage`
- `src/components/ui/` - shadcn-vue components
- `src/assets/fonts/` - Local font LXGWZhenKai (used as body font)

## Key Config

- Path alias: `@/*` → `./src/*`
- Site URL: `https://junsen.online`
- TypeScript: Strict mode (extends `astro/tsconfigs/strict`)
- Markdown plugins apply to both `.md` and `.mdx` files

## Migration Tool

`node convert-frontmatter.js` - Converts Hexo-style frontmatter to Astro format (run once, idempotent)
