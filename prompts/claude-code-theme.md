# BUILD PROMPT — Rebuild the "claude-code" theme + presentation extension for Pi

Copy everything between the `===BEGIN PROMPT===` markers into your Pi agent.
Fill the `[FILL]` items first. The prompt is self-contained: the agent has never seen
the original extension or theme and must not need them.

===BEGIN PROMPT===

You are a senior engineer. Build a complete, working Pi theme and its companion
presentation extension from scratch. Do not install anything from the public npm
registry; everything is written in-repo and reviewable.

## 0. Environment
- Pi version: [FILL: the `pi` version you are building against]. Read the installed
  types from `@earendil-works/pi-coding-agent` and `@earendil-works/pi-tui` before
  writing code — do not guess an API, especially for `ctx.ui`, `CustomEditor`, and the
  theme colour roles.
- Theme schema: Pi ships a JSON schema at
  `.../coding-agent/src/modes/interactive/theme/theme-schema.json`. Locate it in the
  installed package and validate against it; do not invent colour role names.
- No network access and no model calls. This is a presentation-only package.

## 1. Goal
Reproduce a "Claude Code"-style terminal presentation for Pi, in two artefacts:
a **theme file** (colours only) and a **presentation extension** (header, footer, editor
chrome, one command). The extension changes how Pi LOOKS and never changes how it
behaves: no tool interception, no message rewriting, no model or prompt changes, no
network, no file access beyond its own state.

## 2. Package layout (exact)
    claude-code-theme/
      package.json          # name, private, "keywords": ["pi-package"],
                            # pi.extensions: ["./index.ts"], pi.themes: ["./claude-code.json"]
                            # (or document installing the theme into <agent dir>/themes/)
      README.md             # what it is, how to enable, screenshots-as-text, limits
      LICENSE
      index.ts              # presentation extension
      claude-code.json      # the theme
      test.mjs              # tests against the installed Pi

## 3. The theme (`claude-code.json`)
Top level: `$schema` (pointing at the schema file from §0), `name: "claude-code"`,
`appearance: "dark"`, plus `vars`, `colors`, and `export` maps.

