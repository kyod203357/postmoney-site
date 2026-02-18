# PostMoney — Marketing Site

Static HTML marketing site for [PostMoney](https://postmoney.io), a portfolio management tool built for boutique fund managers and family offices.

## Pages

| File | URL | Description |
|------|-----|-------------|
| `index.html` | `/` | Main landing page |
| `pricing.html` | `/pricing` | Pricing tiers and feature comparison |
| `signup.html` | `/signup` | Early access sign up |
| `faq.html` | `/faq` | Frequently asked questions |
| `terms.html` | `/terms` | Terms of Service |
| `privacy.html` | `/privacy` | Privacy Policy |

## Stack

- Pure HTML/CSS — no frameworks, no build step
- Google Fonts (Inter)
- Inline SVG icons (Heroicons v2 outline)
- Base64-embedded product screenshots in `index.html`

## Design System

See `STYLE_GUIDE.md` for the full design token reference, component patterns, and copy guidelines.

Colors, spacing, and typography are defined as CSS custom properties in each page's `<style>` block. The design system is consistent across all pages.

## Deploying

This site is static and deploys anywhere:

**GitHub Pages**
Push to a repo and enable Pages from Settings → Pages → Deploy from branch (`main`, `/ (root)`).

**Netlify / Vercel**
Drop the folder or connect the repo. No build configuration needed.

## Copy & Content

Copy direction and messaging guidelines are documented in `COPY_BRIEF.md`.

## Development Notes

- `index.html` is ~730KB due to embedded base64 product screenshots
- No JavaScript dependencies
- All nav links are relative paths (works locally and deployed)
