# Claude Cloudflare Agent Deploy

How a single `/cloudflare-landing:deploy` command took a static HTML site and autonomously deployed it to a live custom domain with form capture, KV storage, and an admin panel — with zero human intervention.

---

## What Happened

From a Claude Code session in the `postmoney_website` directory, the command `/cloudflare-landing:deploy` was invoked. An autonomous agent:

1. Analyzed the repository and identified all HTML pages and forms
2. Copied the site to a temporary deploy directory (never touching source files)
3. Detected the signup form in `signup.html`, extracted its field names
4. Generated a custom Cloudflare Worker tailored to those exact form fields
5. Created a KV namespace for storing submissions
6. Injected a JavaScript form handler into the signup page
7. Built a `wrangler.toml` config with the KV namespace ID and custom domain
8. Deployed the Worker and static assets to Cloudflare's edge network
9. Set admin credentials as encrypted secrets
10. Configured DNS routing for `postmoney.ai`
11. Verified every endpoint with curl (homepage, form POST, admin auth, exports)
12. Reported the live URLs and admin credentials

Total time: ~5 minutes. No manual steps.

---

## The Skill: `cloudflare-landing`

### What It Is

A Claude Code **plugin** — a bundle of skills, agents, and templates that extend Claude's capabilities for a specific workflow. This one lives at:

```
~/.claude/plugins/cache/avi-ai/cloudflare-landing/1.0.0/
```

### Plugin Structure

```
cloudflare-landing/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata (name, version, author)
├── agents/
│   └── site-deployer.md         # Autonomous agent definition
├── commands/
│   └── deploy.md                # /cloudflare-landing:deploy slash command
├── skills/
│   └── deploy/
│       ├── SKILL.md             # 13-step deployment playbook
│       └── templates/
│           ├── worker.js        # Generic Cloudflare Worker template
│           ├── wrangler.toml    # Config template with {{PLACEHOLDERS}}
│           └── form-handler.js  # Vanilla JS form interceptor
└── README.md
```

### How the Pieces Fit Together

**The command** (`commands/deploy.md`) is the entry point — the `/cloudflare-landing:deploy` slash command. It's a simple passthrough that invokes the skill.

**The skill** (`skills/deploy/SKILL.md`) is the playbook — a 13-step procedure that the agent follows. It defines *what* to do: gather inputs, analyze the site, create deploy directory, wire up forms, create KV, deploy, verify.

**The agent** (`agents/site-deployer.md`) defines *who* does it — an autonomous subprocess with specific tool permissions. It runs as a subagent (a separate Claude instance) with access to Bash, file tools, and Cloudflare MCP tools.

**The templates** are starting points that the agent adapts to each site's specific needs.

---

## How It Inferred Everything

The agent received only two pieces of information: the site path and the domain. Everything else was discovered autonomously:

### Site Analysis

```
Input:  /Users/avi/Development/code/Innovent/Sites/postmoney_website
```

The agent ran:
- `ls` / `fd` to inventory all files (found 6 HTML pages + images directory)
- `rg '<form'` to find which pages contain forms (found `signup.html`)
- `rg 'name="[^"]*"'` to extract form field names (`first_name`, `last_name`, `email`, `fund_name`, `fund_size`)
- Checked for `package.json` — none found, so it knew this was pure static HTML (no build step needed)

### Intelligent Adaptation

The templates provide a generic starting point, but the agent **rewrote the Worker** to match what it found:

| Template (generic)           | Deployed (PostMoney-specific)              |
|-----------------------------|--------------------------------------------|
| `POST /api/submit`          | `POST /api/signup`                         |
| KV prefix: `sub:`           | KV prefix: `signup:`                       |
| Generic form field capture   | Specific fields: `first_name`, `last_name`, `email`, `fund_name`, `fund_size` |
| Generic admin table columns  | Branded columns: Date, Name, Email, Fund, Size, Country |
| Generic `data-cloudflare-form` attribute | Custom inline `<script>` with PostMoney-branded success message |
| Generic admin panel styling  | PostMoney-branded admin with Inter font, matching color scheme |

