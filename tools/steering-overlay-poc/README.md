# Too Many Assumptions <br/> Structured Steering For Coding Agents

![Dynamic structured steering over a real ArchiveBox validation session](screenshots/archivebox-validation-session.png)

This is a standalone macOS proof of concept for a model-generated, interactive Codex status
surface. It does not patch or inject code into Codex Desktop.

The app:

- switches to the foreground Codex session in ChatGPT/Codex Desktop or iTerm;
- keeps independent controls and event history for every session;
- extracts a bounded transcript containing only user and assistant messages;
- asks an ephemeral, read-only `gpt-5.6-luna` observer for current and recent implicit assumptions;
- displays them in an always-on-top panel at the bottom-right of the screen;
- refreshes after new conversation messages arrive; and
- persists every control change into the thread's steering state.

The observer output is constrained by `status-surface.schema.json`. There is no HTML or arbitrary
code in the model-defined UI.

The header uses the session's first-line title verbatim and shows the Git repository name, or the
working-directory basename, beside it in muted text. Titles are never ellipsized: the observer aims
for short action labels and the native UI wraps longer session or control titles when needed.

A compact horizontal tab strip is derived by scanning the existing per-thread state directories.
Each tab shows the first ten characters of its session title and uses a deterministic FNV-1a hash
of its thread ID for a subtle, stable, per-session color bar. Clicking a tab reads and writes that
thread's canonical files; foreground detection changes only the selected tab inside the existing
floating panel.

## Run it

Run it once; it follows the foreground session automatically:

```bash
tools/steering-overlay-poc/run.sh
```

The launcher owns both the observer and overlay under one PID file; starting it again replaces the
previous pair instead of allowing multiple writers to alternate the same session state.

Or target a task explicitly:

```bash
tools/steering-overlay-poc/run.sh --thread 019fefe0-76d4-7e92-aad2-cb01eeab1107
```

To exercise the UI without making a model request:

```bash
tools/steering-overlay-poc/run.sh --demo
```

Changing a control updates the panel immediately, appends an event under
`.build/steering-overlay/threads/<thread-id>/events.jsonl`, and updates the thread's canonical
state directly.

Each control has a fixed-width right-side action column backed by one binary `salience` integer.
An unpinned row reveals a gray check on hover; clicking it sets `+1` and replaces it with an
always-visible yellow pin. Clicking the pin restores `0` (neutral and recalculable). Delete appears
only on hover, removes the item, and records a suppression event so the observer does not
immediately recreate it.
Changing a toggle, choice, or slider automatically sets salience to `+1`, pinning the selected
answer in the same atomic write. Hovering a row reveals a pencil beside its description; editing
either text field and pressing Return updates both strings and pins the customized control so the
same wording reaches the next hook injection. Choice rows also reveal a plus button that accepts,
selects, and pins a custom option on Return.
Repeated changes produce new revisions, while the hook signature prevents automatic reinjection
loops. Sticky controls remain in an unlabeled upper section, separated from neutral controls by a
small centered marker. Both sections are derived from the canonical controls in `state.json`.

The cached surface is also a tiny goal-like model tool. It reads the foreground thread by default:

```bash
python3 tools/steering-overlay-poc/observer.py --get
python3 tools/steering-overlay-poc/observer.py --set test_scope comprehensive --expected-revision 3
```

Both UI and agent edits update the thread's canonical `state.json`, increment its revision, append
a provenance-tagged event, and live-update the overlay. An in-flight observer result is discarded
if it was generated from an older revision.

The workspace hook in `.codex/hooks.json` reads that same state automatically. It injects a full
snapshot on session start or resume, then injects only when the semantic control state changes at
a prompt or tool boundary. Codex requires a one-time trust review for a new workspace hook; use
`/hooks` and approve this definition, then start or resume the thread. The injected snapshot also
contains the exact revision-checked update command available to the agent.

Foreground detection is intentionally pragmatic: iTerm maps its current TTY to
`CODEX_THREAD_ID`; ChatGPT/Codex Desktop uses the most recently active or updated row in Codex's
local thread database. `active.json` is only a pointer; each session's generated state remains in
its own directory. Use `--thread ID` to pin a session while debugging.

The observer re-evaluates the surface only after a new user or assistant message. Foreground
switches load the session's cached surface without a model call. It preserves the exact prior
controls by default and makes a minimal update only when new evidence changes an assumption or the
task phase. The popup shows a live list of current and recent consequential assumptions the agent
made but the user did not explicitly specify. Explicit decisions stay out unless the user has
reversed the same preference at least three times.

The PoC does not mutate rollout JSONL or synthesize user messages. Codex's hook context carries
the current state directly into model context.
