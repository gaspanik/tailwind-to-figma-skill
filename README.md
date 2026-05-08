# tailwind-to-figma — Tailwind CSS v4 Tokens → Figma Variables Skill

A Claude Code skill that reads all `@theme` tokens from `src/index.css` and writes them into a Figma file as Variables — organized into appropriate collections by type, in one command.

---

## What this is

Design tokens defined in Tailwind CSS v4 as `@theme` custom properties are useless in Figma until someone manually re-enters them. That sync is tedious and error-prone, especially as the token set grows.

This skill automates the reverse pipeline: it reads `src/index.css`, parses every CSS custom property in the `@theme` block, converts them to Figma-compatible values, and writes them as Variables into the correct collection — Colors, Typography, Spacing, Radius, Border, or Other.

```
src/index.css  →  tailwind-to-figma  →  Figma Variables (organized by collection)
                                     →  Completion report (collections / variables created)
```

---

## What gets converted

| CSS variable prefix | Category | Figma collection | Figma variable type |
|---|---|---|---|
| `--color-*` | Color | `Colors` | `COLOR` |
| `--*-color-*` | Color | `Colors` | `COLOR` |
| `--font-family-*` / `--default-font-family` / `--heading-font-family` | Font family | `Typography` | `STRING` |
| `--font-*` (string value) | Font family | `Typography` | `STRING` |
| `--text-*` (numeric/rem/px) | Font size | `Typography` | `FLOAT` |
| `--font-weight-*` | Font weight | `Typography` | `FLOAT` |
| `--spacing-*` | Spacing | `Spacing` | `FLOAT` |
| `--radius-*` | Border radius | `Radius` | `FLOAT` |
| `--border-*` (numeric) | Border width | `Border` | `FLOAT` |
| Other `--*` | Other | `Other` | Determined by value |

- CSS prefix is stripped and the remainder becomes the Figma variable name (`--color-brand-500` → `brand/500` in the Colors collection)
- `rem` values are converted to px (× 16); `px` suffix is stripped
- Hex and `rgba()` colors are converted to Figma's 0–1 float format
- Font family fallbacks are stripped — only the primary font name is kept
- Values that cannot be converted (`var()` references, gradients, etc.) are skipped with a warning

---

## Repo structure

```
skills/
  tailwind-to-figma/
    SKILL.md          — skill definition loaded by Claude Code
```

---

## Getting started

**1. Clone this repo**

```bash
git clone https://github.com/gaspanik/tailwind-to-figma-skill
```

**2. Install the skill into Claude Code**

Copy the skill directory into your Claude Code skills folder:

```bash
cp -r skills/tailwind-to-figma ~/.claude/skills/
```

**3. Verify the Figma MCP is connected**

This skill requires the official [Figma MCP server](https://github.com/figma/mcp-server-guide). Confirm it is connected in your Claude Code settings before running the skill.

**4. Run the skill**

Run from your project root (where `src/index.css` lives). Invoke with a slash command or natural language — both work:

```
/tailwind-to-figma
```

```
/tailwind-to-figma My Design Tokens
```

```
/tailwind-to-figma https://www.figma.com/design/<fileKey>/...
```

```
Export Tailwind tokens to Figma
```

```
Tailwindのトークンを Figmaに書き出して
```

The skill responds in the same language you use — no configuration needed.

**Argument options:**

| Argument | Behavior |
|---|---|
| *(omitted)* | Creates a new Figma file named "Design Tokens" |
| File name (e.g. `My Tokens`) | Creates a new Figma file with that name |
| Figma URL | Appends/updates Variables in the existing file |

You will receive a report in this format:

```
✅ Written to Figma file "Design Tokens" (fileKey: abc123)

  Colors      — 20 variables
  Typography  — 8 variables
  Spacing     — 14 variables
  Radius      — 6 variables

  Skipped: Border, Other (no tokens)
  Skipped values (cannot convert): --background: var(--color-base)
```

---

## Agent settings

- Use the most capable Claude model available. The skill parses CSS, converts values across multiple types, and writes structured data to the Figma Plugin API — a stronger model handles edge cases (unusual variable names, complex color formats) more reliably.
- The `SKILL.md` format and `allowed-tools` frontmatter are Claude Code-specific. To use this with another agent (Cursor, Windsurf, etc.), copy the prompt body from `SKILL.md` into that agent's rule format — the conversion logic itself is plain markdown and transfers without modification.

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `src/index.css not found` | Run the skill from your project root, or confirm the file exists at `src/index.css`. |
| `@theme block is empty` | Add CSS custom properties inside `@theme { }` in `src/index.css` and re-run. |
| `create_new_file failed` | Check that you are logged into Figma. Run `whoami` via the Figma MCP to verify your session. |
| `use_figma returned an error` | Check that your Figma MCP is connected and has write access to the target file. |
| Variables appear in the wrong collection | The classification is based on variable name prefixes. Rename variables in `src/index.css` to match the expected prefix (e.g. `--color-*` for colors). |
| Font family fallbacks showing up | The skill strips everything after the first comma. If this causes issues, define font variables without fallbacks in `@theme`. |

---

## Related

- [figma-to-tailwind-skill](https://github.com/gaspanik/figma-to-tailwind-skill) — the reverse direction: imports Figma Variables as Tailwind CSS v4 `@theme` tokens

---

Built by Masaaki Komori - [@cipher](https://x.com/cipher) · Skill for [Claude Code](https://claude.ai/code) + [Figma MCP](https://github.com/figma/mcp-server-guide)
