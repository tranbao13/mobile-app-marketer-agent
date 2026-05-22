# Mobile App Marketer Agent

An AI agent for mobile app marketers. Ask questions in plain English,
get professional mobile advertisement analysis powered by your own MMP data — no
dashboards, no SQL, no data team needed.

Built on [Model Context Protocol (MCP)](https://modelcontextprotocol.io).
Runs on [claude.ai](https://claude.ai) — no installation required.

---

## What it does

**Account health check** — one prompt gives you a full summary of every
app in your Adjust account: active vs inactive, revenue anomalies, and
engagement trends.

> *"Do an initial check and summarize my MMP account's status"*

**App snapshot report** — ask for a dashboard on any app and get an
interactive report with install trends, revenue breakdown, DAU charts,
and data-grounded insights.

> *"Show me a snapshot report of [App Name] in the last 3 months"*

**Profile-aware analysis** — tell the agent what kind of company you
are (gaming, finance, etc.) and how your apps make money. It uses that
context to flag real anomalies — not false positives.

---

## Setup — 3 steps, under 5 minutes

### Step 1 — Connect Adjust to claude.ai

1. Go to [claude.ai](https://claude.ai) → **Settings** → **Integrations**
2. Find **Adjust** and click **Connect**
3. Authenticate with your Adjust account credentials

> Requires a claude.ai Pro or Team plan.

---

### Step 2 — Create a Project and paste the system prompt

1. In claude.ai, click **New Project**
2. Click **Set project instructions**
3. Download the system prompt below and paste it in full

**[⬇ Download system prompt](https://github.com/tranbao13/mobile-app-marketer-agent/releases/latest)**

The download includes:
- `system-prompt.md` — paste this into Project instructions
- `profile.yaml` — your company profile template (see Step 3)

---

### Step 3 — Fill in your company profile

Open `profile.yaml` and fill in 3 fields:

```yaml
account:
  name: "Your Company Name"

business:
  verticals:
    - gaming         # what type of apps you build

  monetization:
    - IAA            # how your apps make money

  notes: ""          # anything the agent should know
                     # e.g. "One finance app — zero MMP revenue is expected"
```

Then open `system-prompt.md`, find the line that says `{{PROFILE_CONTENT}}`,
and replace it with the contents of your filled `profile.yaml`.

Paste the full result into your Project instructions. Done.

---

## Test it

Try these prompts in your new Project:

```
Do an initial check and summarize my MMP account's status
```

```
Show me a snapshot report of [Your App Name] in the last 3 months
```

---

## Prompts reference

| What you want | What to type |
|---|---|
| Full account health check | "Do an initial check" |
| App dashboard (90 days) | "Show me a snapshot of [App Name]" |
| App dashboard (custom period) | "Snapshot of [App Name] last 30 days" |
| Update your profile | "Update my profile — [describe change]" |

---

## File structure

```
/
├── system-prompt.md        ← Paste this into claude.ai Project instructions
├── profile.yaml            ← Fill this in, then paste into system-prompt.md
├── skills/
│   └── mmp-reporting.md   ← Included in system-prompt.md automatically
├── .env.example            ← For Claude Desktop users only (advanced)
├── config/
│   └── claude-desktop-example.json  ← For Claude Desktop users only (advanced)
└── README.md
```

---

## Updating your profile later

You don't need to edit files manually. Just prompt the agent:

> *"Update my profile — add a new app, it's a midcore iOS game with IAP monetization"*

The agent will output the updated YAML. Copy it into your `profile.yaml`,
rebuild the system prompt, and update your Project instructions.

---

## Roadmap

- [x] Phase 1 — MMP reporting (account health check + app snapshot)
- [ ] Phase 2 — Google Ads integration (cross-platform ROAS bridge)
- [ ] Phase 3 — Anomaly detection (automated flags)
- [ ] Phase 4 — Campaign optimization recommendations

---

## Contributing

PRs welcome. If you add a skill, follow the structure in
`skills/mmp-reporting.md` — trigger phrases, step-by-step logic,
and output format clearly defined.

---

## Claude Desktop (advanced)

Prefer a local setup? See the [Claude Desktop setup guide](config/claude-desktop-example.json).
Requires Node.js and manual config file editing.

---

## License

MIT — fork freely, use commercially, attribution appreciated.
