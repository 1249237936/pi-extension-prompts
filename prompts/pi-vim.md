# BUILD PROMPT — Rebuild "pi-vim" (Vim-style prompt editing for Pi)

Copy everything between the `===BEGIN PROMPT===` markers into your Pi agent.
Fill the `[FILL]` items first. The prompt is self-contained: the agent has never seen
the original extension and must not need it.

===BEGIN PROMPT===

You are a senior engineer. Build a complete, working Pi extension from scratch. Do not
search for an existing extension and do not install anything from the public npm
registry — this must be written in-repo and reviewable.

Everything you need is in this prompt. Do not go looking for another project to copy
from. In particular: do not consult, read, vendor, or reference the source code of any
other commercial CLI product, and do not read any leaked or published copy of such
source. Derive the behaviour from the specification below and from Pi's own installed
types. The result must be a self-contained extension with no dependency on any
third-party repository.

## 0. Environment
- Pi version: [FILL: the `pi` version you are building against]. Read the installed
  package's own TypeScript types under `@earendil-works/pi-coding-agent` and
  `@earendil-works/pi-tui` before writing code — do not guess an API.
- This extension touches Pi's editor internals. It must therefore be verified against
  the ACTUAL installed Pi, not only against unit tests. See §6 and §8.
- No network access and no model calls are needed by this extension at any point.

## 1. Goal
Add Vim-style modal editing to Pi's prompt editor, implemented as an extension that
decorates the editor Pi already has. It must not replace Pi's editor, must not change
how the agent edits files or runs tools, and must preserve every existing editor
behaviour: Enter submits, slash commands, autocomplete, history, mouse, paste, IME
cursor handling, and Pi's abort/tree Escape action.

Two layers, strictly separated:
- a pure NORMAL-mode engine with no TUI or editor dependency, fully unit-testable;
- a thin controller that drives the real editor through its documented and
  TS-accessible API.

## 2. Package layout (exact)
    pi-vim/
      package.json          # name pi-vim-prompt, private, "keywords": ["pi-package"],
                            # pi.extensions: ["./index.ts"], peerDependencies on
                            # @earendil-works/pi-coding-agent and @earendil-works/pi-tui
      README.md             # what it is, how to load, key map, known limits
      LICENSE
      index.ts              # extension entry: lifecycle, /vim command, preference file
      engine.ts             # pure NORMAL-mode core + cursor helpers
      editor.ts             # VimController: attaches to CustomEditor, key routing
      engine.test.ts        # pure engine tests (no TUI)
      test.mjs              # integration tests against the installed Pi loader

## 3. Files and contracts

### 3.1 engine.ts — pure core, no editor import
Export:
- `interface VimBuffer { text; cursor; atoms: {start,end}[]; expand(s): string }`
- `interface VimResult { text?; cursor?; action?: string; count?; insert?: boolean; keys?: string[] }`
- `normalCursor(buffer, cursor?)` — snap a cursor onto a valid NORMAL-mode cell
- `previousCursor(buffer, cursor?)` — one grapheme left, never crossing to the previous line
- `class VimEngine { handle(key: string, buffer: VimBuffer): VimResult | undefined; reset(): void; get pending(): string }`

Required behaviour:
- Keys are literal characters. `f/F/t/T/r` consume exactly one grapheme per `handle()` call.
- Counts multiply across operator and motion and **saturate at 10_000**.
- Escape (or `\x1b`), or `escape`/`esc`, cancels pending input but KEEPS the unnamed
  register and the last `f/F/t/T` find.
- Internal positions are grapheme cells even though the public offsets are UTF-16.
  Use `Intl.Segmenter` with `granularity: "grapheme"`.
- A paste atom (see §3.2) counts as exactly one motion cell; deleting across it removes
  the whole marker.
- `u` and `.` must NOT mutate text themselves. They return an action signal with a count;
  the controller performs the undo/redo/replay. Motions and yanks return no change signal.
- Text-mutating commands carry the full key sequence (including counts) so the controller
  can build dot-repeat history.
- Invalid continuations are consumed, never reinterpreted as a new command.
- Failed motions and unknown commands are safe no-ops.

Command inventory: motions `h j k l w b e W B E 0 ^ $ gg G f F t T ; ,`; edits
`d c y` + motion or text object, `dd cc yy x X D C S s r J p P`, plus `Y ~ >> <<`;
text objects `iw aw iW aW` and inner/around quotes, `() [] {} <>` (with `b/B` aliases);
undo/redo/repeat `u`, `Ctrl+R`, `.`.

