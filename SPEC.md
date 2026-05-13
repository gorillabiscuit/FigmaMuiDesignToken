# TokenCheck — Figma Plugin Spec

## What this is

A Figma plugin that validates designs against an external MUI theme file (or any JSON/CSS token file) from your engineering codebase. It treats the engineering tokens as the source of truth and flags any Figma layer that uses a value not present in that token set.

## Why it exists

In teams with many designers, style drift between Figma and the codebase is inevitable. Someone hardcodes a colour, uses the wrong font size, or picks a spacing value that doesn't exist in the theme. This plugin catches that before it reaches engineering. It's the design equivalent of a code linter, but instead of checking code against a style guide, it checks Figma against the actual code tokens.

## Core flow

1. **Load tokens**: User pastes a URL or uploads a JSON file containing their MUI theme (or any structured token file). The plugin parses it and stores the token set.
2. **Scan**: User clicks "Lint" to scan the current page or their selection. The plugin traverses every visible node.
3. **Extract**: For each node, extract fills, strokes, text styles (font family, size, weight, line height), corner radii, and spacing (padding/gap via auto-layout).
4. **Compare**: Check each extracted value against the loaded token set. A value is "valid" if it matches any token value. A value is "unmatched" if no token maps to it.
5. **Report**: Display a list of violations grouped by type (colour, typography, spacing). Each violation shows the node name, the value found, and the closest matching token (if any). Clicking a violation selects and zooms to that node in Figma.

## Token file format

### Important: MUI themes are already structured

MUI enforces a rigid, well-documented theme structure via `createTheme()`. Developers don't need to do anything special to make their theme compatible with this plugin. Every MUI project already has colours at `palette.*`, typography at `typography.*`, spacing as a base unit, and border radii at `shape.borderRadius`. This is what makes the plugin viable: the format is predictable by design.

However, the developer's theme file is typically TypeScript (e.g. `theme.ts`) wrapped in a `createTheme()` call, which the plugin can't parse directly. The developer should either:
- Copy the theme object literal and paste it as JSON into the plugin, or
- Run a small script to `JSON.stringify` their theme and export it (e.g. `console.log(JSON.stringify(theme, null, 2))`)

The plugin works with the resulting JSON. It does not need to parse TypeScript.

The plugin should support two input formats:

### MUI Theme JSON (primary)

```json
{
  "palette": {
    "primary": {
      "main": "#5B3FE4",
      "light": "#8066FF",
      "dark": "#3D2A9E",
      "contrastText": "#FFFFFF"
    },
    "secondary": {
      "main": "#E44B3F",
      "light": "#FF7B6F",
      "dark": "#A8332A"
    },
    "background": {
      "default": "#FAFAFA",
      "paper": "#FFFFFF"
    },
    "text": {
      "primary": "#1A1A1A",
      "secondary": "#666666",
      "disabled": "#BDBDBD"
    }
  },
  "typography": {
    "fontFamily": "Inter, sans-serif",
    "h1": { "fontSize": "2.5rem", "fontWeight": 700, "lineHeight": 1.2 },
    "h2": { "fontSize": "2rem", "fontWeight": 700, "lineHeight": 1.3 },
    "body1": { "fontSize": "1rem", "fontWeight": 400, "lineHeight": 1.5 },
    "body2": { "fontSize": "0.875rem", "fontWeight": 400, "lineHeight": 1.43 },
    "button": { "fontSize": "0.875rem", "fontWeight": 600, "textTransform": "uppercase" }
  },
  "shape": {
    "borderRadius": 8
  },
  "spacing": 8
}
```

### Flat CSS custom properties

```json
{
  "--color-primary": "#5B3FE4",
  "--color-secondary": "#E44B3F",
  "--font-size-body": "14px",
  "--font-size-h1": "40px",
  "--spacing-sm": "8px",
  "--spacing-md": "16px",
  "--spacing-lg": "24px",
  "--radius-default": "8px"
}
```

The plugin should auto-detect the format. If the JSON has a `palette` key, treat it as MUI theme and flatten it. If it's flat key-value pairs, use directly.

### Flattening logic for MUI themes

Recursively walk the theme object and extract all leaf values. Store them with their dot-path as the token name:

- `palette.primary.main` → `#5B3FE4`
- `typography.body1.fontSize` → `1rem` (convert to px: 16px)
- `shape.borderRadius` → `8` (treat as px)
- `spacing` → `8` (MUI spacing is a base unit; valid values are multiples: 8, 16, 24, 32...)

### Unit conversion

