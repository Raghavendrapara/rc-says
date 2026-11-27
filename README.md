# Raghavendra Singh Chauhan — portfolio & writing

Live: **https://raghavendrapara.github.io/rc-says/**

A single-page portfolio plus a writing section for articles, research notes and work logs.
GitHub Pages builds it with Jekyll on every push to `main`, so there is nothing to install or deploy.

## Publishing an entry

1. Copy `_templates/entry.md` to `_posts/YYYY-MM-DD-short-slug.md` (the date is the publish date).
2. Fill in the front matter, then write in Markdown.
3. Commit and push to `main`. The site updates in about a minute. Check the **Actions** tab
   ("pages build and deployment") if it doesn't.

Entry types, set with `type:` in front matter:

| type | label | use it for |
|---|---|---|
| `article` | Article | essays, opinions, explainers |
| `research` | Research | deep-dives and experiments; supports `abstract`, `repo`, `demo`, `paper` |
| `log` | Work log | short build notes and progress updates |

Extras: `math: true` turns on LaTeX (KaTeX; write `$$...$$` on its own line for display maths, inside a sentence for inline), `canonical_url` credits the original source when
you cross-post, and `image` sets the link-preview image. Put images and PDFs under `assets/`.

Work in progress goes in `_drafts/`, which is never published.
`_drafts/example-research-entry.md` shows every element the layout supports.

## What's where

| Path | What it is |
|---|---|
| `index.html` | The portfolio page (experience, projects, skills, education) |
| `writing/index.html` | The writing list, filterable by type |
| `_posts/` | Published entries |
| `_layouts/`, `_includes/` | Shared page shell, nav, footer, entry layout |
| `style.css` | All styles; theme colours are CSS variables at the top |
| `_config.yml` | Site settings: name, links, résumé file, entry types |

The RSS feed is at `/rc-says/feed.xml`. Cross-posting tools such as dev.to's RSS import and
Substack's importer can read it.

## Previewing locally (optional)

```bash
gem install jekyll jekyll-feed jekyll-seo-tag
jekyll serve --drafts --baseurl /rc-says
# open http://localhost:4000/rc-says/
```

Code is open for reuse. Personal content and résumé details are not.
