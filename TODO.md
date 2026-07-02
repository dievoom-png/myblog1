# Blog & Portfolio Roadmap

## Blog — ship the redesign (`blowfish-redesign` branch)
- [ ] Push branch to GitHub + open PR → Netlify deploy-preview URL → review → merge to `main` (deploys to blog.yahyaayman.com)
- [ ] Verify Netlify deploy previews are enabled for the site (Site settings → Build & deploy)

## Blog — visual polish
- [ ] Feature/cover images for the posts that have none (WannaCry, SillyPutty, mta3) — their cards are text-only; could design purple-branded covers
- [ ] Experiment with homepage layout variants: `hero`, `background`, or `card` instead of `profile` (`config/_default/params.toml` → `[homepage] layout`)
- [ ] Tune the `deep-purple` scheme if needed (`assets/css/schemes/deep-purple.css` — one line per shade)

## Blog — features to enable later
- [ ] Comments via giscus (GitHub Discussions) — Blowfish has a comments partial
- [ ] Analytics (Umami / Fathom / Google Analytics — all supported in params)
- [ ] Post view counts & likes (Blowfish built-in, needs a free Firebase project)
- [ ] Use Mermaid diagrams / KaTeX / chart shortcodes in technical write-ups (e.g. malware flow diagrams)
- [ ] About page
- [ ] Finish & publish HandOnKeylogging draft (part 2 of the Keyloggers series)

## Blog — maintenance
- [ ] Periodically `hugo mod get -u github.com/nunocoracao/blowfish/v2` for theme updates (also clears the Hugo 0.163 vs theme-max-0.161 compat warning once the theme catches up)
- [ ] Keep `HUGO_VERSION` in netlify.toml in sync with local Hugo when upgrading

## Portfolio — yahyaayman.com (apex domain)
- [ ] Decide approach:
  - Option A: separate Hugo + Blowfish repo (profile/hero layout, reuse deep-purple scheme + avatar for consistent branding)
  - Option B: custom hand-built single-page site (full design freedom)
- [ ] New Netlify site + point apex DNS at it
- [ ] Prominent link to blog.yahyaayman.com (and back-link from blog menu)
- [ ] Content: whoami, projects/tooling, certs/courses (PMAT etc.), featured write-ups pulled from the blog
