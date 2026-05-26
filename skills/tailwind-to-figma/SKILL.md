---
name: tailwind-to-figma
description: Finds the Tailwind CSS v4 entry file by searching for @import "tailwindcss", reads @theme tokens, and writes them to a Figma file as Variables organized into appropriate collections by type.
argument-hint: '(optional: Figma file URL → append to existing file / file name only → create new file with that name / omit → create new file named "Design Tokens")'
allowed-tools: mcp__plugin_figma_figma__use_figma, mcp__plugin_figma_figma__create_new_file, mcp__plugin_figma_figma__whoami, Read, Bash, Skill
---

# Export Tailwind v4 Tokens as Figma Variables

Find the Tailwind CSS v4 entry file, read all CSS custom properties from the `@theme` block, and write them as Figma Variables.

**Output language:** Respond in the same language the user is using in this conversation.

## Step 0: Load the figma-use skill

Before calling `use_figma`, always load the `figma-use` skill first:

```
/figma:figma-use
```

## Step 1: Find and parse the Tailwind CSS entry file

First, use the `Bash` tool to locate the CSS file that imports Tailwind:

```bash
grep -rl --include="*.css" --exclude-dir=node_modules --exclude-dir=dist '@import ["'"'"']tailwindcss' . 2>/dev/null
```

- If **one file** is found → use it
- If **multiple files** are found → ask the user which one to use
- If **no file** is found → ask the user to provide the CSS file path

Then use the `Read` tool to load that file and extract all CSS custom properties inside the `@theme { ... }` block.

### Variable prefixes and their classification

| CSS variable prefix | Category | Figma collection | Figma variable type |
|---|---|---|---|
| `--color-*` | Color | `Colors` | `COLOR` |
| `--*-color-*` | Color | `Colors` | `COLOR` |
| `--*-font-family` / `--font-family-*` / `--default-font-family` / `--heading-font-family` | Font family | `Typography` | `STRING` |
| `--font-*` (string value) | Font family | `Typography` | `STRING` |
| `--text-*` (numeric/rem/px) | Font size | `Typography` | `FLOAT` |
| `--font-size-*` | Font size | `Typography` | `FLOAT` |
| `--font-weight-*` | Font weight | `Typography` | `FLOAT` |
| `--spacing-*` | Spacing | `Spacing` | `FLOAT` |
| `--radius-*` | Border radius | `Radius` | `FLOAT` |
| `--border-*` (numeric) | Border width | `Border` | `FLOAT` |
| Other `--*` | Other | `Other` | Determined by value |

### Name conversion rules (CSS → Figma slash notation)

Figma variable names use slash-separated path format:

- `--color-primary` → `primary` (inside Colors collection)
- `--color-brand-500` → `brand/500`
- `--default-font-family` → `default` (inside Typography collection)
- `--heading-font-family` → `heading`
- `--font-sans` → `sans`
- `--text-base` → `size/base` (inside Typography collection)
- `--spacing-4` → `4` (inside Spacing collection)
- `--radius-md` → `md` (inside Radius collection)

Strip the prefix (`--color-`, `--spacing-`, etc.) and use the remainder as the variable name.

## Step 2: Convert CSS values to Figma values

### COLOR conversion

- `#rrggbb` → `{ r: parseInt(rr,16)/255, g: parseInt(gg,16)/255, b: parseInt(bb,16)/255, a: 1 }`
- `#rrggbbaa` → converted with alpha
- `rgba(r, g, b, a)` → `{ r: r/255, g: g/255, b: b/255, a: a }`
- `rgb(r, g, b)` → `{ r: r/255, g: g/255, b: b/255, a: 1 }`

### STRING conversion (font families)

- `"Font Name", fallback` → `"Font Name"` (extract only the first quoted font name)
- `FontName, fallback` → `"FontName"`
- Remove quotes and everything after the first comma (fallback fonts)

### FLOAT conversion (numeric values)

- `1rem` → `16` (× 16 for px equivalent)
- `0.5rem` → `8`
- `16px` → `16` (strip px)
- `0` → `0`
- `1px` → `1`
- Plain numbers pass through as-is

## Step 3: Prepare the Figma file

Branch based on `$ARGUMENTS`:

### Pattern A: A Figma URL is provided (append to existing file)

Extract `fileKey` from the URL. No need to call `whoami` or `create_new_file`.

- `figma.com/design/:fileKey/...` → take `:fileKey`
- The `?node-id=` query parameter can be ignored

Use the extracted `fileKey` in subsequent `use_figma` calls.

### Pattern B: A file name is provided, or no argument (create a new file)

First call `whoami` to get the plan list:

- **1 plan** → use that `key` as `planKey`
- **Multiple plans** → ask the user which team/organization to create the file in before proceeding

Create a new file with `create_new_file`:

```
create_new_file({
  fileName: "<argument name or "Design Tokens">",
  planKey: "<planKey from whoami>",
  editorType: "design"
})
```

Use the returned `fileKey` in subsequent `use_figma` calls.

## Step 4: Create Figma Variables

Use `use_figma` with the Plugin API to create collections and variables in bulk.

**Skip any collection that has no variables** — only create collections for categories that have at least one token.

Pass the following code to `use_figma` (fill `tokenData` with the actual values extracted in the previous steps):

```javascript
// ===== Actual token data (fill with values from Steps 1–2) =====
const tokenData = {
  Colors: [
    // e.g. { name: "primary", type: "COLOR", value: { r: 0.173, g: 0.094, b: 0.063, a: 1 } }
  ],
  Typography: [
    // e.g. { name: "default", type: "STRING", value: "Gen Interface JP" }
    // e.g. { name: "size/base", type: "FLOAT", value: 16 }
  ],
  Spacing: [
    // e.g. { name: "4", type: "FLOAT", value: 16 }
  ],
  Radius: [],
  Border: [],
  Other: [],
};
// ================================================================

const results = [];

for (const [collectionName, variables] of Object.entries(tokenData)) {
  if (variables.length === 0) continue;

  const collections = await figma.variables.getLocalVariableCollectionsAsync();
  let collection = collections.find(c => c.name === collectionName);
  if (!collection) {
    collection = figma.variables.createVariableCollection(collectionName);
  }
  const modeId = collection.defaultModeId;

  for (const token of variables) {
    const existingVars = await figma.variables.getLocalVariablesAsync();
    let variable = existingVars.find(
      v => v.variableCollectionId === collection.id && v.name === token.name
    );

    if (!variable) {
      variable = figma.variables.createVariable(token.name, collection, token.type);
    }

    variable.setValueForMode(modeId, token.value);
    results.push(`${collectionName}/${token.name} (${token.type})`);
  }
}

return { created: results.length, variables: results };
```

### Notes for use_figma calls

- Always fill `tokenData` with the **actual values** extracted in the previous steps before executing
- Decide whether to run all collections in one call or split by collection based on the situation
- If the variable count is large (30+), split the calls by collection

## Step 5: Report completion

Report the following:

- Figma file name and fileKey (or URL)
- List of created collections and variable count per collection
- List of created variables (by collection)
- Skipped collections (those that were empty)
- Any variables that could not be converted, with reasons

## Error handling

- If no CSS file with `@import "tailwindcss"` is found: ask the user to provide the CSS file path
- If the `@theme` block is empty: report that no variables are defined
- If `create_new_file` fails: ask the user to check Figma login status (verifiable with `whoami`)
- If `use_figma` returns an error: check the error details and review the variable data format
- Values that cannot be converted (e.g. `var(--other)` references, `linear-gradient`, etc.) are skipped with a warning and included in the report
