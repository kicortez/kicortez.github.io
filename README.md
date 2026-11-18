# kicortez.github.io

Personal iOS developer portfolio, built with Jekyll and served by GitHub Pages from `master`.

## Run locally

Needs Ruby 3.3 (GitHub Pages' gem set doesn't support Ruby 4 yet).

```sh
brew install ruby@3.3
export PATH="/opt/homebrew/opt/ruby@3.3/bin:$PATH"   # add to ~/.zshrc to make permanent
bundle install
bundle exec jekyll serve --livereload
```

Open http://localhost:4000. Pages reload on save; restart the server after editing `_config.yml`.

## Where content lives

| What | File |
|---|---|
| Name, bio, links, skills | `_data/profile.yml` |
| Work history | `_data/experience.yml` |
| Project cards | `_data/projects.yml` (`case_study: <slug>` adds a "Read case study" link) |
| Case studies | `_case_studies/<slug>.md` → `/case-studies/<slug>/` |
| Images | `assets/images/` |
| Styles | `assets/css/main.scss` |

### Add a case study

Create `_case_studies/my-new-study.md`:

```yaml
---
title: My new study
summary: One-line description shown on cards.
hero: /assets/images/my-hero.png
role: Lead iOS Engineer
timeline: 3 months · 2025
platforms: [iOS]
stack: [SwiftUI, Swift Concurrency]
order: 4          # position in lists and prev/next links
featured: true    # also show on the home page
---
```

Then write Markdown. Helpers:

```liquid
{% include figure.html src="/assets/images/x.png" alt="..." caption="..." class="narrow" %}
{% include gallery.html images="/assets/images/a.png,/assets/images/b.png" caption="..." %}
[Link to another study]({{ '/case-studies/other-slug/' | relative_url }})
```

## Deploy

Work on `dev`, preview locally, then merge `dev` → `master` and push. GitHub Pages builds automatically.
