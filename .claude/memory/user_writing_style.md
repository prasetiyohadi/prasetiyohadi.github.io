---
name: Medium writing style guide
description: Pras' writing style for Medium technical blog posts (CI/CD Lab series and similar hands-on engineering content)
type: user
---

## Voice and Tone
- First-person, practitioner perspective ("I asked", "I created", "I use", "Let me walk you through")
- Conversational and direct, not corporate or polished — like a senior engineer sharing lab notes with peers
- Honest about tooling choices and trade-offs, including when things don't work out of the box
- Comfortable referencing LLM/AI as a starting point for exploration

## Structure
- **Series framing**: Posts belong to a named series (e.g., "CI/CD Lab") with continuity between posts
- **Introduction**: Brief recap of previous post, then state what this post covers and why
- **Sections**: Each section covers one concept with code, then explanation, then business/practical impact
- **Conclusion**: Summary of what was built, bullet-point tool recap, and clear business takeaway
- **"What's Next"**: Always end with a teaser for the next post, sign off with "Stay tuned for next stories with The Hands-On Engineer!"
- **Tags**: Listed at the bottom after a horizontal rule

## Code and Technical Content
- Heavy use of actual code blocks (Dockerfile, YAML, bash, JS) — not pseudocode
- Show the real configuration, not simplified examples
- Explain each code block immediately after it with bullet points or short paragraphs
- When listing variables or flags, use bullet points with bold names and inline descriptions
- Compare approaches with concrete differences (e.g., build times, security trade-offs)
- Link to the actual repository so readers can follow along

## What to Avoid
- No abstract thought-leadership framing or executive-speak
- No heavy use of em-dashes for dramatic effect
- No "The business case for X" as a rhetorical device — instead, state the practical impact plainly after showing the implementation
- No DORA metrics or framework references unless directly relevant to what was built
- Don't over-editorialize — let the code and results speak

## Formatting Preferences
- Markdown headings: H1 for title, H3 for sections (Medium renders this well)
- Horizontal rules (`---`) to separate major sections
- Inline code backticks for tool names, file paths, commands, and variable names
- Bold for tool names in summary lists
- Images/diagrams referenced where they would appear (tables, pipeline screenshots)
- Reading time target: 5-8 minutes