The agent didn't just fill in placeholders — it understood the site's data model and built a Worker tailored to it.

### Form Handler Injection

The original `signup.html` had a standard HTML form. The agent:

1. Copied `signup.html` to the deploy directory
2. Injected a `<script>` block before `</body>` that:
   - Intercepts the form's submit event
   - Collects all fields via `FormData`
   - POSTs them as JSON to `/api/signup`
   - On success, replaces the form with a branded "You're on the list!" confirmation (with a checkmark SVG icon)
   - On failure, shows an inline error message

The **source file was never modified** — only the copy in `/tmp/postmoney-deploy/public/`.

---

## Prerequisites: What Made This Possible

### 1. Wrangler CLI (Cloudflare's deployment tool)

```bash
bun install -g wrangler   # or npm install -g wrangler
wrangler login             # authenticates via browser OAuth
```

Wrangler is Cloudflare's official CLI for deploying Workers. `wrangler login` opens a browser window for OAuth authentication and stores the token locally at `~/.wrangler/config/default.toml`. This token gives Wrangler permission to:
- Create and deploy Workers
- Create KV namespaces
- Set encrypted secrets
- Configure custom domain routes

### 2. Cloudflare Account with the Domain

The domain `postmoney.ai` must already be added to your Cloudflare account with its nameservers pointing to Cloudflare. This is the one manual prerequisite — you set up DNS delegation once, and then the agent can configure routing automatically.

When the agent ran `wrangler domains add postmoney.ai`, Cloudflare automatically:
- Created a DNS record pointing the domain to the Worker
- Provisioned an SSL certificate
- Configured the CDN edge routing

### 3. Cloudflare MCP Server (Model Context Protocol)

Claude Code has access to Cloudflare's MCP server, which exposes Cloudflare API operations as tools:

```
mcp__claude_ai_Cloudflare_Developer_Platform__kv_namespace_create
mcp__claude_ai_Cloudflare_Developer_Platform__kv_namespaces_list
mcp__claude_ai_Cloudflare_Developer_Platform__workers_list
mcp__claude_ai_Cloudflare_Developer_Platform__accounts_list
... (20+ tools)
```

The agent used these as a fallback/alternative to Wrangler CLI commands. For example, `kv_namespace_create` can create a KV namespace without shelling out to `wrangler kv namespace create`.

### 4. The Plugin Installed in Claude Code

