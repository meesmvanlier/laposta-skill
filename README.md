# Laposta skill

An [Agent Skill](https://agentskills.io) that lets an AI assistant work with
[Laposta](https://www.laposta.nl), the Dutch email marketing platform, through its REST
API: lists, subscribers, custom fields, segments, campaigns, results and webhooks.

The skill knows the **whole** API. Its endpoint reference is generated from the same
source that produces Laposta's published API documentation, so it never drifts: when the
API changes, you regenerate it.

## What it can do

Ask in plain language and the assistant works out the calls:

- "Add these twelve people to my Laposta newsletter list, first names in the Voornaam field."
- "Which subscribers unsubscribed from the customer list last month?"
- "Create a campaign from this html, send me a test, and tell me how many people it would reach."
- "How did the August newsletter do?"
- "Set up a webhook so my shop hears about new subscribers."
- "What does the field endpoint return exactly?"

It refuses to guess. Deleting anything, scheduling and sending all wait for your explicit
confirmation, and sending is preceded by a test mail.

## Install

You need a Laposta API key. In Laposta, go to **Koppelingen** and create one. It is shown
only once. A free account can hold three keys.

```bash
export LAPOSTA_API_KEY="your-key"
```

### Claude Code

This repository is also a plugin marketplace, so two commands are enough:

```
/plugin marketplace add meesmvanlier/laposta-skill
/plugin install laposta@laposta-skill
```

The install command asks where you want it: for yourself, for one project, or locally.
Later on, `/plugin marketplace update laposta-skill` pulls in changes.

`/plugin` exists only in Claude Code in the terminal. In the desktop app and in the
browser it reports that the command is not available there, so use the manual route
below.

Prefer to copy the files yourself:

```bash
git clone https://github.com/meesmvanlier/laposta-skill.git
mkdir -p ~/.claude/skills
cp -r laposta-skill/plugins/laposta/skills/laposta ~/.claude/skills/
```

Use `.claude/skills/` inside a project instead if you only want it there. Claude Code
picks it up without a restart.

### claude.ai

Zip the skill folder so that the folder itself is the root of the archive, then upload it
under Skills in your settings.

```bash
cd laposta-skill/plugins/laposta/skills && zip -r laposta.zip laposta
```

There is no marketplace on claude.ai; the zip is the way in there.

Network access has to be on for the assistant to reach `api.laposta.nl`. On Free, Pro and
Max that is the default; on Team and Enterprise an administrator controls it.

Skills do not work over the Claude API for this purpose: that sandbox has no network
access, so it cannot call Laposta at all.

### Other assistants

`SKILL.md` sticks to the six frontmatter fields that are valid everywhere, so the same
folder works in tools that support the Agent Skills standard, among them Gemini CLI
(`~/.gemini/skills/`), GitHub Copilot (`.github/skills/` or `.claude/skills/`), Codex and
Cursor. Check your tool's own documentation for where skills live.

## What is in it

```
.claude-plugin/marketplace.json     makes this repository installable in one command
plugins/laposta/
├── .claude-plugin/plugin.json      the plugin manifest
└── skills/laposta/
    ├── SKILL.md                    conventions, safety rules, index of all 41 operations
    └── references/
        ├── endpoints.md            every operation with its parameters      (generated)
        ├── schemas.md              every response object, field by field    (generated)
        ├── tasks.md                the common jobs as end to end flows
        ├── pitfalls.md             the traps, each one verified against a live account
        └── errors-and-limits.md    status codes, error codes, rate limits
spec/                               the API source the reference is built from
scripts/generate-reference.py       regenerates the two generated files
```

The skill folder is a plain Agent Skill. The plugin wrapper around it only exists to make
installing a one liner; copying that folder by hand works just as well.

## Keeping it current

The two generated files come from `spec/`. When the API changes, refresh the spec and run:

```bash
python3 scripts/generate-reference.py
```

It rewrites `endpoints.md` and `schemas.md` and updates the operation index inside
`SKILL.md`, and leaves every hand written line alone. In CI:

```bash
python3 scripts/generate-reference.py --check
```

Requires Python 3 and PyYAML.

## Safety

The skill is built so that an assistant with your API key cannot quietly do damage:

- The key is read from the environment or asked for. It is never written to a file, a
  commit or a log, and never printed back to you.
- Deleting, emptying, scheduling and sending all require you to say yes first, one action
  at a time.
- `action/send` cannot be undone, so a test mail comes first.
- Unsubscribing and deleting are treated as different things, because they are.

A Laposta API key is the account. Anyone holding it can read your whole subscriber base
and send in your name. Use a separate key for this, and revoke it in the app when you are
done.

## What the API cannot do

- **Labels** can be read and set, but not created: only labels the list already has are
  accepted. They are also absent from Laposta's published API documentation.
- Campaigns built in the drag and drop editor cannot have their content read or written.
- Bulk sync is off unless the account is paid and has it enabled.
- Segment definitions cannot reliably be written from scratch; build them in the app and
  read them back.
- There is no sandbox. Every call hits the real account behind the key.

## Licence

MIT. Laposta is a product of Laposta B.V.; this skill is not an official Laposta release.
