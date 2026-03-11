---
name: obsidian-best-practices
description: Best practices and established patterns for developing, debugging, and maintaining Obsidian plugins and themes. Make sure to use this skill whenever the user is building or modifying an Obsidian plugin or theme, dealing with Obsidian API issues, fixing plugin load errors, doing error hunting, or struggling with CSS/UI adjustments inside Obsidian.
---

# Obsidian Plugin & Theme Best Practices

This skill outlines crucial patterns, common pitfalls, and effective debugging strategies for developing within the Obsidian ecosystem.

## 1. Safety and Null Checking

When interacting with Obsidian's APIs—especially internal or undocumented ones—defensive programming is mandatory.

- **Internal API Gracefulness**: Never assume `app.internalPlugins` or sub-plugins are fully loaded or exist, as this will crash the plugin load sequence.

  ```javascript
  // Bad: will crash if workspaces plugin is disabled or loading
  const ws = app.internalPlugins.plugins.workspaces.instance.activeWorkspace;

  // Good: safely traverse the object
  const ws =
    app.internalPlugins?.plugins?.workspaces?.instance?.activeWorkspace || "";
  ```

- **File Access Verification**: If your plugin binds to events that deal with a current file, but the active file event returns null/undefined, gracefully fall back by checking `app.workspace.getActiveFile()`.

- **Regex and Document Parsing**: If you are doing manual scrapes of the DOM (like checking `document.title` for version strings), always verify `.match()` returns an array before accessing properties like `.length`.

## 2. Editor & CodeMirror 6 (Live Preview)

In Obsidian's "Live Preview" mode, standard HTML elements (like `<a>` tags with `href` attributes) are often replaced by CodeMirror 6 widgets. For example, a Markdown link `[text](url)` might render as a series of spans (`.cm-link`, `.cm-url`) in the DOM, but the `.cm-url` span may be completely empty of text and lack an `href` attribute.

- **Extracting Data under the Cursor/Mouse**: Never rely solely on DOM attributes (`element.href` or `element.innerText`) when hovering over editor content. Instead, convert the mouse coordinates to a CodeMirror document position, and read the raw Markdown text from the editor state:

  ```javascript
  // 1. Get the CodeMirror EditorView instance
  const leaf = this.app.workspace.getMostRecentLeaf();
  const cmView = leaf?.view?.editor?.cm;
  if (!cmView) return;

  // 2. Convert mouse event coordinates to a document position
  const pos = cmView.posAtCoords({ x: event.clientX, y: event.clientY });
  if (pos === null) return;

  // 3. Extract the surrounding line text from the document state
  const lineStart = cmView.state.doc.lineAt(pos).from;
  const lineEnd = cmView.state.doc.lineAt(pos).to;
  const lineText = cmView.state.doc.sliceString(lineStart, lineEnd);
  
  // 4. Parse the lineText (e.g., with regex) to find the actual URL or markdown token
  ```

## 2. Plugin Structure & Installation

- **manifest.json Versioning**: All plugins _must_ declare a `minAppVersion` in their `manifest.json`. Without this, modern versions of Obsidian will reject the plugin and it will fail to enable.
  ```json
  {
    "id": "my-plugin",
    "name": "My Plugin",
    "version": "1.0.0",
    "minAppVersion": "0.15.0"
  }
  ```

- **Manual Installation Naming**: When cloning or manually placing a plugin folder into `.obsidian/plugins/`, the directory name exactly must match the `id` string inside its `manifest.json`. If they do not match, Obsidian will silently fail to recognize the plugin. (e.g., if `id: "bases-kanban"`, the folder cannot be named `obsidian-bases-kanban`).

- **First-Time Loading**: If you manually install a new plugin, `obsidian plugin:reload id=<plugin-id>` will return a "Plugin not found" error because the plugin is not yet indexed by the app. You must first ensure it is added to `community-plugins.json`, and then run a full `obsidian reload` to restart the vault environment.

## 3. UI and Window Overrides

- **Emptying Titles / Hiding Text**: If you are overriding native Obsidian text structures (like the application window title) to be entirely blank, a simple empty space (`" "`) is rarely sufficient. Obsidian's core engine often strips pure whitespace and reverts to its default backup text (like the standard Vault + Filename title).
  - **Solution**: Inject a **Zero-Width Space** (`\u200B`). This safely tricks the application into rendering a purely blank element that Obsidian respiects as "contentful text".

## 4. The CLI Debugging Workflow

If developing plugins, you do not need to rely on the physical inspector window inside Obsidian. Leverage the `obsidian-cli` (installed globally):

- **Fast Iteration Loop**: Avoid manually toggling plugins in the settings. Chain your build logic with the CLI reload to instantly reflect code changes:
  ```bash
  npm run build && obsidian plugin:reload id=<your-plugin-id>
  ```
- **Error Hunting**: If a plugin fails to load quietly, or crashes silently, stream the errors directly to your terminal:
  ```bash
  obsidian dev:errors
  ```
- **Live Context Execution**: Check the state of the app without needing `console.log` trace logs. Execute code directly:
  ```bash
  obsidian eval code="app.plugins.enabledPlugins.has('my-plugin')"
  ```
- **Live Memory & View Inspection**: When visual bugs or silent state failures occur without explicit console errors (e.g., a card failing to snap into a new kanban column), use `obsidian eval code="..."` to traverse the live component UI tree. For example, looking into `app.workspace.getLeavesOfType('view-type')[0].view...` allows you to inspect deep configuration settings and runtime variables powering active workspace views in real-time.

## 5. UI Elements Customization

When creating DOM elements for a plugin (like file trees or recent file views), adhering to the following ensures it feels like native Obsidian UI:

- **Avoiding Hover "Jumpiness"**: Place dynamic, on-hover UI actions (like delete 'x' buttons) carefully. Instead of transitioning from `display: none` to `display: flex` (which shifts other elements and causes visual jumpiness), use `visibility: hidden` and `visibility: visible` to preserve layout size. If the element should overlap or appear precisely in a corner, leverage `position: absolute` so it falls out of the document flow and prevents reflowing the flex/grid parents.

- **Grid for Multi-Line Items**: If a list item needs to stack information (e.g., a file name on top, a parent folder path below), `display: grid` is often superior to flexbox. By defining `grid-template-columns` and `grid-template-rows` (with the bottom text set to `grid-column: 1 / -1; grid-row: 2;`), you guarantee exact alignment between the parent and children without messy left margins.

- **Native CSS Variables**: Never hardcode colors, fonts, or cursors unless strictly necessary. Utilize Obsidian's built-in CSS variables for seamless theme compatibility:
  - Text colors: `var(--text-normal)`, `var(--text-muted)`, `var(--text-faint)`
  - Fonts: `var(--font-smallest)`, `var(--font-smaller)`
  - Interactions: `var(--nav-item-color-hover)`, `var(--cursor)`

- **Icon Alignments**: When embedding SVGs next to text, use relative `em` units instead of absolute pixels (e.g., `width: 0.85em; height: 0.85em`) combined with `vertical-align: middle` or a clean `align-items: center` flex wrapper to keep the icon exactly vertically aligned with the adjacent text.

- **Nav/Tree Structure**: When building sidebars (like file explorers), try to match or mimic the HTML structure Obsidian uses for its native sidebars. If using custom DOM elements, recreate the flex-grow structure carefully so that elements fill horizontal space appropriately (e.g., `.tree-item-spacer { flex-grow: 1; }`).