MUI uses rem for typography. The plugin needs a base font size setting (default 16px) to convert rem to px for comparison against Figma values, which are always in px.

MUI spacing is a base unit (default 8). Valid spacing values are multiples of this base. So if spacing is 8, then 8, 16, 24, 32, 40... are all valid.

## What to lint

### Colours (fills and strokes)

- For every node with solid fills or strokes, extract the hex colour value
- Compare against all colour tokens from the loaded file
- Flag if the colour doesn't match any token
- Show the closest matching token (by colour distance) as a suggestion
- Ignore nodes with image fills or gradients (skip, don't flag)

### Typography

- For every TextNode, extract: fontFamily, fontSize (px), fontWeight, lineHeight (px)
- Compare each property against typography tokens
- Flag mismatches per property (e.g., "fontSize 15px doesn't match any token; closest is body1 at 16px")
- Handle mixed text styles within a single text node (Figma allows this)

### Corner radius

- For every node with cornerRadius, compare against shape.borderRadius and any other radius tokens
- Flag non-matching values

### Spacing (auto-layout only)

- For frames with auto-layout, extract: itemSpacing, paddingTop/Right/Bottom/Left
- Check if each value is a valid multiple of the spacing base unit
- Flag values that aren't valid multiples

## What NOT to lint

- Opacity/alpha (too many valid use cases for non-token opacity)
- Shadows/effects (v2 feature)
- Layout constraints (not token-related)
- Hidden/invisible nodes (skip them)
- Nodes inside components from external libraries (only lint local work)

## Plugin UI

The plugin runs in a panel (not a modal). Use Figma's standard plugin UI pattern: an iframe with HTML/CSS/JS.

### Screens

**1. Setup screen (shown on first run or when no tokens loaded)**

- Heading: "TokenCheck"
- Subheading: "Validate your designs against engineering tokens"
- Textarea to paste JSON token content
- OR a file upload button to load a .json file
- "Base font size" input (default: 16px, for rem conversion)
- "Spacing base unit" input (default: 8px)
- "Load Tokens" button
- Show count of tokens parsed: "Loaded 47 tokens (23 colours, 12 typography, 4 spacing, 2 radii)"

**2. Results screen (shown after linting)**

- Summary bar: "14 issues found across 8 layers" with breakdown by type
- Filter tabs: All | Colours | Typography | Spacing | Radius
- Violation list, each item showing:
  - Layer name (clickable, selects + zooms to node)
  - Issue type icon (colour swatch, text icon, spacing icon, radius icon)
  - Description: "Fill #2563EB doesn't match any token. Closest: palette.primary.main (#5B3FE4)"
- "Re-lint" button
- "Change tokens" link to go back to setup

**3. Clean state**

- When no violations found: "All clean. Every value matches your tokens." with a checkmark

### Visual style

Keep it clean and minimal. Use Figma's plugin UI conventions:
- 11px font size for body text
- Figma's standard colours for backgrounds and borders
- No heavy branding, no gradients
- The focus is readability and utility

## Architecture

Follow the Design Lint pattern (MIT, https://github.com/destefanis/design-lint):

```
tokencheck/
├── manifest.json
├── package.json
├── tsconfig.json
├── webpack.config.js
├── src/
│   ├── plugin/
│   │   ├── controller.ts        # Figma API: traverse nodes, send messages to UI
│   │   └── lintingFunctions.ts  # All comparison/matching logic
│   ├── app/
│   │   ├── components/
│   │   │   ├── App.tsx          # Main UI component
│   │   │   ├── SetupScreen.tsx  # Token loading screen
│   │   │   ├── ResultsScreen.tsx # Violation list
│   │   │   └── ViolationItem.tsx # Single violation row
│   │   ├── styles/
│   │   │   └── styles.css       # Plugin styles
│   │   └── index.html           # Plugin UI entry
│   └── shared/
│       ├── types.ts             # Shared types between plugin and UI
│       └── tokenParser.ts       # Parse MUI theme / flat tokens into normalised format
```

### manifest.json

```json
{
  "name": "TokenCheck",
  "id": "tokencheck-plugin",
  "api": "1.0.0",
  "main": "dist/code.js",
  "ui": "dist/ui.html",
  "documentAccess": "dynamic-page",
  "editorType": ["figma"],
  "networkAccess": {
    "allowedDomains": ["none"]
  }
}
```

Note: `networkAccess` is set to none because the plugin reads pasted/uploaded JSON, it doesn't fetch from URLs. If we add URL fetching later, add the relevant domains.

### Communication

Figma plugins use message passing between the plugin sandbox (controller.ts, which has access to the Figma API) and the UI iframe (React app):

- UI → Plugin: `parent.postMessage({ pluginMessage: { type: 'LINT', ... } }, '*')`
- Plugin → UI: `figma.ui.postMessage({ type: 'LINT_RESULTS', violations: [...] })`

### Key Figma API methods

```typescript
// Traverse nodes
figma.currentPage.children        // top-level nodes
node.findAll(n => n.visible)      // find all visible nodes recursively

// Check if a fill uses a style or is hardcoded
node.fillStyleId                  // SymbolId if using a style, '' if hardcoded
node.fills                        // Array of Paint objects with colour values

// Text properties
textNode.fontName                 // { family: string, style: string }
textNode.fontSize                 // number (px)
textNode.lineHeight               // { value: number, unit: 'PIXELS' | 'PERCENT' | 'AUTO' }
textNode.getRangeFontSize(0, 1)   // for mixed text styles

// Auto-layout spacing
frame.itemSpacing                 // gap between children
frame.paddingTop / paddingRight / paddingBottom / paddingLeft

// Corner radius
node.cornerRadius                 // number or figma.mixed

// Select and zoom to a node
figma.currentPage.selection = [node]
figma.viewport.scrollAndZoomIntoView([node])
```

### Token storage

Store the parsed tokens in Figma's clientStorage so they persist between sessions:

```typescript
await figma.clientStorage.setAsync('tokens', parsedTokens)
const tokens = await figma.clientStorage.getAsync('tokens')
```

## Colour matching

For "closest match" suggestions, use CIE76 colour distance (Euclidean distance in LAB colour space). This is more perceptually accurate than comparing RGB values. Libraries like `color-diff` or a simple LAB conversion + distance calc work fine.

If exact hex match: valid.
If no exact match: flag as violation, show closest token with distance.

## Types

```typescript
interface ParsedToken {
  path: string           // e.g. "palette.primary.main" or "--color-primary"
  value: string          // normalised value, e.g. "#5B3FE4", "16", "Inter"
  category: 'color' | 'typography' | 'spacing' | 'radius'
  originalValue: string  // original value from theme, e.g. "1rem"
}

interface Violation {
  nodeId: string
  nodeName: string
  nodeType: string       // 'FRAME' | 'TEXT' | 'RECTANGLE' etc
  property: string       // 'fill' | 'stroke' | 'fontSize' | 'fontFamily' | 'cornerRadius' | 'itemSpacing' | 'padding'
  foundValue: string     // what Figma has
  category: 'color' | 'typography' | 'spacing' | 'radius'
  closestToken: ParsedToken | null
  distance: number       // how far from closest match (0 = exact match but wrong token name)
}

interface LintSummary {
  totalViolations: number
  totalNodes: number
  nodesWithIssues: number
  byCategory: {
    color: number
    typography: number
    spacing: number
    radius: number
  }
}
```

## Build setup

- TypeScript
- React for the UI (keep it simple, no heavy state management)
- Webpack for bundling (standard Figma plugin setup)
- No external runtime dependencies beyond React. Keep the bundle small.
- The colour distance calculation can be a local utility function, no need for a library.

## Development workflow

1. `npm install`
2. `npm run build:watch` (webpack in watch mode)
3. In Figma: Plugins → Development → Import plugin from manifest → select manifest.json
4. Run the plugin, paste in your MUI theme JSON, hit lint

## V1 scope (build this)

- Parse MUI theme JSON and flat CSS token JSON
- Lint colours (fills + strokes)
- Lint typography (font family, size, weight)
- Lint corner radius
- Lint spacing (auto-layout itemSpacing + padding)
- Show violation list with click-to-navigate
- Persist loaded tokens between sessions
- Clean, minimal UI

## V2 scope (don't build yet, but design for extensibility)

- Fetch tokens from a URL (GitHub raw file, API endpoint)
- Export Figma values as tokens (reverse direction)
- Shadow/effect linting
- Severity levels (error vs warning)
- Ignore rules (mark specific layers as intentionally non-standard)
- CI integration (headless linting via Figma REST API)

## Reference

- Design Lint source: https://github.com/destefanis/design-lint
- Figma Plugin API: https://www.figma.com/plugin-docs/api/figma/
- Figma Plugin Quickstart: https://www.figma.com/plugin-docs/plugin-quickstart-guide/
- MUI Default Theme: https://mui.com/material-ui/customization/default-theme/
- MUI Theming: https://mui.com/material-ui/customization/theming/
