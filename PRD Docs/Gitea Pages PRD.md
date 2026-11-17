# PRD: Gitea Pages

| | |
|---|---|
| **Status** | Draft v0.4, in review |
| **Author** | Mara Lindqvist (Product) |
| **Engineering lead** | Tomasz Okafor |
| **Design** | Priya Venkataraman |
| **Contributors** | Jonas Reiter (Actions), Wen-Li Hsu (Infrastructure) |
| **Last updated** | 12 November 2026 |
| **Target release** | 1.28 (feature freeze 22 January 2027) |
| **Tracking issue** | go-gitea/gitea#23521 |

## 1. Summary

Gitea Pages lets any repository publish a static website directly from Gitea: documentation, project landing pages, API references, internal handbooks. A site can be published from a branch, from an Actions artifact, or built by Gitea itself. Sites get a URL on the instance's Pages domain, can be mapped to custom domains with automatic TLS, can be restricted to people who have access to the repository, and get a live preview for every pull request.

Pages is the most-discussed missing feature in the tracker (#23521 has been open since March 2023) and the single most common reason our design partners still keep a second hosting system next to Gitea.

## 2. Background and problem

Today, people who want to host a site from a Gitea repository have three options, and none is good:

1. **Run a third-party pages server** (Codeberg's pages-server, the Caddy `gitea-pages` plugin, or a home-grown nginx job). These need a separate deployment, an admin token with read access to every repository, and their own TLS setup. Six of our nine design partners run one of these; four of them described it as "the thing nobody wants to own".
2. **Push the built site somewhere else** (S3, Netlify, a VM) from an Actions workflow. This works, but it moves credentials for an external host into repository secrets and breaks down for private documentation, because the external host knows nothing about Gitea permissions.
3. **Don't publish.** Three partners told us internal documentation lives as raw Markdown in the repository because publishing it "properly" was never worth the setup.

What we heard in 14 interviews across 9 organisations between August and October 2026:

- "Our docs are in the repo. I want them on a URL. I don't want a ticket to the platform team for it." (docs lead, Halvorsen Robotics)
- "Private docs are the whole point for us. If I have to make the site public or put it behind a VPN, it's a non-starter." (platform engineer, Kestrel Mutual)
- "Reviewing a docs PR by reading a Markdown diff is useless. I need to see the page." (maintainer, OpenLattice)
- "We have 40 Raspberry-Pi-class instances in schools. There is no runner. There will never be a runner." (IT coordinator, Tidewater Schools Trust)

Migration is the other driver. Of the 31 inbound migration conversations the community team logged in Q3, 11 named "we use GitHub Pages" as a blocker or a major cost.

## 3. Goals

- **G1. Zero-to-published in under five minutes** for a repository that already contains static files or a supported site generator, with no admin involvement after the instance-level setup.
- **G2. Work on every instance size**, including instances with no Actions runner and no container runtime.
- **G3. Private sites that follow repository permissions**, so internal documentation can be published without a VPN or a second login system.
- **G4. Custom domains with automatic TLS**, without the site owner needing to talk to the instance admin.
- **G5. Pull request previews** so reviewers see the rendered result of a change.
- **G6. Stay inside Gitea's existing trust model.** Pages must not give anyone access they do not already have: people without write access to a repository never obtain its secrets, and access to private content is always governed by repository permissions.
- **G7. Drop-in for GitHub Pages and Netlify users**: existing Jekyll sites, `CNAME` files and `_redirects` files should work unchanged.

## 4. Non-goals

- Server-side rendering, functions, or any dynamic hosting. Pages serves files.
- Running arbitrary user-defined build commands on the Gitea host. Anything beyond the built-in generators belongs in Actions.
- A CDN. Admins who need one can put it in front of the Pages listener.
- Form handling, analytics, A/B testing, or identity features beyond "does this visitor have access".
- Replacing the wiki. Wikis remain the low-ceremony option.

## 5. Personas

- **Dana, docs maintainer.** Has write access to two or three repositories, is not a repository admin, and is not an engineer by training. Owns the docs site end to end and is the person who gets paged when a link is broken.
- **Ravi, repository admin.** Owns the repository settings and decides which units are enabled. Wants to turn Pages on once and not be asked again.
- **Sofia, instance admin.** Runs Gitea for 30 to 3,000 people. Wants one config section, predictable resource use, and an audit trail.
- **Eli, external contributor.** Has no access beyond what is public. Opens pull requests from a fork, mostly typo and docs fixes.
- **Noor, reader.** May or may not have a Gitea account. Follows a link and expects a page.

## 6. Functional requirements

### 6.1 Enabling Pages and permissions

Pages is a new repository unit, `TypePages`, alongside Code, Issues, Wiki, Packages and Actions. It appears under **Settings → Units** and gets its own settings page at **Settings → Pages**.

Research was unambiguous that the people who run docs sites (Dana) are usually not repository admins, and that every extra hand-off to an admin is where sites go stale. The permission model therefore keeps the one-time decision with admins and everything day-to-day with maintainers:

| Action | Required permission |
|---|---|
| Enable or disable the Pages unit | Repository admin |
| Change the source (branch, folder, artifact, built-in generator) | Write access to Code |
| Trigger, cancel or roll back a deployment | Write access to Code |
| Change site visibility (public, internal, private) | Write access to Code |
| Add or remove custom domains | Write access to Code |
| View deployment history and logs | Read access to Code |

Site visibility is independent of repository visibility. A private repository may have a public site (the common "closed source, open docs" case) and a public repository may have a private site.

Organisation owners can set a default visibility for new sites in the organisation and can disable the Pages unit for the whole organisation.

### 6.2 Sources and deployments

A site has exactly one source:

- **Branch.** A branch and a folder (`/` or `/docs`). Every push to the branch creates a deployment of the folder's contents as-is.
- **Actions artifact.** A workflow uploads an artifact named `gitea-pages`; when the run succeeds, Gitea creates a deployment from it. This is the recommended path for anything with a build step. We will publish `actions/upload-pages-artifact`-compatible and `actions/deploy-pages`-compatible actions so migrated workflows keep working.
- **Built-in generator.** For instances without runners (G2), Gitea builds the site itself. The repository selects a generator in `.gitea/pages.yml`: `jekyll`, `hugo` or `mdbook`. On each push to the source branch, Gitea checks the branch out into a temporary directory, runs the generator there, and deploys the output directory.
  - The generator binaries are discovered on the host `PATH`; the admin page shows which are available.
  - Builds run as a child process of Gitea under the same OS user, without a container, since the instances this is for typically have no container runtime. A build is killed after `BUILD_TIMEOUT` (default 10 minutes).
  - For GitHub Pages compatibility (G7), Jekyll builds honour the repository's `_config.yml`, `Gemfile` and `_plugins/` directory, and Hugo builds honour the repository's `config.toml` and modules, so existing sites build without changes.
  - Build output is streamed to a deployment log visible on the Pages settings page.

Deployments are immutable and numbered. The site always serves the most recent successful deployment; any earlier one can be re-activated with **Roll back**. The last 10 deployments are kept per site (`KEEP_DEPLOYMENTS`).

Artifact and branch contents are unpacked into Pages storage (`[storage.pages]`, same backends as other storage types). Archives are limited to `MAX_SITE_SIZE` (default 1 GiB) and 50,000 files; symlinks, hard links and entries resolving outside the site root are rejected and fail the deployment.

**Pages secrets and variables.** Sites commonly need a few values at build time, such as a search index key, an analytics ID or a token for a private package registry. These are stored as ordinary Actions secrets and variables at the repository and organisation level, and are injected into both artifact-based deployments (as the workflow already sees them) and built-in generator builds (as environment variables). To make previews useful, **pull request preview builds run with the same Pages secrets as the production site, including for pull requests opened from forks**, so a contributor's preview renders exactly like the real thing rather than with blank values.

### 6.3 Serving and site URLs

Each site is reachable at a default URL on the instance's Pages domain. To avoid requiring every operator to set up wildcard DNS and a second certificate before they can try the feature, the default rollout serves sites as a path on the **main instance domain** — `https://<instance-host>/-/pages/<owner>/<repo>/` — and the admin can later switch the instance to per-site subdomains (`<repo>--<owner>.<pages-domain>`) once wildcard DNS is in place. The path-based default is what new instances get out of the box.

Static files are served with long-lived cache headers. HTML is served with the site's configured headers; a `_headers` file in the site root (Netlify-compatible) can set additional response headers per path.

### 6.4 Private and internal sites (visibility)

- **Public** sites are served to anyone.
- **Internal** sites are served to any signed-in user on the instance.
- **Private** sites are served only to users who have read access to the source repository.

Because static assets are requested by the browser without carrying Gitea's normal API auth, private and internal sites use a dedicated **Pages access cookie**. On a visitor's first request to a private site, Gitea checks the visitor's read access to the repository once, then issues a signed `gitea_pages_access` cookie scoped to that site; subsequent asset requests are served whenever that cookie is present and its signature is valid, which keeps image and stylesheet loads fast without a permission lookup per file. The cookie is long-lived so that readers are not repeatedly bounced through a login check while browsing a site.

### 6.5 Custom domains and TLS

A site owner (write access to Code) adds a custom domain on the Pages settings page. Verification:

1. The owner adds a `CNAME`-style record pointing at the instance, and a `TXT` record containing a Gitea-issued token.
2. Gitea verifies the domain by **fetching `http://<custom-domain>/.well-known/gitea-pages-challenge`** and checking the token, and by resolving the `TXT` record. Verification is also re-run whenever the owner clicks **Re-check**.
3. On success the domain is marked verified and a certificate is obtained via ACME.

A verified domain binds to the site. If a domain is removed from a site, it is released and **immediately available for any other site to add and verify**, so owners who mistype a domain can re-add it without waiting.

### 6.6 Redirects and rewrites

For Netlify and GitHub Pages compatibility (G7), a `_redirects` file in the site root defines redirect and rewrite rules. Rules support:

- Client redirects (301/302) to another path or absolute URL.
- **Rewrites / proxying**: a rule with a `200` status and an absolute-URL target makes Gitea fetch the target server-side and return its body under the site's path, so a site can place an API or an asset host "behind" its own domain without CORS. This mirrors Netlify's rewrite proxy.

### 6.7 Pull request previews

When the Pages unit is enabled and a pull request changes the source, Gitea builds a preview deployment and posts its URL as a status check and a comment. Previews use the same build path as production (artifact or built-in generator) and are torn down when the pull request closes. See 6.2 for how previews handle secrets.

## 7. Rollout plan

- **1.28, behind `[pages] ENABLED = false`.** Branch and artifact sources, public and internal visibility, path-based serving on the main instance domain, built-in Jekyll generator. Dogfood on our own docs.
- **1.29.** Private visibility and the Pages access cookie, custom domains with ACME, `_redirects`, Hugo and mdBook generators, PR previews.
- **1.30.** Optional per-site subdomain serving, rollback UI, organisation-level defaults, migration actions for GitHub Pages workflows.

## 8. Metrics

- Time from enabling the unit to first successful deployment (target: p50 under 5 minutes).
- Sites published per week; share using built-in generators vs artifacts.
- Migration conversations where Pages is named as resolved.
- Build success rate and p95 build time for built-in generators.

## 9. Open questions

- Should built-in generator builds be opt-in per instance, given they run generator plugins from the repository? (Infra leans yes; Product worries it re-introduces the admin hand-off G1 is trying to remove.)
- Do we need per-path visibility, or is per-site enough for 1.29?
- Quota model for storage and build minutes on large instances.

## Appendix A — Design partners

| Organisation | Primary contact | Size | Driver | Committed to pilot |
|---|---|---|---|---|
| Halvorsen Robotics | docs lead | ~120 eng | Docs on a URL, no platform ticket | Yes (1.28) |
| Kestrel Mutual | platform engineer | ~400 eng | Private internal docs | Yes (1.29) |
| OpenLattice | maintainer | OSS, ~30 core | PR previews for docs | Yes (1.28) |
| Tidewater Schools Trust | IT coordinator | 40 small instances | Built-in generator, no runners | Yes (1.28) |
| Meridian Freight | SRE | ~250 eng | GitHub Pages migration | Interested |
| Nordvik University | research IT | ~2,000 users | Per-project sites, custom domains | Yes (1.29) |
| Caldera Studios | tech artist | ~60 | Marketing + docs from one repo | Interested |
| Beacon Health Co-op | security eng | ~180 | Private docs following repo perms | Yes (1.29) |
| Aether Payments | staff eng | ~300 | Closed source, open docs | Yes (1.29) |