The plugin is registered in the project's `.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "cloudflare-landing@avi-ai": true
  }
}
```

This tells Claude Code to load the plugin's agents, skills, and commands when working in this project directory.

---

## The Deployed Architecture

```
                    ┌─────────────────────────────────┐
                    │         Cloudflare Edge          │
                    │                                  │
                    │  ┌──────────┐   ┌────────────┐  │
  postmoney.ai ──▶ │  │  Assets  │   │   Worker   │  │
                    │  │   CDN    │   │  (worker.js)│  │
                    │  └────┬─────┘   └─────┬──────┘  │
                    │       │               │          │
                    │  Static files    API routes      │
                    │  /, /pricing,   /api/signup,     │
                    │  /faq, etc.     /admin/*         │
                    │                      │           │
                    │               ┌──────┴──────┐    │
                    │               │  KV Store   │    │
                    │               │ SUBMISSIONS │    │
                    │               └─────────────┘    │
                    └─────────────────────────────────┘
```

**Static assets** (HTML, images) are served directly from Cloudflare's CDN — the Worker is never invoked for these requests. This is configured via `run_worker_first` in `wrangler.toml`, which only routes `/api/*` and `/admin/*` through the Worker.

**The Worker** handles three concerns:
- `POST /api/signup` — Parses the JSON body, adds metadata (IP, country, user agent, timestamp), stores in KV
- `GET /admin` — HTTP Basic Auth gate, then renders an HTML dashboard of all submissions
- `GET /admin/export.{json,csv}` — Downloads all submissions in the requested format

**KV Store** — Cloudflare's key-value store. Each submission is stored with the key format `signup:{timestamp}:{uuid}`, which ensures chronological ordering when listing keys by prefix.

---

## The Deployed Files

All artifacts live at `/tmp/postmoney-deploy/` (source repo untouched):

### `wrangler.toml`

```toml
name = "postmoney-site"
main = "worker.js"
compatibility_date = "2024-12-01"

assets = { directory = "./public", binding = "ASSETS" }

[[kv_namespaces]]
binding = "SUBMISSIONS"
id = "33425ccbde534c7fabf5b47efe40dfd4"

[[routes]]
pattern = "postmoney.ai"
custom_domain = true

[vars]
ADMIN_USER = "admin"
```

The KV namespace ID (`33425ccb...`) was obtained by the agent when it created the namespace. The custom domain route tells Cloudflare to serve this Worker when requests hit `postmoney.ai`.

### `worker.js`

A 224-line Cloudflare Worker with:
- Form submission handler that captures PostMoney-specific fields
- HTTP Basic Auth middleware
- KV-backed submission storage with paginated listing
- Admin dashboard with a styled HTML table
- JSON and CSV export endpoints
- HTML escaping for XSS prevention

### `public/`

A complete copy of the source site with one modification: `signup.html` has an injected `<script>` block that intercepts form submission and POSTs to `/api/signup`.

---

## Redeployment

To redeploy after making changes to the source site:

```
/cloudflare-landing:deploy
```

The agent will detect the existing deploy directory, copy updated files, and run `wrangler deploy`. Or manually:

```bash
cd /tmp/postmoney-deploy
cp -r /Users/avi/Development/code/Innovent/Sites/postmoney_website/* public/
wrangler deploy
```

---

## Verification Results

The agent verified every endpoint after deployment:

| Endpoint | Method | Expected | Result |
|----------|--------|----------|--------|
| `/` | GET | 200 | 200 (51,974 bytes) |
| `/pricing` | GET | 200 | 200 (48,704 bytes) |
| `/signup` | GET | 200 | 200 (43,037 bytes) |
| `/faq` | GET | 200 | 200 |
| `/images/screenshot-dashboard.png` | GET | 200 | 200 (197,320 bytes) |
| `/api/signup` | POST | 200 | 200 (submission stored) |
| `/admin` | GET (no auth) | 401 | 401 |
| `/admin` | GET (with auth) | 200 | 200 |
| `/admin/export.json` | GET | 200 | 200 |
| `/admin/export.csv` | GET | 200 | 200 |

---

## Why This Is Hard to Reproduce Manually

This deployment required coordinating across multiple systems in a specific sequence:

1. **Static site analysis** — Understanding file structure, detecting forms, extracting field names
2. **Code generation** — Writing a custom Cloudflare Worker tailored to the site's data model
3. **Infrastructure provisioning** — Creating a KV namespace via Cloudflare API
4. **Configuration** — Generating `wrangler.toml` with the correct namespace ID and domain
5. **Form wiring** — Injecting JavaScript that matches the Worker's API contract
6. **Deployment** — Uploading static assets + Worker code via Wrangler
7. **Secrets management** — Setting admin credentials as encrypted environment variables
8. **DNS configuration** — Routing the custom domain to the Worker
9. **Verification** — Testing every endpoint including auth flows

Each step depends on outputs from previous steps (KV namespace ID feeds into config, config feeds into deployment, deployment URL feeds into verification). The agent handled this entire dependency chain autonomously.

The skill encodes the *procedure*. The agent provides the *intelligence* to adapt it to any site. The MCP tools and Wrangler CLI provide the *infrastructure access*. Together, they turn "deploy this site" into a single command.
