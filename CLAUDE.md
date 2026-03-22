# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal website for Oddur Magnusson, built with Hugo and the PaperMod theme, hosted on GitHub Pages at oddur.me.

## Development

All tools are managed through devbox. Enter the dev shell with `devbox shell` or prefix commands with `devbox run --`.

```bash
devbox run -- task dev          # Start dev server (includes drafts)
devbox run -- task build        # Production build (--minify)
devbox run -- task new -- slug  # Create new post: content/posts/slug.md
devbox run -- task clean        # Remove public/ and resources/
devbox run -- task theme-update # Update PaperMod submodule
```

## Architecture

- **Hugo** static site with **PaperMod** theme (git submodule in `themes/PaperMod/`)
- Content lives in `content/` as markdown with TOML frontmatter (`+++`)
- Posts go in `content/posts/`, standalone pages in `content/` root
- Custom layout overrides go in `layouts/` (currently empty — all from theme)
- Static assets (like the `CNAME` file) go in `static/`
- Site config is in `hugo.toml` (home-info mode, auto dark/light theme, GA4 analytics)

## Deployment

Push to `main` triggers `.github/workflows/hugo.yml` which builds and deploys to GitHub Pages. The `static/CNAME` file must remain for the custom domain to work.

## Post Frontmatter

```toml
+++
date = '2026-03-22T11:41:03+01:00'
draft = false
title = 'Post Title'
+++
```

Set `draft = true` to hide from production builds (visible with `task dev`).

## Writing Style

### Voice
- First person, practitioner tone. Writing from experience, not authority.
- Direct and confident but not preachy. State what you think without performing it.
- Comfortable saying "I" and grounding claims in personal practice ("I've seen", "I keep coming back to").

### Structure
- Lead with insight, not complaints. Frame around what you've learned, not what others are doing wrong.
- Remove throat-clearing. No "I'm not going to cover everything here" or "let me start by saying." Just start.
- Sections should earn their space. If a section makes one point, it gets one paragraph, not three.
- Front-load the strongest framing, then get into specifics.

### Sentence Level
- No em dashes. Ever.
- No staccato fragment pairs used for rhythm ("Statement. Fragment." / "X is Y. It's also Z."). These read as LinkedIn/marketing copy.
- No mirrored sentence constructions ("X doesn't become Y. It becomes Z.").
- No punchy closers at the end of sections ("That's the flywheel." / "That's not engineering. That's hoping."). Let the argument close itself.
- No "Here's the thing:" or "Here's where X happens" openers.
- Prefer natural flowing sentences over constructed rhetorical beats.
- Tighten aggressively. If two sentences say the same thing, one of them goes.

### Tone
- Expert but not performative. No TED talk punchlines.
- Less cleverness, more directness. Say the thing.
- Don't hedge to sound balanced just for the sake of it. If you think something, say it.
- Don't qualify every strong claim with an immediate counterpoint. It's okay to be opinionated.

### Technical Writing
- Explain patterns by walking through how they work concretely, not by naming them abstractly ("issue a proof type, consume it" is too abstract; show the code and explain what's happening).
- Use real examples with real bug scenarios, not hypothetical "imagine if" setups.
- Code examples should show the full picture in one block rather than splitting across multiple fragments with prose between them.
- When explaining a mechanism, ground it in a problem first ("Take a database pool. You don't want anyone running queries before it's initialized.").

### Comparisons
- Don't weave comparisons to other languages throughout every section. Make the case for the thing you're writing about, then address alternatives in a dedicated section.
- Give credit where due without over-hedging.

### What to Avoid
- Marketing cadence and fragment lists used for rhythm ("More signals. More constraints. Fewer bad decisions.")
- "This is a given. It's just on." style throwaway lines.
- Sentences that exist only to sound good rather than communicate ("You should expect more from your toolchain.")
- Preambles before opinions ("I'll say something that might be unpopular:")
- Filler paragraphs that scope or disclaim ("There's a lot to say about X. I won't cover it all here.")
- Repetition across paragraphs. If the point is made, move on.
