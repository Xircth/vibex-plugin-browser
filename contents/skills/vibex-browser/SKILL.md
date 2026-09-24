---
name: vibex-browser
description: Drive VibeX's built-in Browser plugin and its MCP (vibex-browser-browser-tools). Use when the user wants to open, list, read, click, type, navigate, screenshot, or close in-app browser tabs (GitHub, Baidu, docs, web apps)—not the system browser.
---

# VibeX Browser

Use this Skill whenever the user wants to look at or operate a **web page inside VibeX**, not an external browser.

The tools live on the MCP server **`vibex-browser-browser-tools`**. They appear only after the Browser plugin is enabled **and** the current Agent session was opened or rebound after that. If the tools are missing, tell the user to enable **设置 → 插件 → 浏览器 → 智能体工具**, then start a **new** session. Do not invent a fallback that opens the OS browser.

## What this plugin is

VibeX Browser is a dock tab next to the editor. Each dock tab is one page (`tabId`). Cookies stay in the app profile. The user can also open tabs from the right rail or the `+` menu.

You do **not** control the address bar UI. You call MCP tools. New tabs you open with `browser_open_tab` show up in the workspace dock.

## Default share (grant)

New tabs default to **control** (可读可操作): you may list, snapshot, click, and type.

Grant is a standing setting on that tab. Same-site loads and cross-site navigations keep it. It becomes empty only when:

1. Plugin config **默认共享级别** is `none`, or
2. The user picked **停止共享** on the tab chrome.

Levels:

| UI | `grant.level` | You may |
| --- | --- | --- |
| 停止共享 / empty | missing / `none` | `browser_list_tabs` only (title, URL, tabId) |
| 可读 | `read` | snapshot |
| 可读可操作 | `control` | snapshot, click, type, eval (if enabled) |

If a tool returns `browser_grant_required`, tell the user to share that tab from the address-bar chip. If it returns `browser_control_required`, they shared read-only; ask them to switch to 可读可操作.

Do not tell the user to paste cookies, passwords, or DevTools protocol endpoints.

## Tools

Call these by name. Do not rename them. Do not call Host `browser.dispatch` yourself.

### `browser_list_tabs`

No arguments. Returns every built-in tab: `tabId`, `url`, `title`, `loading`, `origin`, `grant`.

Always list first when the user says “current page”, “this tab”, or “all tabs”, unless they just gave you a `tabId` in this turn.

Pick the tab they mean by URL/title. If several match, list them and ask. Never guess a UUID.

### `browser_open_tab`

Arguments: `{ "url": "https://..." }` (http or https only).

Opens a **new** dock tab and returns the tab object, including `tabId` and `grant`. Use this when they ask to open a site they are not already looking at.

Do not pass `javascript:`, `file:`, `data:`, or `about:` except that the Host may allow `about:blank` internally—you still open real http(s) URLs.

After open, wait until `loading` is false (list again) before snapshotting, unless they only wanted the tab created.

### `browser_navigate`

Arguments: `{ "tabId", "url" }`.

Points an **existing** tab at a new address. Prefer this over opening another tab when they say “go to”, “open in this tab”, or “navigate”.

### `browser_snapshot`

Arguments: `{ "tabId" }`, optional `maxChars`.

Reads the page as an ARIA tree. Returns `generation`, `url`, `title`, `snapshot`, `refsCount`, `truncated`.

- Refs look like `e1`, `e2`. They are valid **only until the next snapshot** on that tab (or a navigation).
- `generation` is required for the next click/type. Copy it exactly; do not invent one.
- If `truncated` is true, snapshot again with a larger `maxChars` or narrow the task.
- Requires at least **read** grant.

### `browser_click`

Arguments: `{ "tabId", "ref", "generation" }`, optional `kind` (`click` default).

Clicks the ref from the **latest** snapshot. Requires **control** grant. After a click that changes the page, snapshot again before the next click.

### `browser_type`

Arguments: `{ "tabId", "ref", "generation", "text" }`.

Types into the snapshot ref (inputs, textareas, contenteditable). Requires **control**. Snapshot again if the DOM changed.

### `browser_close_tab`

Arguments: `{ "tabId" }`. Closes that dock tab. Confirm if they did not clearly ask to close it.

### `browser_eval`

Arguments: `{ "tabId", "code" }`.

Runs JavaScript in the page. The Host **always** shows the snippet to the user first. Use only when snapshot/click/type cannot do the job (read computed style, call a page API).

Requires **control**, plugin **运行页面 JavaScript** enabled, and a new session after that flag is turned on. On `browser_eval_disabled`, tell them to enable that setting. Never hide code from the confirmation dialog. Never eval credentials.

## Playbooks

### Read the current page

1. `browser_list_tabs`
2. Choose `tabId`
3. If `grant` is empty, stop and ask them to share the tab
4. `browser_snapshot`
5. Answer from the tree; quote visible text, do not fabricate unseen content

### Open a site and summarize

1. `browser_open_tab` with the full https URL
2. `browser_list_tabs` until that `tabId` has `loading: false` (or snapshot once; retry if it fails as still loading)
3. `browser_snapshot`
4. Summarize

### Fill a form / click a button

1. List or open the tab
2. Snapshot
3. `browser_click` or `browser_type` with that `generation` and `ref`
4. Snapshot again
5. Repeat until done

### Search on a site (example)

User: open GitHub and search `vibex`

1. `browser_open_tab` `{ "url": "https://github.com" }`
2. Snapshot, find the search input ref
3. `browser_type` the query, then click the search control (or type then snapshot for the submit button)
4. Snapshot results and report

## Errors

| Signal | What you do |
| --- | --- |
| Tools absent / handshake failed | Plugin off, **智能体工具** off, or session too old. Enable plugin + tools, **new session**. |
| `browser_tools_disabled` | Same: turn on 智能体工具, new session. |
| `browser_grant_required` | Ask them to share the tab (chip next to the URL). |
| `browser_control_required` | Ask them to switch the chip to 可读可操作. |
| `browser_no_such_tab` | List tabs again; the tab was closed. |
| `browser_bad_address` | Only http(s). Ask for a real URL. |
| `browser_eval_disabled` | Enable 运行页面 JavaScript, new session. |
| Stale ref / generation | Snapshot again, then retry the act. |
| Click did nothing | Snapshot; the control may be in a closed menu, iframe, or canvas. Say so. Do not loop the same ref. |

## Do not

- Use the system default browser, `window.open`, or curl HTML as a stand-in for this plugin.
- Invent `tabId`, `ref`, or `generation`.
- Click or type without a snapshot from **this** tab in the current step chain.
- Keep clicking after `ok: false`; snapshot and re-plan.
- Retry a failed MCP handshake in a tight loop.
- Claim you read the page when you only listed title/URL.

## Config the user can change

**设置 → 插件 → 浏览器** (applies immediately; MCP tools still need a new session if tools were just enabled):

- **智能体工具** (`toolsEnabled`): master switch for this MCP.
- **运行页面 JavaScript** (`evalEnabled`): allows `browser_eval`.
- **默认共享级别** (`defaultGrant`): `control` (default), `read`, or `none` for newly opened tabs.
