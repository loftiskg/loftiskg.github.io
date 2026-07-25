---
title: 'How this site works'
description: 'A runbook for adding posts, pages, and collections to an Astro site deployed on GitHub Pages — plus the gotchas that cost me time the first go round.'
pubDate: 2026-07-25
tags: ['astro', 'github-pages', 'how-to', 'meta']
---

This site is a static [Astro](https://astro.build) build, deployed to GitHub Pages by Actions on every push to `main`. It replaced the default Jekyll scaffold that had been serving GitHub's own Markdown tutorial here for years.

This post is the reference I want when I come back in six months having forgotten all of it.

## Quick reference

```bash
npm run dev      # dev server at localhost:4321, hot reload
npm run build    # production build into dist/
npm run preview  # serve the real dist/ output locally
```

Always run `build` before pushing. The dev server is more forgiving than the production build — bad frontmatter and broken imports frequently only surface at build time.

Deploy is just `git push origin main`. The workflow at `.github/workflows/deploy.yml` builds and publishes. Nothing else to do.

## Adding a post

Drop a Markdown file in `src/content/writing/`. The filename becomes the URL slug, so `src/content/writing/why-x-is-hard.md` publishes at `/writing/why-x-is-hard/`.

The frontmatter schema is enforced at build time:

```yaml
---
title: 'Some title'
description: 'One sentence. Used for the post list and the meta description.'
pubDate: 2026-07-25
tags: ['optional', 'list']
draft: false
---
```

`title`, `description`, and `pubDate` are required — a build fails loudly if any is missing, which is the point. `tags` and `draft` both have defaults and can be omitted entirely.

### Drafts

Set `draft: true` and the post is visible on the dev server but excluded from the production build. The filter lives in two places, and both need to agree:

```js
await getCollection('writing', ({ data }) =>
  import.meta.env.PROD ? !data.draft : true
);
```

That's in `src/pages/writing/index.astro` and `src/pages/writing/[...slug].astro`. It means half-finished posts can sit on `main` indefinitely without publishing — which matters, because the alternative is keeping drafts on a branch and rebasing every time the site changes underneath them.

Drafts show a small amber `draft` badge in the local post list so it's obvious what state you're looking at.

## Adding a standalone page

Any `.astro` file under `src/pages/` becomes a route by its path. `src/pages/about.astro` serves at `/about/`. No registration step.

The minimum viable page:

```astro
---
import Layout from '../layouts/Layout.astro';
---

<Layout title="About — Kevin Loftis" description="Who I am.">
  <main>
    <h1>About</h1>
  </main>
</Layout>
```

`Layout.astro` takes optional `title` and `description` props. Title falls back to `Kevin Loftis`; description is omitted from the head entirely if not passed. The canonical URL is derived automatically from `Astro.site` and the current path, so it needs no per-page wiring.

Note that page styles are currently **scoped per page**, not global. That was deliberate — it kept the writing section from disturbing the homepage during the rebuild — but it does mean typography is duplicated between the two writing routes. If a third page needs the same treatment, that's the moment to lift the shared styles into `Layout.astro` rather than copy them a third time.

## Adding a new collection

Say projects. Three steps.

**1.** Register it in `src/content.config.ts`:

```ts
const projects = defineCollection({
  loader: glob({ pattern: '**/*.md', base: './src/content/projects' }),
  schema: z.object({
    title: z.string(),
    description: z.string(),
    url: z.string().url().optional(),
  }),
});

export const collections = { writing, projects };
```

**2.** Create `src/content/projects/` and put Markdown files in it.

**3.** Add routes under `src/pages/projects/` — an `index.astro` listing them and a `[...slug].astro` for detail pages, if the projects warrant writeups. The writing routes are a working template to copy.

Two API details that are easy to get wrong, both of which changed in recent Astro versions:

- Import `z` from `astro/zod`, **not** from `astro:content`. The `astro:content` re-export is deprecated and gone in Astro 7.
- Rendering an entry is `const { Content } = await render(post)` — a standalone `render()` imported from `astro:content`. The older `post.render()` method form no longer applies.

Use `z.coerce.date()` for any date field. YAML hands you a string, and plain `z.date()` fails validation on every single entry.

## Gotchas

Four things that cost me time, all still live traps.

### Don't set `base` in `astro.config.mjs`

Most Astro-on-Pages guides say to. For this repo it's wrong, and the distinction isn't loudly advertised:

- **Project sites** at `username.github.io/repo-name/` need `base: '/repo-name'`.
- **User sites** — a repo named exactly `<username>.github.io`, which this is — serve from the domain root and need no `base`.

Set it here and every asset resolves to `/loftiskg.github.io/_astro/...` and 404s. Correct HTML, zero styling. The whole config is:

```js
export default defineConfig({
  site: 'https://loftiskg.github.io',
});
```

`site` is still worth setting — sitemap and canonical URL generation both depend on it.

### Jekyll eats underscore directories

Astro emits hashed assets into `_astro/`. Jekyll ignores every path starting with an underscore — that's the mechanism behind `_layouts` and `_posts`. If a Jekyll build ever touches Astro output, it silently strips the entire CSS and JS bundle.

Deploying through Actions avoids this entirely; the artifact uploads verbatim. But note the symptom is *identical* to the `base` mistake above — unstyled page, correct markup — and the causes are unrelated. Check both.

### Pin the Node version in CI

The first deploy failed on:

```
Node.js v20.20.2 is not supported by Astro!
Please upgrade Node.js to a supported version: ">=22.12.0"
```

The local build had succeeded in 195ms, because this machine runs Node 25. The runner defaulted to 20. Fix:

```yaml
- uses: withastro/action@v3
  with:
    node-version: 22
```

`engines` in `package.json` documents the requirement but does not enforce it. A green local build says nothing about CI unless something pins the runtime in both places.

### Green deploy, stale site

If Actions is green and the live site doesn't change, check **Settings → Pages → Source**. If it says *Deploy from a branch*, GitHub is running its own build and ignoring your artifact.

Worth knowing: this is not something you have to configure in advance. `actions/deploy-pages@v4` flips the build type from `legacy` to `workflow` itself on the first successful deploy. But when deploys go green and nothing moves, this is still the first place to look.

## What's not built yet

Recording these so the gaps are deliberate rather than forgotten:

- The homepage is still the Astro scaffold splash and doesn't link here.
- No projects collection.
- No RSS feed. `@astrojs/rss` is the standard answer when it matters.
- No sitemap. `@astrojs/sitemap` needs `site` set, which it already is.
- `package.json` still says `"name": "astro-tmp"`, a fossil from the temp directory it was scaffolded in.
