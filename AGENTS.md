# Repository Instructions

## Scope

These instructions apply to the entire repository.

This repository is a Jekyll-based personal technical blog for Heuristic Wave. Treat `_posts/` as the primary authored content area and avoid broad theme or build-system changes unless the user explicitly asks for them.

## Source Of Truth For OpenAI Topics

When questions involve OpenAI APIs, SDKs, Codex, or other OpenAI products, use the OpenAI developer documentation MCP server as the source of truth. Avoid web search unless the MCP server cannot answer.

## Project Shape

- Static site generator: Jekyll.
- Main configuration: `_config.yml`.
- Posts live under `_posts/<category>/`.
- Reusable templates live in `_layouts/` and `_includes/`.
- Site data lives in `_data/`.
- CSS source lives in `assets/css/`.
- Generated CSS and optimized assets live in `assets/built/`.
- Jekyll output is written to `output/`.

Do not edit `output/`, `.jekyll-cache/`, or `node_modules/` by hand. They are generated or dependency directories.

## Blog Post Conventions

Use the existing post structure when creating or editing posts:

```yaml
---
layout: post
current: post
cover: assets/built/images/background/example.png
navigation: True
title: "Post Title"
date: YYYY-MM-DD 00:00:00
tags: [aws, genai]
class: post-template
subclass: "post tag-ai"
author: HeuristicWave
---
```

- File names should follow `_posts/<category>/YYYY-MM-DD-Slug.md`.
- Keep the filename date and front matter `date` aligned unless the user intentionally wants scheduled publication or backdating.
- Use `author: HeuristicWave`.
- Use `layout: post`, `current: post`, `navigation: True`, and `class: post-template` unless there is a clear local precedent for a different value.
- Set `subclass` to the primary category style, such as `post tag-ai`, `post tag-aws`, `post tag-devops`, or `post tag-extracurricular`.
- Prefer existing categories: `ai`, `aws`, `backend`, `devops`, `ppt`, `security`, and `uncategorized`.
- Prefer existing tags where they fit. Common tags include `aws`, `devops`, `terraform`, `genai`, `ai`, `llm`, `database`, `security`, `backend`, `programming`, `serverless`, `event`, `extracurricular`, and `lab`.
- Add the `lab` tag to posts that document experimental technical trials, prototypes, early architecture ideas, proof-of-concept demos, or hands-on explorations intended to be collected later as lab notes. Use it alongside the normal subject tags, for example `tags: [aws, genai, lab]`.
- If adding a new top-level tag/category that needs archive metadata, update `_data/tags.yml` consistently.

## Writing Style

- Most posts are Korean-first technical explanations with English product names and technical terms left in English.
- Preserve the author's practical, explanatory tone: start from motivation or context, then move into workflow, implementation, results, and closing notes.
- Existing posts commonly use `Intro`, `Outro`, numbered sections, emoji-prefixed headings, and occasional manual anchors like:

```markdown
## <a href="#topic">🔧 Section Title</a><a id="topic"></a>
```

- Match the style of neighboring posts in the same category rather than introducing a new article format.
- Use fenced code blocks with language identifiers such as `shell`, `python`, `yaml`, `json`, `hcl`, or `text`.
- Never include real secrets, API keys, account IDs, private endpoints, or credentials. Replace sensitive values with clear placeholders.
- For vendor APIs, cloud services, and fast-moving product behavior, verify current details before writing definitive claims.

## Images And Assets

- Front matter `cover` paths are usually root-relative, without a leading slash, for example `assets/built/images/background/aws.png`.
- Markdown images inside posts commonly use paths relative to `_posts/<category>/`, for example:

```markdown
![description](../../assets/built/images/post/example.png)
```

- Prefer storing post images under `assets/built/images/post/<topic>/` or an existing nearby folder.
- Prefer storing reusable background/cover images under `assets/built/images/background/`.
- Check that referenced image files exist before finalizing a post.

## Build And Verification

Use the smallest relevant verification for the change:

```bash
# Install Ruby dependencies when needed
bundle install

# Build the Jekyll site
bundle exec jekyll clean && bundle exec jekyll build

# Local preview
bundle exec jekyll serve

# Local preview including drafts
bundle exec jekyll serve --drafts

# Rebuild CSS after editing assets/css/
npx gulp css
```

- Run `bundle exec jekyll clean && bundle exec jekyll build` after adding or editing posts, layouts, includes, `_data/`, or `_config.yml`.
- Run `npx gulp css` after editing `assets/css/`.
- Do not treat changes under `output/` as source changes.

## Editing Discipline

- Keep changes narrowly scoped to the requested post, template, or asset.
- Preserve unrelated user changes, including untracked files.
- Use `rg` for repository searches.
- Avoid destructive git commands unless the user explicitly requests them.
- Do not rewrite generated lockfiles or built assets unless the requested change requires it.
