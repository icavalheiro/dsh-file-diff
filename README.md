# dsh-file-diff — File Diff Overview (File Change Overview)

> **Deprecated: dsh has supported this since version dsh-v0.1.6-alpha.2.**

DSH Web plugin for a file change overview: it shows an "N modified files" row at the end of each conversation turn (click a file chip to open that file's diff in the **right sidebar**) and provides a "Change history" button in the session header (the session-level file change overview). The diff view is rendered as a right-sidebar tab (kind `filediff`) and reuses the sidebar dock panel capabilities (zoom, fullscreen/push, float, and close). **Pure plugin implementation**: it does not modify the DSH repository; its built-in tokenizer supports GDScript (`.gd`), GDShader (`.gdshader`), and GDResource (`.gdresource`), and it self-registers highlighted sidebar previews for these files.

------

Session file change overview
![Modified files](./snapshot/files_1.png)
Individual file changes (right-sidebar tab)
![File changes](./snapshot/fdiff_1.png)

## Installation (install this package as an official client package in ~/.dsh/profiles/web/)

### 1. Let the profile Loader resolve the package

**Principle (read first)**: `client-modules` scans the Loader entry. The `name` on that composition line is the
**package specifier**, which the Loader resolves as a node from the profile tree (rooted at `%HOME%\.dsh\profiles\web\`).
It resolves the package's `package.json`, then reads the `dsh.client` declaration and `exports["./client"]` to register the web plugin.
There is only one key requirement: **the package must be discoverable by the profile's node resolver** (usually at
`profiles/web/node_modules/dsh-file-diff`). It can be a pnpm link, a junction, or a copied directory.

> ⚠️ `healProfileModuleFallback` projects only packages in the "selected bundle dependency closure" to
> `.dsh-module-fallback\node_modules`, and **deletes** owned links outside that closure at startup.
> Therefore, **do not put user packages in `.dsh-module-fallback`** (the cleanup logic has deprecated the old approach).

**Recommended approach A: add it as a profile dependency (the standard option; source stays in place and is not copied)**

Add the following to `dependencies` in `%HOME%\.dsh\profiles\web\package.json` (`file:` paths are resolved relative to
the profile directory; use an absolute path across drives and use forward slashes):

```json
"dsh-file-diff": "link:./dsh-file-diff"
```

Then run this in that directory:

```sh
cd %HOME%\.dsh\profiles\web
pnpm install
```

pnpm adds it as a link at `profiles/web/node_modules/dsh-file-diff` (the actual files remain in your workspace; no duplicate copy is created).

**Approach B: create a junction manually (no dependency installation)**

```sh
mklink /J "%HOME%\.dsh\profiles\web\node_modules\dsh-file-diff" ".\dsh-file-diff"
```

**Approach C: copy directly** (works, but is not managed by pnpm and can drift; not recommended):

```sh
robocopy .\dsh-file-diff "%HOME%\.dsh\profiles\web\node_modules\dsh-file-diff" /E
```

### 2. Add a line to the profile composition

Edit `%HOME%\.dsh\profiles\web\cordis.patch.yml` and append the following. **`name` must be the package name
`dsh-file-diff`** (the Loader resolves it by name, and client-modules also uses it to validate bundle registration; see below).
`id` is only the identifier for this composition line and can be arbitrary:

```yaml
- insert:
    - id: filediff-overview
      name: 'dsh-file-diff'
```

The Loader must resolve a new package name at startup (the module link from step 1 must also exist before startup). Once complete, **restart the dsh web process**:

```sh
dsh web --profile web
```

### 3. This package's `dsh.client` declaration (no manual action; applied automatically during build/startup)

- `inject` adds `@deepseek-ai/dsh-client-ui-sidebar-right` (provides `ctx.sidebarRight` /
  `ctx.sidebarRightTabs`, ensuring the sidebar composes first so the plugin can register its tab) and
  `@deepseek-ai/dsh-client-ui-sidebar-documentpreview` (provides `ctx.documentPreviews` and the
  `sidebar.right.tab.document` document-preview seat, which the plugin uses to register gd-family previews).
- The plugin bundle only `require('react')` (a platform seed word), with **no external dependencies and no runtime imports from
  `@deepseek-ai/*`**. Highlighting is provided by the plugin's built-in tokenizer, so the harness client bundle does not need to be rebuilt.

### 4. Verify

- After startup, open any session and have the agent modify a file. An "N modified files" row should appear at the end of each turn.
- Click a file chip. The right sidebar should open or focus the "Change history" tab and show the file's line-numbered, syntax-highlighted diff.
- Use the header "Change history" button or the inline "All changes" button to open the session-level change overview, then drill into a file.
- Open a `.gd` / `.gdshader` / `.gdresource` file in the sidebar `files` tab. The plugin should render a highlighted preview
  (line numbers and syntax colors); changes to these files in the diff should be highlighted as well.
- Read a `.gd`-family file with the `read` tool. The card remains plain text (without highlighting), which is a known limitation of the pure-plugin approach.
- Confirm that the browser console has no `client-modules` composition errors. If the package is not resolved, errors such as `client-modules: 1 client package failed to compose` may appear.

## Directory structure

```
package/
├── package.json              # name + dsh.client declaration (inject) + exports["./client"] + scripts.build
├── scripts/
│   └── build-client.mjs      # build: lib/styles.css + lib/client.template.js → lib/client.js
└── lib/
    ├── styles.css            # ★ single style source (edit directly; plain CSS needs no escaping)
    ├── client.template.js    # ★ application logic template (contains the __PLUGIN_CSS__ placeholder)
    ├── client.js             # deployable output (GENERATED by npm run build; do not edit manually)
    ├── index.js              # node half placeholder (client-only plugin; host logic is empty)
    └── types/                # type placeholders
```

`lib/client.js` is generated by `npm run build` from `lib/styles.css` + `lib/client.template.js`. The output retains the `__ModuleLoader__.load({ id: "dsh-file-diff" })` format, matching ui-deliverables, and `ctx.get('uiConversation')` / `ctx.get('slots')` / `ctx.get('sidebarRight')` inside `apply` degrade safely when unavailable.

## Manually adjust styles (style workflow)

The **single source of truth** for styles is `lib/styles.css` (plain CSS; edit it directly without escaping). After editing, run:

```sh
npm run build     # regenerate lib/client.js
```

- `link:` deployment (recommended): the junction points directly to the workspace, so **restart web after building**; no dependency reinstall is needed.
- `file:` deployment: pnpm snapshots a copy during installation, so run `pnpm install --force` to resync after building, then restart.

The main colors for added/deleted diff rows, file headers, and the turn list are in `lib/styles.css`; **token colors are controlled by the plugin tokenizer's `.fdiff-tok-*` classes** (the pure-plugin approach does not depend on the harness shared highlighter).

> Do not edit `lib/client.js` directly (GENERATED; the next build will overwrite it). Edit `lib/client.template.js` for logic changes.

## How it works

- **Session node**: register a `kind: 'filediff'` node with `uiConversation.events.register`. During `update`, track `write` / `edit` tool calls and results, aggregate each successful change into `{ seq, path, diffs }`, and store it by turn in the turn node data (key `filediff`).
- **Two data sources**: the turn row prefers node data; the session overview rebuilds the same shape from the Trajectory ledger (`eventNodes` + `eventLocations`) using `collectSessionChanges`. The right-sidebar tab body is **fully stateless**, rebuilding from that ledger and navigation parameters, so the session overview does not depend on live node data and works across replays.
- **Right-sidebar tab** (replaces the old custom `shell.overlay` drawer):
  - Type registration: `ctx.sidebarRightTabs.register({ id: 'dsh-file-diff', kind: 'filediff', ... })` (page type, address `sidebar://filediff`, with a guide entry).
  - Body / title: `sidebar.right.pane.tab` and `sidebar.right.pane.tab.title` (keyed, key = `dsh-file-diff`).
  - Navigation parameters: `{ mode: 'file', path }` or `{ mode: 'session' }`; the body renders from `useTabInfo().tab.navigation.params` + `useTrajectory`. Drill-down inside the tab uses `tab.actions.openTab('filediff', { params })` to navigate the same tab.
- **Two session entry points**: `conversation.chat.turnTail` (chain, selector `selectFileDiffs`) -> the "N modified files" row; `conversation.session.header.utilities` (list) -> the "Change history" button. Clicking either uses `ctx.sidebarRight.openTab` to open or focus the tab.
- **Diff engine**: compact LCS (longest common subsequence).
- **Syntax highlighting (pure plugin tokenizer)**: a compact regex tokenizer with per-language rules for comments, strings, numbers, and keywords;
  `.gd` -> gdscript, `.gdshader` -> gdshader, and `.gdresource` -> gdresource (`gdresource` highlights `[...]`
  section headers as keywords, with embedded key/value pairs and `ExtResource("...")` colored normally). **The DSH repository is not modified**.
  Therefore, only the plugin's diff view and its self-registered gd sidebar preview are highlighted; the `read` tool card and other surfaces
  keep `.gd`-family files as plain text (this is the boundary of the pure-plugin approach).
- **gd sidebar preview (plugin-rendered)**: `ctx.documentPreviews.register({ id: 'dsh-file-diff/gd-preview',
  extensions: ['gd','gdshader','gdresource'], priority: 'extension', ... })` overrides the built-in plain-text preview with extension priority.
  Its body is registered in `sidebar.right.tab.document` (key = that id) and renders source with line numbers and syntax colors using the same tokenizer.

### Troubleshooting common errors

- **`client-modules: bundle ... loaded without registering "dsh-file-diff"`**
  -> The `id` in `__ModuleLoader__.load({ id })` in `lib/client.js` must be the **package name `dsh-file-diff`**,
    not `filediff-overview`. A wrong id means the bundle is not registered and the entire Loader entry import fails.
- **`client-modules: duplicate factory registration for "@deepseek-ai/..." (bundle executed twice)`**
  -> This is a chain reaction from the previous error: bundle registration fails, the Loader retries the entire combo, and the first module is registered twice.
    After fixing the id, **fully restart the web process** and perform a **hard refresh** in the browser to clear the old combo state.
- **`client-modules: 1 client package failed to compose`**
  -> The package was not resolved by the Loader. Confirm that the package exists in `profiles/web/node_modules` and that the composition line's `name` is the package name.
- **`require("...") missed the module table`**
  -> An external required by the bundle is not in `dsh.client.external` (and is not in the web boot graph).
    This package only `require('react')` (a platform seed word), so this error should not occur; if it does, an external dependency was added accidentally.
- **Clicking a chip does nothing in the sidebar** -> Confirm that the right sidebar has composed (the web bundle includes it by default) and check the console for
  `sidebarRight: no session surface is mounted` (normally swallowed by the plugin's try/catch, resulting in no response; extremely rare).
- **The two ids are different**: composition line `id: filediff-overview` (arbitrary); bundle registration
  `__ModuleLoader__.load({ id: "dsh-file-diff" })` (**must equal the package name**, as client-modules validates it).

## Rollback

- Delete the `insert` block added to `cordis.patch.yml`.
- Remove the dependency from `package.json` (or delete the junction / copied directory).
- Restart web.