Deliberate limits, stated in code comments and README: no visual mode, no `:` ex
commands, no search (`/` belongs to Pi's slash commands), no marks, macros, named
registers, mappings, screen-row motions, syntax-aware matching, autoindent, or Vim
options. Words are Unicode letters/digits/marks/underscore versus punctuation. Columns
are grapheme columns, not terminal display widths. Bracket objects require an enclosing
pair and select its literal contents. Quotes are line-local, honour backslash escapes,
and may seek the next pair.

### 3.2 editor.ts — VimController
Exported class `VimController` with, at minimum:
- `constructor(editor: CustomEditor, tui: TUI, kb: KeybindingsManager)`
- `get currentMode(): "insert" | "normal"`
- `get isEnabled(): boolean`
- `setEnabled(enabled: boolean): void`
- `dispose(): void`

Non-negotiable design rules:
1. **Decorate, do not replace.** Wrap the SAME editor instance Pi (or a previously
   installed extension) produced. Pi's factory switch copies raw `getText()` and thereby
   loses folded paste payloads, so before installing, materialise the draft through the
   public expanded-text API: read `ctx.ui.getEditorText()` and, if non-empty, call
   `ctx.ui.setEditorText(draft)`. Never call `setText(getText())` afterwards.
2. **Keep all reliance on Pi's TS-private editor ABI in one place.** Define an
   `interface EditorABI` and validate every field it needs in the constructor
   (`state.lines`, `pastes` as a Map, `undoStack.push/pop`, `cancelAutocomplete`,
   `exitHistoryBrowsing`, `pushUndoSnapshot`, `undo`, `submitValue`,
   `renderBottomBorder`, `editor.getCursor`, `editor.actionHandlers` as a Map).
   On any mismatch, throw a clear error; the caller reverts to the previous editor and
   notifies. Never fail silently.
3. **Restore on dispose.** Save and restore `handleInput`, `setText`,
   `insertTextAtCursor`, the ABI's `submitValue` and `renderBottomBorder`. Restore
   BEFORE invoking callbacks, because `/vim` can change or replace the editor.
4. **Paste atoms.** `[paste #N ...]` markers are opaque: keep `pastes` and `pasteCounter`
   in every snapshot, expand through `buffer.expand()` when a register or replay needs
   real text, and never let an edit split a marker.
5. **Undo grouping.** Track an insert group (`before`, `depth`, `start`, `keys`). On
   leaving INSERT, if the text actually changed, collapse the editor's intermediate undo
   snapshots down to the group's depth and push the pre-insert snapshot, so one `u`
   undoes the whole insert. Maintain a redo stack.
6. **Dot-repeat.** Record `{keys, patch?}` where `patch` is a minimal
   `{delta, remove, text, cursorDelta}` insertion diff computed over expanded text in
   grapheme units. Replay by re-running the keys with the engine, and when the replay
   reaches INSERT, apply the patch instead of re-typing. Replay uses expanded text, so
   clear `pastes` and `pasteCounter` afterwards.
7. **Never fight Pi.** `submitValue` resets mode to INSERT, clears pending input and the
   redo stack first, then calls through. `renderBottomBorder` appends a
   ` NORMAL <pending> ` label using `editor.borderColor`, and falls back to the plain
   border when the width is too small. `insertTextAtCursor` begins an insert group and
   clears redo. `setText` resets the prompt.
8. Reset events: `escape` closes an unfinished command first; it does NOT reach Pi's
   abort/double-Escape action unless nothing is pending. Autocomplete is dismissed on
   any state-changing edit.

### 3.3 index.ts — extension entry
- `export default function (pi: ExtensionAPI): void`
- Read the saved preference from `<agent dir>/vim.json` (`{"enabled": boolean}`) using
  `getAgentDir()`; missing file means enabled. Any other read error notifies as a warning
  and keeps the default. Write atomically: temp file with `mode: 0o600` then `rename`.
  Persisting the preference must never break the session — a write failure is a warning.
- Install on `session_start` only when `ctx.mode === "tui"` and enabled; `dispose()` the
  controller on `session_shutdown`. The install function must be idempotent, and must
  return `false` (reverting to the previous editor) instead of leaving a broken editor in
  place.
- Command `/vim` with args `[on|off|status|help]`. Empty arg toggles. `status` reports
  both the live state with the current mode AND the saved preference. `help` prints the
  key map. Loading the extension forces INSERT mode, so typing works immediately.

## 4. README requirements
State plainly: what it is (an extension, not an embedded Vim), how to load it (one of:
`pi -e ./pi-vim/index.ts`, `pi install ./pi-vim`, or a symlink into the extensions
directory — never two at once), that `/reload` is needed in a running session, the full
key map, the Escape behaviour, that `/` starts a Pi slash command rather than Vim search,
the preference file location, the deliberate limits from §3.1, and the compatibility
statement from §7.

## 5. Tests (engine tests run with no TUI, no network, no model)
`engine.test.ts` must cover at least:
- cursor helpers: clamping to grapheme starts, never trailing a nonempty line, never
  moving into the previous line;
- motions `hl`, `jk` (logical lines, no crossing into history), `wbe WBE`, `0 ^ $ gg G`
  with counts, `f F t T ; ,` staying inside the line;
- operators: `dw`/`d$` must not eat the newline; counts multiply; `dd yy cc` linewise
  register semantics; `cw` keeps following whitespace; `x X s r D C S J`; `y` sets no
  change and `p P` use the unnamed register; ungrouped `dd` then `p` uses the linewise
  flag rather than inferring from a trailing newline; failed motions are no-ops;
- text objects `iw aw iW aW` and brackets/quotes;
- undo/repeat signalling: `u` and `.` return actions and never mutate; motions and yanks
  produce no change; insert entries carry keys; mutating commands carry full keys counting
  counts;
- unicode and atoms: grapheme clusters stay intact, atoms behave as one cell,
  `expand()` is used for the register;
- pending grammar: incomplete input exposes pending and resets cleanly, Escape cancels
  pending but keeps register and find, invalid continuations are consumed.

## 6. Integration tests (`test.mjs`)
Drive the extension against the INSTALLED Pi: real loader, real `CustomEditor`, real
theme, real key parser. Assert that typing, Enter-to-submit, slash commands, autocomplete
dismissal, and paste handling still behave; that installing and disposing restores the
previous editor exactly; and that a deliberately incompatible editor ABI is rejected with
a notification and no broken state. Do not touch the user's real preference file — point
`PI_CODING_AGENT_DIR` at a temporary directory. Keep narrow-width rendering inside the
range the native editor supports (wide glyphs at width 1 recurse in some Pi versions).

## 7. Compatibility statement
Document the exact Pi version range you verified against, and what happens on a
mismatch: constructor validation fails, the controller is not installed, the previous
editor stays active, and the user sees a warning naming the missing ABI field. "Works
until Pi changes its editor internals" is not acceptable wording — the failure mode must
be explicit and non-destructive.

## 8. Acceptance criteria
- Loading with the installed Pi puts the editor in INSERT mode; typing works normally.
- Esc reaches NORMAL; `i/a/I/A/o/O` return to INSERT; Enter submits in both modes.
- `/vim off` restores the previous editor and persists; `/vim on` reinstalls it.
- One `u` undoes an entire insert session; `Ctrl+R` redoes; `.` repeats the last change
  including its count.
- Folded pastes survive edits, undo, redo, and replay without corruption.
- Pending commands render in the bottom border and are cleared by Escape.
- No test touches the network, a model, or the user's real configuration.

## 9. Deliverables
Ship the `README.md` from §4 and a `LICENSE` file. The extension is a clean-room
implementation from this specification: state that in its own words in the README — no
code was copied, no other product's source was consulted, and the package has no runtime
dependency outside Pi itself. When you finish, report the exact `pi` version you built
and tested against, and the exact commands a reader can run to reproduce your test
results.

Work in small commits. After each section, run the tests you have so far and report
status. If a requirement cannot be met against the installed Pi, stop and say so instead
of silently substituting a weaker behaviour.

===END PROMPT===

---

## Notes for whoever hands this over
- The value of this spec is §3.2. Every rule there exists because of a specific failure
  mode: replacing the editor loses folded pastes, `setText(getText())` destroys them,
  ungrouped undo makes `u` useless after an insert, and replaying INSERT by re-typing
  breaks on unicode. An agent that only reads §1 will produce something that works in the
  happy path and corrupts pastes.
- The prompt is self-contained on purpose: an agent needs nothing but Pi's installed
  types and this file. Ask it to report the exact `pi` version it validated against; that
  line is the maintenance contract.
