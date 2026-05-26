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

### Figma MCP tool not found / `use_figma` fails to resolve

The Figma MCP server can be connected in two ways, and the internal tool name Claude Code uses differs between them. If the skill cannot find the tool, this is the most likely cause.

#### Installation method A — Plugin (Claude.ai / Claude Code desktop)

Install the Figma MCP via the Claude Code plugin marketplace or by adding it from the Claude.ai integrations panel. When installed this way, Claude Code registers the tools under a plugin-namespaced prefix:

```
mcp__plugin_figma_figma__use_figma
mcp__plugin_figma_figma__get_design_context
...
```

The skill's `SKILL.md` refers to `use_figma` without a namespace, and Claude Code resolves it automatically to the plugin-prefixed version.

#### Installation method B — Direct settings configuration (settings.json)

Add the Figma MCP server manually to your Claude Code settings file:

```json
// .claude/settings.json  (project-level)
// or ~/.claude/settings.json  (user-level)
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "@figma/mcp-server"]
    }
  }
}
```

With this approach the tools are registered under a shorter prefix:

```
mcp__figma__use_figma
mcp__figma__get_design_context
...
```

#### If the skill still fails to call `use_figma`

1. Open Claude Code and type `/mcp` or check **Settings → MCP** to confirm the Figma server appears in the connected-servers list.
2. Ask Claude directly: *"What Figma MCP tools are available?"* — the response will show the exact prefixed names that are active.
3. If both a plugin and a settings entry exist at the same time, two sets of tools will be registered and may conflict. Remove one of them.
4. Restart Claude Code after any settings change.

---

### Figma MCPツールが見つからない / `use_figma` が解決されない場合（日本語）

Figma MCPサーバーの接続方法は2通りあり、Claude Codeが内部で使うツール名がそれぞれ異なります。スキルがツールを見つけられない場合、これが最も多い原因です。

#### 方法A — プラグイン（Claude.ai / Claude Codeデスクトップ）

Claude Codeのプラグインマーケットプレイスや、Claude.aiのインテグレーションパネルからFigma MCPをインストールした場合、ツール名はプラグイン用のプレフィックスで登録されます。

```
mcp__plugin_figma_figma__use_figma
mcp__plugin_figma_figma__get_design_context
```

スキルの `SKILL.md` では `use_figma` とだけ書かれていますが、Claude Codeがプラグイン用のプレフィックス付き名前に自動解決します。

#### 方法B — settings.json への直接記述

Claude Codeの設定ファイルにFigma MCPサーバーを手動で追加した場合：

```json
// .claude/settings.json（プロジェクト）または ~/.claude/settings.json（ユーザー）
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "@figma/mcp-server"]
    }
  }
}
```

この場合、ツール名は短いプレフィックスで登録されます。

```
mcp__figma__use_figma
mcp__figma__get_design_context
```

#### それでも `use_figma` の呼び出しに失敗する場合

1. Claude Codeで `/mcp` と入力するか、**Settings → MCP** を開き、Figmaサーバーが「接続済み」の一覧に表示されていることを確認する。
2. Claudeに「使えるFigma MCPツールは何ですか？」と聞くと、現在アクティブなプレフィックス付きのツール名一覧が返ってきます。
3. プラグインと settings.json の両方に同時に設定が存在すると、2セットのツールが登録されて競合することがあります。どちらか一方を削除してください。
4. 設定を変更したら、必ずClaude Codeを再起動してください。

---

## Related

- [figma-to-tailwind-skill](https://github.com/gaspanik/figma-to-tailwind-skill) — the reverse direction: imports Figma Variables as Tailwind CSS v4 `@theme` tokens

---

Built by Masaaki Komori - [@cipher](https://x.com/cipher) · Skill for [Claude Code](https://claude.ai/code) + [Figma MCP](https://github.com/figma/mcp-server-guide)
