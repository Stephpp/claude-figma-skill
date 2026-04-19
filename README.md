# claude-figma-skill

A [Claude Skill](https://docs.claude.com/en/docs/claude-code/skills) that pulls design data straight from the **Figma REST API** using a personal access token. Works with any Figma plan — including free — and doesn't require the Figma desktop app, a Dev Mode seat, or a running MCP server.

Paste a Figma URL into Claude Code and it will fetch the file, read the node tree, optionally grab a PNG render, and implement the design in your codebase.

## Disclaimers

**Why this exists.** I built this because I was tired of hitting the extremely low usage limits on the Figma MCP. It's a workaround, not a replacement — if you have a paid Figma plan and a Dev seat, the official MCP will give you nicer, more semantic design-to-code output than raw REST JSON.

**You don't actually need this repo.** You can get the same result in five minutes by asking Claude to create a skill for you:
```text
Create a Claude skill that fetches Figma design data through the REST API using a personal access token instead of Figma's MCP server.
```

<details>
<summary>Or for a more detailed prompt that will give you better results:</summary>

```text
Create a Claude Agent Skill that fetches Figma design data through 
the Figma REST API using a personal access token, as a replacement 
for the Figma MCP server (which has tight rate limits on free plans).

The skill should:

1. **Auto-trigger** when the user pastes a Figma URL (figma.com/design/, 
   figma.com/file/, figma.com/proto/) or mentions implementing a Figma 
   design. Write the YAML frontmatter description with strong trigger 
   keywords so auto-invocation is reliable.

2. **Read the token from the $FIGMA_TOKEN environment variable.** If it's 
   missing, stop and give the user clear instructions to add it to 
   ~/.claude/settings.json under an "env" block.

3. **Normalize node IDs** from all three common URL formats before 
   calling the API: `1-2` (dashes), `1:2` (colons), and `1%3A2` 
   (URL-encoded). The Figma API requires the colon form.

4. **Handle the branch URL case** where figma.com/design/:fileKey/branch/:branchKey/... 
   uses the branchKey as the effective fileKey in API calls.

5. **Limit response size.** Use `?depth=4` on node fetches and `?depth=1` 
   on full-file overview fetches to avoid blowing out context on large 
   design files.

6. **Fetch a PNG render alongside the JSON** (via /v1/images) for visual 
   reference, and have Claude actually read the returned image URL 
   rather than just describing it.

7. **Handle HTTP errors with actionable messages**:
   - 403 → token expired or missing scopes (file_content:read, 
     file_metadata:read)
   - 404 → wrong file key, wrong node ID, or no access
   - 429 → rate limited, retry after a few seconds
   Don't dump raw JSON errors to the user.

8. **Note the Enterprise-only endpoints.** /v1/files/{fileKey}/variables/local 
   returns 403 on all plans except Enterprise — mention this in the 
   skill so users aren't confused when it fails.

9. **Guide the implementation step.** After fetching, point Claude at 
   the fields that matter for design-to-code: layoutMode → flexbox, 
   itemSpacing/padding, fills/strokes/cornerRadius, style (typography), 
   and componentId for INSTANCE nodes. Tell Claude to prefer auto-layout 
   mappings over absolute positioning.

10. **Gracefully fall back when jq is not installed** — check once at 
    the start, don't run dual curl commands.

Reference the relevant endpoints:
- GET /v1/files/{fileKey} (full file, use depth param)
- GET /v1/files/{fileKey}/nodes?ids={nodeId}
- GET /v1/images/{fileKey}?ids={nodeId}&format=png&scale=2
- GET /v1/files/{fileKey}/styles
- GET /v1/files/{fileKey}/variables/local (Enterprise only)
- GET /v1/files/{fileKey}/meta

All require the X-Figma-Token header.

Structure it as a proper Agent Skill: YAML frontmatter with `name` and 
`description` fields, then numbered markdown steps Claude follows at 
runtime.
```
</details>


Claude will scaffold it. If you're more interested in **how Skills work** than in just getting Figma access, I'd recommend that route — you'll learn the skill structure, the frontmatter, and the trigger system by building it yourself. This repo is for people who just want a working one they can drop in.

## How this works

**What it reads.** The Figma REST API returns JSON — a tree of nodes that describes layout, typography, colors, spacing, and component structure. It does not return pixel data directly. Claude reads this JSON to understand what to build.

**How images work.** To visually reference a frame, the skill requests a rendered PNG from Figma's image export endpoint. This returns a signed S3 URL pointing to the render. Because Claude's `Read` tool only works on local file paths — not remote URLs — the image is first downloaded to a local temp directory, then read from there.

The temp directory used depends on your OS:

| OS | Path |
|---|---|
| macOS | `/tmp/figma-images/` |
| Linux | `/tmp/figma-images/` |
| Windows | `$env:TEMP\figma-images\` (e.g. `C:\Users\you\AppData\Local\Temp\figma-images\`) |

These locations are ephemeral — cleared on reboot or under memory pressure. Nothing is stored permanently.

**Why both?** JSON gives Claude the structure it needs to write code (layout mode, spacing, colors, typography). The rendered image gives it visual ground truth to self-correct against. The two together produce much better output than either alone.

## Why this instead of the Figma MCP?

The official Figma MCPs are great, but they have access constraints:

- The **Dev Mode MCP** (desktop-integrated) needs the Figma desktop app running with the file open
- The **Remote Dev Mode MCP** needs a Dev or Full seat on a paid Figma plan — not available on free accounts

A personal access token works on any plan, in any editor, with no desktop app. The tradeoff is that raw REST responses are more verbose than MCP output — this skill handles that by limiting fetch depth and pointing Claude at the fields that matter.

## What the skill struggles with

This skill gets you ~70% of the way there on a first pass. Knowing where it falls short upfront will save you frustration later.

- **Component variants and states.** If your design uses a Button component with 12 variants (hover, disabled, loading, etc.), the API tells Claude which variant is *currently placed on the canvas* — not the full variant system. Claude will usually get the visible state right and miss the others. Mention other states explicitly if you need them.

- **Interactions and animations.** Prototype links, hover transitions, and animation specs exist in Figma but won't come through cleanly via REST. Treat the skill as "static design → static code." Anything interactive you'll describe manually.

- **Nested component instances.** When a frame uses a Card that uses a Button that uses an Icon, Claude sometimes flattens the hierarchy into one big chunk of markup instead of recognizing the reusable structure. Prompt explicitly: *"Identify reusable components before implementing."*

- **Design tokens.** Unless you're on Figma Enterprise (where `/variables/local` works), the skill can't read your design token system directly. Claude will hardcode color values like `#3B82F6` instead of referencing `var(--color-primary)`. You can work around this by pointing Claude at your existing tokens file and telling it not to hardcode values.

- **Pixel-perfect layouts.** Raw REST output is noisier than MCP output, so Claude occasionally miscalculates spacing by a few pixels or misreads alignment intent. The PNG reference helps it self-correct, but you'll still be polishing manually.

- **Icons and custom vector art.** The API returns vector path data, but Claude won't reliably reconstruct complex SVGs. Export icons as SVG via the `/v1/images?format=svg` endpoint, or swap them for equivalents from lucide/heroicons/etc.

## Tips for better results

A few habits that make the difference between "meh, this looks wrong" and "okay, this is actually usable."

- **Feed one frame at a time.** Don't paste a top-level file URL and say "build everything." Pick a specific frame (click it in Figma → copy URL with the `node-id`), implement it, iterate, then move to the next. Smaller scopes = better output and smaller context usage.

- **Give Claude your stack upfront.** Before pasting the Figma URL, tell Claude what you're working with. This one line saves you a lot of rework:

   > *"This is a Next.js + Tailwind project. Use Tailwind classes, match the existing components in `src/components`, and reference the tokens in `tailwind.config.js` — don't hardcode colors."*

- **Use plan mode for anything non-trivial.** In Claude Code, press Shift+Tab to enter plan mode, then paste your Figma link. Claude will propose a plan — which components it'll extract, what files it'll create, how it'll structure the output — before writing any code. Review the plan, adjust, then approve. Much faster than building and then refactoring.

- **Set up your foundation before using the skill.** If your project is brand new, create your design tokens, base typography, and core reusable components (Button, Input, Card) *yourself* first. Then point Claude at the Figma design and say *"use my existing Button component from `src/components/Button.tsx`, don't create a new one."* Claude is much better at composing existing pieces than inventing a component system from scratch.

- **Ask for a component plan first on complex screens.**

   > *"Before coding, list the reusable components you'd extract from this design and how they nest."*

   This forces Claude to think in terms of reusable pieces instead of flattening everything into one giant component.

- **Pair with the PNG reference.** The skill already fetches a rendered PNG — make sure Claude actually looks at it. If the output drifts from the design, tell Claude: *"compare your output to the rendered PNG and fix discrepancies."*

- **Expect to iterate.** First pass is ~70% there. Second pass after targeted feedback ("the padding is off," "this should be a grid not a stack") gets you to ~90%. The last 10% is you, manually, as always.

## Requirements

- [Claude Code](https://claude.com/product/claude-code)
- A Figma personal access token (free to generate on any plan)
- Optional: `jq` for prettier JSON output — the skill works without it

## Step 1 — Generate a Figma token

1. Open [figma.com](https://figma.com) and log in
2. Account menu (top-left) → **Settings**
3. **Security** tab → scroll to **Personal access tokens** → **Generate new token**
4. Name it something memorable (e.g. `claude-skill`)
5. Enable these scopes:
   - `file_content:read`
   - `file_metadata:read`
6. Leave all other scopes off
7. Click **Generate token** and copy it immediately — Figma won't show it again

> ⚠️ **This token grants read access to every Figma file your account can see**, including team and organization files. Treat it like a password.

## Step 2 — Install the skill

### Option A — Clone (recommended)

```bash
git clone https://github.com/Stephpp/claude-figma-skill.git
```

Then copy the skill folder into your Claude skills directory:

**macOS / Linux:**
```bash
mkdir -p ~/.claude/skills
cp -r claude-figma-skill/figma ~/.claude/skills/figma
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills"
Copy-Item -Recurse claude-figma-skill\figma "$env:USERPROFILE\.claude\skills\figma"
```

### Option B — Terminal one-liner

Skip the clone and pull just the skill file:

**macOS / Linux:**
```bash
mkdir -p ~/.claude/skills/figma
curl -o ~/.claude/skills/figma/SKILL.md \
  https://raw.githubusercontent.com/Stephpp/claude-figma-skill/master/figma/SKILL.md
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\figma"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/Stephpp/claude-figma-skill/master/figma/SKILL.md" `
  -OutFile "$env:USERPROFILE\.claude\skills\figma\SKILL.md"
```
### Option C — VS Code (Claude Code extension)

Same as Option A or B — the skill lives in your home directory and the extension picks it up automatically.

## Step 3 — Add your token

Claude Code often doesn't inherit your shell's environment, so the most reliable place to set the token is Claude's own settings file.

| OS | Settings path |
|---|---|
| macOS | `~/.claude/settings.json` |
| Linux | `~/.claude/settings.json` |
| Windows | `%USERPROFILE%\.claude\settings.json` |

Add an `env` block:

```json
{
  "env": {
    "FIGMA_TOKEN": "your_token_here"
  }
}
```

Merging with existing settings:

```json
{
  "model": "sonnet",
  "env": {
    "FIGMA_TOKEN": "your_token_here"
  }
}
```

Save, then **fully restart Claude Code** (or VS Code if you're using the extension).


### Verify

In any Claude Code session:

```
/figma check
```

Or:

```bash
echo $FIGMA_TOKEN
```

## Step 4 — Use it

### Auto-triggered (just paste a link)

```
Implement this component: https://www.figma.com/design/abc123/MyApp?node-id=12-34
```

```
Here's the login screen — build it in React:
https://www.figma.com/design/abc123/MyApp?node-id=56-78
```

Claude detects the URL and fetches the design automatically.

### Slash command

```
/figma https://www.figma.com/design/abc123/MyApp?node-id=12-34
```

### Full-file overview (no node ID)

```
/figma https://www.figma.com/design/abc123/MyApp
```

Claude returns a list of pages and top-level frames so you can pick one.

### Export a frame as an image

```
Export this frame as a PNG: https://www.figma.com/design/abc123/MyApp?node-id=12-34
```

The skill will fetch a rendered image URL via the `/v1/images` endpoint.

## Finding a node ID

1. Open the file in Figma
2. Click any frame or component
3. The URL updates to include `?node-id=XX-XX`
4. Copy the full URL

The skill handles all three common node-id formats: `1-2`, `1:2`, and `1%3A2`.

## Supported endpoints

| Goal | Endpoint |
|---|---|
| Full file structure | `GET /v1/files/{fileKey}` |
| Specific node(s) | `GET /v1/files/{fileKey}/nodes?ids={nodeId}` |
| Rendered image | `GET /v1/images/{fileKey}?ids={nodeId}&format=png` |
| File styles | `GET /v1/files/{fileKey}/styles` |
| Design variables | `GET /v1/files/{fileKey}/variables/local` *(Enterprise plan only)* |
| Comments | `GET /v1/files/{fileKey}/comments` |
| File metadata | `GET /v1/files/{fileKey}/meta` |

## Troubleshooting

**"FIGMA_TOKEN is not set"**
The token isn't reaching Claude's process. Confirm it's in `settings.json` (not only in your shell) and that you fully restarted Claude Code after saving.

**403 Forbidden**
Either the token is expired or it's missing scopes. Regenerate it with `file_content:read` and `file_metadata:read`. Also note: `/variables/local` returns 403 on non-Enterprise plans — that's expected.

**404 Not Found**
Usually the node ID format. Figma URLs use dashes (`12-34`) but the API needs colons (`12:34`) — the skill normalizes this, but manual input might not be. Also verify you actually have access to the file.

**"jq: command not found"**
Optional; the skill falls back to raw JSON. To install:
- macOS: `brew install jq`
- Linux: `sudo apt install jq`
- Windows: [jqlang.github.io/jq/download](https://jqlang.github.io/jq/download/)

**Huge response / context runs out**
On complex files, fetch a specific node rather than the whole file. The skill uses `depth=4` by default to limit nested children — if you need more depth for a specific component, ask explicitly.

**Auto-trigger isn't firing**
Make sure the skill lives at `~/.claude/skills/figma/SKILL.md` — the `commands/` directory only enables the slash command, not auto-trigger.

## Updating

If you cloned the repo, pulling the latest version is two commands:

```bash
cd claude-figma-skill
git pull
cp -r figma ~/.claude/skills/figma
```

On Windows (PowerShell):

```powershell
cd claude-figma-skill
git pull
Copy-Item -Recurse -Force figma "$env:USERPROFILE\.claude\skills\figma"
```

Restart Claude Code to pick up the changes.

If you used the one-liner install, re-run it to overwrite your local `SKILL.md` with the latest version

## Keep your token safe

- Never commit it to git (the repo's `.gitignore` excludes `settings.json` and `.env`)
- Don't share it — it reads every file your Figma account can see
- Revoke anytime in Figma → Settings → Security
- Rotate periodically if used long-term

## License

MIT