Required palette (`vars` → `colors`), with these exact values where given:
- `claude: "#d77757"` (the Claude orange, used for inline markdown code), `muted: "#999999"`,
  `dim: "#808080"`, `border: "#888888"`, `subtle: "#505050"`, `permission: "#b1b9f9"`,
  `violet: "#af87ff"`, `blue: "#82aadc"`, `green: "#4eba65"`, `red: "#ff6b80"`,
  `yellow: "#ffc107"`, `pink: "#fd5db1"`, `userBg: "#373737"`, `selection: "#2c323e"`,
  `page: "#1c1c29"`, and `foreground: ""` (meaning: inherit the terminal's default).
- Role mapping: `accent`/`borderAccent`/`syntaxKeyword` → `violet`; `border`/`borderMuted`/
  `thinkingOff` → `border`; `success` → `green`; `error` → `red`; `warning`/
  `searchMatchBg` → `yellow`; `mdCode` → `claude`; `mdLink`/`customMessageLabel`/
  `syntaxFunction` → `permission`; `userMessageBg` → `userBg`; `userMessageText: "#ffffff"`;
  `selectedBg` → `selection`; `searchMatchText`/`export.pageBg` → `page`;
  `toolErrorBg: "#302026"`; `bashMode` → `pink`.
- Thinking roles must escalate: `thinkingOff` → border, `thinkingMinimal` → muted,
  `thinkingLow` → blue, `thinkingMedium` → permission, `thinkingHigh`/`thinkingXhigh`/
  `thinkingMax` → violet.
- Syntax: `syntaxComment` → muted, `syntaxString: "#91c882"`, `syntaxNumber: "#f5ab73"`,
  `syntaxType: "#6a9bcc"`, `syntaxOperator`/`syntaxVariable` → foreground,
  `syntaxPunctuation` → muted.
- Diffs: `toolDiffAdded` → green, `toolDiffRemoved` → red, `toolDiffContext` → muted.
- `export`: `pageBg` → page, `cardBg: "#242432"`, `infoBg` → selection.

Leave a role unset rather than inventing a colour you cannot justify; the schema is the
source of truth for which roles exist.

## 4. The extension (`index.ts`)
`export default function (pi: ExtensionAPI)`.

### 4.1 Hard gating rule
Everything is active ONLY while `ctx.ui.theme.name === "claude-code"`. On
`session_start`, install if the theme matches, otherwise uninstall. If the theme is not
installed, `/claude-code on` must fail with a clear notification, not a broken UI.

### 4.2 Mascot ("Clawd") — animated, in the header
- Three rows of box-drawing art, 9 columns wide, rendered per half-cell; body colour
  `rgbColor(215, 119, 87)` (orange).
- Under `max` thinking effort the body turns purple `rgbColor(122, 47, 243)` and a
  shimmer band crosses it, mixing toward `rgbColor(210, 214, 241)` with weights
  `1 / 0.55 / 0.25 / 0` at distance thresholds `0.6 / 1.6 / 2.6`.
- The band travels RIGHT TO LEFT: with `SWEEP_FRAMES = 9` and `SWEEP_PAUSE = 10`, the
  position is `9 - (step / SWEEP_FRAMES) * 12` during the sweep and off-screen (`-99`)
  during the pause. The band must not wrap around the row.
- A two-column sparkle sits to the right of the first row only: a glint `╱` in
  `rgbColor(140, 85, 35)` then a star `✦` in `rgbColor(233, 90, 15)`. It is blank unless
  effort is `max` (the "ultracode" state).
- Animation: a `setInterval` at 160 ms incrementing a frame counter and calling
  `requestRender()`. Start it when the footer mounts; `clearInterval` on uninstall and on
  `session_shutdown`. Never leak a timer.

### 4.3 Header
Three lines, blank line above and below: `Pi Code v<VERSION>` in bold, then
`<model id> · <level> effort · <provider>`, then the abbreviated cwd (`~` for home). When
`width >= 40`, prefix each line with the mascot row; below that, render text only.

### 4.4 Editor chrome (`extends CustomEditor`)
- Keep Pi's wrapping, mouse coordinates, paste handling, autocomplete, IME cursor,
  history, and shortcuts. Use existing padding, never extra columns.
- Only when the theme is active and `width >= 6`: force padding to
  `min(max(configuredPadding, 2), max(0, floor((width - 1) / 2) - 2))`, restoring the
  configured value otherwise. Replace the leading two spaces of the second line with a
  `❯ ` prompt glyph in the `text` colour.
- Border colour follows run state: `bashMode` when the draft starts with `!`, else
  `borderMuted`. Always restore the original `borderColor` in a `finally` block.
- Top border gains a right-aligned effort label ONLY when the theme is active, the app is
  idle, nothing is hidden above, and `width >= 28`: `" ultracode "` in `thinkingMax` when
  effort is `max`, otherwise `" <level> effort "` in the matching `thinking<Level>`
  colour, followed by one border-coloured `─`.
- `uninstall` must restore the previous editor component and leave other extensions'
  editors alone: if `ctx.ui.getEditorComponent()` is not the one we installed, do nothing.

### 4.5 Footer
One line: left side `muted` text `<cwd> · <provider>/<model id>` plus ` · <git branch>`
when on a branch (via the footer data provider); right side a status block —
`◈ <level> · /thinking`, or for `max` effort `◈ <level> · ultracode · /thinking` with
`ultracode` in `thinkingMax` and the rest muted. Pad with at least one space and truncate
to width.

### 4.6 Lifecycle and command
- `install(ctx)`: no-op unless `ctx.mode === "tui"`. Sets header, footer, and (only if no
  foreign editor is already installed) the editor component. Idempotent.
- `uninstall(ctx)`: clears header and footer, restores the editor only if it is ours,
  stops the animation.
- `session_shutdown` stops the timer. Do not uninstall on shutdown — the next session
  re-installs.
- `/claude-code [on|off]`, default `on`. `on`: remember the current theme name, then
  `ctx.ui.setTheme("claude-code")`; if that fails, notify the error and change nothing.
  `off`: restore the remembered theme, then uninstall. `off` for a theme that failed to
  install must not throw.

## 5. README requirements
What it is (presentation only), how to install the theme file and enable the extension,
what each visual element is, the exact animation timing, that the extension yields to
another extension's editor (for example a Vim-mode editor) rather than fighting it, and
that it is inert while any other theme is active.

## 6. Tests (`test.mjs`, against the installed Pi; no network, no model)
- Theme loads and validates against the shipped schema; assert the exact hex values from
  §3 for the palette entries that are specified.
- Colour roles resolve: every `colors` entry naming a `vars` key resolves to the expected
  value, and `thinking*` roles escalate in the stated order.
- Mascot render: at frame 0 the purple shimmer is at the right edge; the band moves left
  and disappears during the pause; with effort below `max` the body is orange and the
  sparkle area is blank spaces; sparkle appears only at `max`.
- Header: mascot prefixes all three lines at `width >= 40` and never appears below it;
  the cwd renders as `~` when in the home directory.
- Editor: `render()` never throws across widths from 6 to 120, the `❯ ` glyph appears only
  when the theme is active, `borderColor` is restored after render, and the top-border
  effort label is suppressed while not idle or when lines are hidden.
- Footer: left and right sections both present, at least one space between them, output
  truncated to width.
- Command: `/claude-code off` on an active theme restores the previous theme; `/claude-code
  on` with the theme absent notifies and leaves the theme unchanged.
- Isolation: installing a fake foreign editor first, then running `install`, must leave
  that editor in place.
- Point `PI_CODING_AGENT_DIR` at a temporary directory so tests never touch real user
  state.

## 7. Acceptance criteria
- With the theme selected, Pi shows the mascot header, the `❯` prompt, the footer status
  bar, and the animated shimmer at `max` effort — and nothing of that appears under any
  other theme.
- With another extension's editor installed (e.g. a Vim-mode prompt), `/claude-code on`
  changes colours and header/footer but does not replace or break that editor.
- No timer, header, footer, or editor override survives `session_shutdown` plus a theme
  switch to a non-`claude-code` theme.
- The package performs no network access and no model calls, and reads/writes no files
  outside Pi's own theme and config directories.

Work in small commits. After each section, run the tests you have so far and report
status. If a requirement cannot be met against the installed Pi, stop and say so instead
of silently substituting a weaker behaviour.

===END PROMPT===

---

## Notes for whoever hands this over
- §4.1 and §4.4 are the parts that matter. A colour-only theme renders fine but loses all
  the chrome; an extension that installs its editor unconditionally silently breaks a
  Vim-mode editor installed by another extension, and that failure is invisible until
  someone presses Escape.
- Ask the agent to name the `pi` version it validated against and to paste the schema
  file it validated the theme against. Those two facts are the maintenance contract.
