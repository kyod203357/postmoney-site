# PostMoney Marketing Website

## Project Overview
Static HTML/CSS marketing site for PostMoney — a portfolio management SaaS for boutique fund managers and family offices. No build step, no JavaScript framework.

## Tech Stack
- Pure HTML/CSS (no framework, no build tools)
- Google Fonts (Inter: 400, 500, 600, 700)
- Inline SVG icons (Heroicons v2)
- CSS custom properties for design tokens (HSL color system)
- Deployed via Cloudflare Workers (cloudflare-landing plugin)

## Pages
- `index.html` — Main landing page (hero, features, KPIs, testimonials, CTA)
- `pricing.html` — Pricing tiers and feature comparison
- `signup.html` — Early access signup form
- `faq.html` — Frequently asked questions
- `terms.html` — Terms of Service
- `privacy.html` — Privacy Policy

## Design System
- **Colors:** Defined as CSS custom properties using HSL. Minimal palette (grays) with accent colors (blue, teal, purple, amber)
- **Typography:** Inter font family. Consistent heading/body hierarchy
- **Layout:** Mobile-first responsive design with CSS media queries
- **Screenshots:** Base64-embedded product images with mobile/desktop variants

## Conventions
- All styling is inline within each HTML file (no external CSS files)
- Each page shares a common header/nav and footer structure — keep them in sync across pages
- Navigation uses relative paths (e.g., `pricing.html`, not `/pricing.html`)
- Semantic HTML with proper heading hierarchy (h1 > h2 > h3)
- No JavaScript unless absolutely necessary for a specific feature

## Deployment
- **Platform:** Cloudflare Workers via `cloudflare-landing` plugin
- **Domain:** postmoney.ai
- Use `/cloudflare-landing:deploy` skill to deploy

## SEO
- Apply SEO best practices when creating or editing pages
- Use the `/avinyc:seo` skill for SEO guidance on page changes
- Ensure proper meta tags, Open Graph tags, and semantic HTML

## Git
- Remote: `git@github.com:innovent-capital/postmoney_website.git`
- Branch: `main`
- Use conventional commit messages (feat:, fix:, style:, content:, docs:)
