# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Jekyll-based personal blog website hosted on GitHub Pages. The site features:
- Technical blog posts primarily about Unity3D, algorithms (Lua, C++), and book reviews
- Chinese language content focused on game development and software engineering
- Gitalk-based comment system integrated with GitHub Issues

## Technology Stack

- **Static Site Generator**: Jekyll 
- **Markdown Processor**: Kramdown with GitHub Flavored Markdown (GFM)
- **Frontend**: Bootstrap 4, jQuery, FontAwesome
- **Comment System**: Gitalk (GitHub OAuth-based)
- **Hosting**: GitHub Pages

## Development Commands

### Local Development
```bash
# Serve the site locally
jekyll serve

# Build the site
jekyll build

# The site will be available at http://localhost:4000
```

### Content Preview
The site configuration (`_config.yml`) has `future: true` enabled, which allows posts with future dates to be rendered immediately during development.

## Site Architecture

### Content Structure
```
_posts/           # Blog posts in markdown (YYYY-MM-DD-title.md format)
_layouts/         # Page templates (post.html, page.html, home.html, tag_index.html)
_includes/        # Reusable components (header, footer, sidebar, social, toc, share)
assets/           # Static assets (CSS, JS, fonts)
_site/            # Generated static site (auto-generated, ignored in git)
```

### Layout Hierarchy
- **post.html**: Main blog post layout with sidebar, TOC support, and comment integration
- **page.html**: Simple page layout for About, Archive, etc.
- **home.html**: Homepage listing
- **tag_index.html**: Tag-based post filtering

### Key Components

**Table of Contents (`_includes/toc.html`)**
- Automatically generated from markdown headers
- Enabled per-post via frontmatter: `toc: true`

**Comment System (`_layouts/post.html`)**
- Supports both Gitalk and Disqus
- Configured via `_config.yml`: `comment_provider: gitalk`
- Per-post control: `comments: true` in frontmatter
- Gitalk uses MD5-hashed pathname as issue ID

**Social Sharing (`_includes/share.html`)**
- Facebook, Twitter, LinkedIn, Pinterest, Email sharing
- Configured in `_config.yml` under `share:`

## Post Frontmatter

Standard frontmatter for blog posts:
```yaml
---
layout: post
title: 【Lua】标题
date: 2022-03-24 23:47
description: 简短描述
toc: true              # Enable table of contents
comments: true         # Enable comments
categories:
 - blog
tags:
 - Algorithms
---
```

## Configuration Notes

### Time Zone and Future Posts
- Timezone: `Asia/Shanghai`
- Future posts are enabled: `future: true`
- This allows publishing posts dated in the future

### Comment System Setup
Gitalk configuration in `_config.yml`:
- Uses GitHub OAuth App credentials
- Creates GitHub Issues automatically for each post
- Labels issues with post tags + "Gitalk"
- Issue ID uses MD5 hash of pathname to stay within GitHub's character limit

### URL Structure
- Permalink format: `/:year/:month/:day/:title/`
- Development URL: `http://localhost:4000`
- Production: Served via GitHub Pages

## Content Categories

The blog covers three main content types:
1. **Technical Posts**: Unity3D, game development, algorithms (Lua, C++), data structures
2. **Book Reviews**: Philosophy, business, personal development
3. **Project Documentation**: Unity tools, Cinemachine, coordinate systems

## Working with Posts

### Creating a New Post
1. Create file in `_posts/` with format: `YYYY-MM-DD-title.md`
2. Add proper frontmatter (layout, title, date, tags, categories)
3. Set `toc: true` if the post needs a table of contents
4. Set `comments: true` if comments should be enabled
5. Use Chinese for titles and content (this is a Chinese-language blog)

### Post Content Conventions
- Code blocks use standard markdown triple backticks with language hints
- Mathematical content may include inline HTML (`<font color="red">` for emphasis)
- Algorithm posts typically include step-by-step analysis followed by code implementation

## Jekyll Specific

### Excluded Files
The following are excluded from Jekyll build (in `_config.yml`):
- README.md
- LICENSE

### Markdown Configuration
- Input: GFM (GitHub Flavored Markdown)
- Hard wrap enabled: `hard_wrap: true`

## Navigation Structure

Main site sections:
- **Home**: Post listing
- **Archive**: Chronological post archive
- **Blog**: Blog category posts
- **Note**: Note category posts
- **Tags**: Tag-based navigation
- **About**: Author information
