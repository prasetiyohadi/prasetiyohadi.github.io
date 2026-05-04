# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a GitHub Pages personal blog repository (`prasetiyohadi.github.io`) hosting a technical blog on platforms like Medium. The repo contains:

- **index.html**: Simple landing page for the GitHub Pages site
- **medium/**: Drafts and published Medium blog posts in Markdown format

The project focuses on hands-on technical content, particularly the "CI/CD Lab" series covering CI/CD pipelines, security scans, and DevOps practices.

## Writing and Content

### Medium Blog Posts

All blog posts follow a specific first-person, hands-on engineering style documented in `.claude/memory/user_writing_style.md`. Key conventions:

- **Format**: Markdown files in the `medium/` directory
- **Structure**: Series framing → brief intro/recap → sections with code → conclusion → "What's Next" teaser
- **Code**: Heavy use of real configuration files (Dockerfile, YAML, bash) with immediate explanations
- **Voice**: First-person practitioner perspective, conversational and direct, honest about trade-offs
- **Length**: Target 5-8 minutes reading time
- **Closing**: Always end with "Stay tuned for next stories with Pras' Labs: The Hands-On Engineer!"

When drafting or editing blog posts, consult the detailed style guide in the memory file for formatting, code block conventions, and what to avoid (e.g., no corporate jargon, no heavy em-dashes, no abstract thought-leadership).

## File Organization

```
.
├── index.html             # Main GitHub Pages landing page
├── README.md              # Repo description
├── medium/                # Blog post drafts and content
│   ├── cicd-lab-*.md      # CI/CD Lab series posts
│   └── *.md               # Other blog content
└── .claude/
    └── memory/            # Claude Code memory (writing style, preferences)
```

## Development Notes

- No build process or dependencies required — this is a static content repository
- No tests, linting, or CI/CD pipelines in the repo itself (only documented in blog content)
- GitHub Pages serves `index.html` directly from the repository root
- Blog posts are versioned alongside the code for reference and continuity

When working on blog content or the landing page, ensure changes align with the repository's focus on accessible, practical technical engineering content.

## Social Media Post Generation

Use the `/social-post` slash command to generate platform-specific social media content from a finished Medium post:

```
/social-post medium/my-post.md
```

Omit the path to auto-detect the most recently edited `.md` file in `medium/`.

Generates ready-to-copy posts for **Twitter/X** (2-tweet chain), **LinkedIn**, **email subject lines** (4 variations), **Bluesky** (2-post chain), **Threads** (3-post chain), and **Facebook** — all in Pras' first-person practitioner voice, using the actual tool names from the post.

After drafting with `/medium-story`, it will remind you to run `/social-post` as a follow-up step.
