# pi-extension-prompts

Build prompts for [Pi](https://github.com/badlogic/pi-mono) extensions and skills.

Each file is a **specification** meant to be handed to a coding agent, not code.
The agent builds the package from scratch; the prompt is written so that the
result stays reviewable and auditable.

## Prompts

| Prompt | What it builds |
|---|---|
| [`prompts/youtube-summary.md`](prompts/youtube-summary.md) | A Pi package that turns a YouTube URL into a summary of the video's full transcript. Five-step retrieval ladder (cache → public captions → browser transcript panel → audio download + local STT → browser audio capture), JSONL subprocess protocol, and tests that enforce its own security claims. |
| [`prompts/pi-vim.md`](prompts/pi-vim.md) | Vim-style modal editing for Pi's prompt editor, as an extension that decorates the editor Pi already has. Splits a pure, unit-testable NORMAL-mode engine from a thin controller that drives the real editor through its TS-accessible ABI, preserving folded pastes, undo grouping, and dot-repeat. |
| [`prompts/claude-code-theme.md`](prompts/claude-code-theme.md) | A Claude Code-style terminal presentation for Pi: a theme file plus a presentation-only extension adding an animated mascot header, a `❯` prompt, a footer status bar, and an effort label on the editor border that yields to another extension's editor. |

## How these prompts are written

They follow a few rules that are worth keeping when adding new ones:

- **Specification, not source.** State the contract, the file layout, and the
  acceptance criteria. Never say "copy this package" — say what it must do.
- **No unreviewed dependencies.** The agent builds in-repo and never pulls from a
  public registry at runtime.
- **Security claims must be testable.** "Never reads cookies" is worthless; an
  assertion that the injected JavaScript contains no `document.cookie` is not.
- **No silent degradation.** Anything unsupported must fail with a stable error
  code and a hint, never fall back to a weaker behaviour without saying so.
- **Explicit `[FILL]` slots.** Environment-specific values (OS, endpoints,
  allowed egress) are left as placeholders so the same spec adapts to different
  deployments instead of guessing.
- **Verifiable deliverables.** Each prompt ships a privacy note and a one-page
  verification checklist that a reviewer can run.

## Using a prompt

1. Copy everything between the `===BEGIN PROMPT===` markers.
2. Fill the `[FILL]` items for your environment.
3. Hand it to your agent: "Read this and implement it. Report your plan first,
   then work in small commits."
4. Run the acceptance criteria before trusting the result.

## Adding a prompt

One Markdown file per prompt, named after the extension it builds. Keep the same
shape: environment, goal, file layout, contracts, hard rules, tests, acceptance
criteria, deliverables, and a short admin checklist at the bottom.

## License

MIT — see [LICENSE](LICENSE).
