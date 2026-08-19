---
date: 2026-07-26
title: "A metro map for what the agent actually did"
description: "Codex rollouts are JSONL. Agent Run Map compresses them into a transit map: spine, subagents, failures, and the files left on disk. Local-first, macOS, no LLM in the graph."
tags:
  - SwiftUI
  - macOS
  - agents
  - Codex
  - visualization
---

A coding agent’s rollout is a transcript. Hundreds of tool calls, nested subagents, patches, tests, and a user turn every so often. The useful question is not “what did the model say.” It is “what was the execution.” That is a graph problem: a spine of intent, branches of work, and evidence that stays on disk until you ask for it.

Agent Run Map is a local-first macOS menu-bar app that turns Codex `rollout-*.jsonl` files under `~/.codex/sessions` into that graph, then draws it as a metro map. The repository is [`Cree0618/agent-run-map`](https://github.com/Cree0618/agent-run-map). Status is `0.1.0` source-beta: usable from `./scripts/build_and_run.sh`, no signed/notarized binary, MIT license.

I did not write Codex. OpenAI did. The rollout format is an evolving implementation detail, reverse-engineered here, not a stable public contract. This post is about the viewer and the parser in front of it.

This post focuses on that compression and the native UI. Claude Code logs, a Swift rewrite of the parser, and a Mac App Store build are related and out of scope. The schema document still says “Claude deferred.” There is no Swift unit-test target yet.

I explicitly mention “local-first” because a rollout can contain prompts, source excerpts, absolute paths, and credentials a tool printed. Normal browsing stays on the machine. The optional Better titles action is the only network path, and it is opt-in per session.

## The object being visualized

Every Codex session is a JSONL file. Each line is roughly `{timestamp, type, payload}`. Useful types: `session_meta`, `event_msg`, `response_item`, `compacted`. `turn_context` is metadata. Unknown types warn and skip. Tool names drift across Codex versions (`shell_command` / `exec` / `exec_command` all become `command`; `apply_patch` becomes `patch`). The adapter normalizes them.

The contract in `docs/codex-run-graph-schema.md` is layered on purpose:

```text
JSONL file
   → SessionRef     (enough to list in the menu bar)
   → Event[]        (normalized stream)
   → RunGraph       (nodes + a spine, for the UI)
```

Six design principles, written before the SwiftUI window:

1. **Deterministic first.** v1 uses no LLM. Same file → same graph.
2. **Compress aggressively.** Target ≤ 40 visible nodes for a multi-hour session, not hundreds of tool calls.
3. **Prefer author intent.** `update_plan` is the best free spine; otherwise user turns.
4. **True branches only when explicit.** `spawn_agent` / child rollouts are tree edges. Heuristic “the agent changed strategy” is a badge, not topology.
5. **Store refs, not blobs.** Nodes point at source events. Full tool output stays on disk until the inspector.
6. **Fail soft.** Unknown tools and compaction gaps become warnings. They do not crash the parse.

The current Python/Swift payload is a reduced form of that contract (`spine` plus node fields). Explicit edges, warning lists, and some event IDs are still target, not all emitted.

The parser is `scripts/codex_run_graph.py`, ~3,400 lines, no third-party runtime dependencies. Rollouts are local but still untrusted input. Hard caps:

```python
MAX_ROLLOUT_BYTES = 256 * 1024 * 1024
MAX_JSONL_LINE_CHARS = 8 * 1024 * 1024
MAX_ROLLOUT_RECORDS = 500_000
```

Oversized files raise `RolloutLimitError`. Tests cover that. Re-run 2026-08-19: `python3 -m unittest discover -s tests -v` → **41 passed** in 0.019 s.

## Design patterns

Compared with “tail the JSONL in a terminal,” the app is closer to a runtime view of the agent: how it planned, branched, failed, and what it left on disk.

### Pattern 1: Compress the window, not the session slogan

Inside a turn, consecutive tools of the same family collapse. Reads and searches become `Explored (N reads/searches)`. Patches become `Edited K files: a.swift, b.py +N`. Commands that look like tests (`pytest`, `cargo test`, `xcodebuild`, …) become verify stations. Subagents keep their own node and a child session id.

```python
def compress_window(events, start, end, parent_id, id_prefix, *, turn_id=None):
    # groups consecutive explore / edit / verify / command / subagent events
    # returns (top_level children of the turn, nested nodes under a cluster cap)
```

The ≤40-node target is a design goal in the schema, not a measured average over my personal sessions. I will not invent a compression ratio. The point of the cap is scannability. A metro map that still has 400 stations is a log.

Plan steps are the spine when Codex emits `update_plan`. Titles get rewritten mid-run; the parser aliases later wording onto the fullest spine snapshot so tools stay attached to the station that announced the work. Tests exist specifically for “rewritten plan titles still attach tools” and “unrelated rewrites do not false-alias.”

### Pattern 2: Draw a transit map, not a node-link blob

The default center view is `MetroMapView`. Layout is a **strict column grid**, not force-directed scatter:

| Column | X | What goes there |
|---|---:|---|
| Subagents | 260 | purple stations; left half of work if there are no subagents |
| Spine | 520 | goal, user turns, plan steps |
| Work / failures | 780 | explore / edit / verify / command, failed clusters |

All circles on a side share an X. Branch Y is centered on the parent with 72 pt spacing. Routes are orthogonal T-junctions only. Minimum gap on the spine is 160 pt. Time-scaled layout (on by default) adds 0.35 px per second of wall time, capped at 420, so a ten-minute stall is visible without exploding the canvas. Canvas width is 1,100. At most three activity stations per spine node per side; the rest cluster.

Station language is transit, not UML:

- hollow station — ordinary turn
- double-ring — evidence-backed pivot (recovered from failure, spawned subagents, explicit reversal)
- gray route ending in × — failed child cluster
- gray route ending hollow — explicit subagent
- lighter gray branches — explore / edit / verify / command inside a turn
- solid endpoints — session goal and final visible outcome

{{< responsive-image src="images/agent-metro.png" alt="Schematic three-column metro map for an agent run" maxWidth="720px" >}}

*Station names are invented for the figure. Real Codex rollouts are not published here.*




Click a station for the inspector. Context menus expose Resume, Fork, Fork from here, and Copy station handoff. Scroll-to-station uses invisible 1×1 anchors at `station.position`, not the button frames, because visual offsets would lie to `ScrollViewReader`.

Failure highlighting is a separate pass. `FailurePath.highlightedIDs` keeps failed nodes, their parent turn, the next spine station (recovery), and edits/verifies in the same turn. Everything else dims to 0.22 opacity. If nothing failed, the set is empty and the UI does not dim. Goal always stays as an anchor.

### Pattern 3: Disk as a third projection

The **Disk** view answers a different question: what did this leave behind? It is a chronological filmstrip of `apply_patch` touches (add / update / delete), grouped by time or by file, with hunks from retained tool I/O. Reveal opens the file in Finder.

The graph JSON carries a `disk` index built by the Python parser. Markdown export adds a **Files changed on disk** section. The Swift `ApplyPatchParser` re-parses hunks for the filmstrip and caps a body at 12,000 characters.

This is the same “file system as persistent memory” idea as a coding harness, pointed at the *result* of the run instead of the agent’s working context. The map is topology. The filmstrip is the working tree.

### Pattern 4: The app is a poller, the parser is a child process

The menu bar lists sessions. The main window is sidebar + metro/outline/disk + inspector. Closing the window leaves the menu-bar browser running. The app is intentionally unsandboxed so it can read `~/.codex` and launch Python.

Loading a map is:

```text
python3 codex_run_graph.py --list --json
python3 codex_run_graph.py <rollout.jsonl> --json
```

The script is bundled by Xcode. During development the app walks up from the source tree. `GraphScriptPath` / `AGENT_RUN_MAP_SCRIPT` override it.

Session list monitoring is one shared task, 2 seconds between refreshes, started from either the popover or the window. Live tail on the *open* rollout is tighter: 750 ms, but it only re-parses when a cheap filesystem fingerprint (size / mtime) changes. Idle sessions are not re-parsed in a loop.

```swift
try await Task.sleep(nanoseconds: 2_000_000_000)  // list
try await Task.sleep(nanoseconds: 750_000_000)    // open-rollout tail
```

Initial list is 30 rows; further pages are 50. Duplicate titles get Codex-style ` (2)` suffixes. In-progress rows keep a live indicator.

## Case study: actions that go back into the agent

A viewer that cannot resume the run is a museum. From the map header, inspector, or context menus:

| Action | Behavior |
|---|---|
| Resume | Preferred terminal, `codex resume <session_id>` in the session `cwd` |
| Fork | `codex fork` — new thread, full history |
| Fork from here | Experimental app-server `thread/fork` + `lastTurnId`; on failure, full fork + station Markdown on the clipboard |
| Copy station handoff | Scoped Markdown: prefix of the spine + focus evidence |
| Reveal rollout | Selects the JSONL in Finder |

Subagent stations target the **child** session id. Forks change conversation history only. The working tree is not rewound. That sentence is in the macOS README because it is the kind of thing people get wrong.

Terminal discovery: UserDefaults → `CODEX_CLI` → known install locations (`~/.local/bin`, npm/nvm/fnm/volta, Homebrew) → login-shell `command -v` → bare `codex`. Automatic mode uses Kitty if it is running, otherwise Terminal.app. Kitty launches Codex directly. Terminal.app uses a private temporary `.command` that closes on success and stays open on failure.

Better titles is optional and separate. One click, once per open session, runs:

```bash
codex exec -m gpt-5.4-mini --ephemeral --output-schema … -o titles.json - < prompt.txt
```

Same ChatGPT/Codex login as `codex login`. No extra API key. The subprocess starts in an isolated temp directory, ignores user config, disables local/web tools, uses low reasoning, and reads the prompt on stdin. Payload is sanitized station labels, kind/status, tool counts, and file basenames. Commands, summaries, session IDs, full paths, and tool I/O stay local. Cache: `~/Library/Application Support/AgentRunMap/title-cache/<session_id>.json`. Unchanged stations are not resent.

## What this is not

It is not a signed product. Source builds use the placeholder icon. Do not redistribute a locally built `.app` as an official binary.

It is not sandboxed. That is a privacy tradeoff, not an oversight. Resume and Fork launch an external process after an explicit click.

It is not a complete log. Tool I/O is capped for display and not automatically secret-redacted. Compaction seams are markers. Hidden reasoning is excluded from Markdown on purpose.

It is not multi-harness. Codex only. Session discovery covers active `~/.codex/sessions`, not every archived or remote source. New Codex versions may need adapters; the tests are synthetic JSONL, not a vendor compatibility suite.

The SwiftUI surface is large (`RunMapView.swift` is about 3,000 lines; `MetroMapView.swift` about 670). Graph compression stays in Python until the UI settles. Porting the parser into Swift is an option in the macOS README, not done.

## Implications

Agent traces are closer to distributed traces than to chat. The right UI is a map with a bounded number of stations, true branches, and a way to open the evidence. A raw JSONL dump trains you to skim. A metro spine trains you to ask where the run forked and where it recovered.

The interesting layer is the contract between parser and view. Deterministic compression, explicit topology, refs instead of blobs. An LLM can rename stations later. It should not be what decides whether two plan titles are the same node.

If I picked this up again, the missing pieces are the ones the beta notes already list: a Swift test target, a signed build, redaction, Claude (or a second adapter), archived sessions. The next useful visual is probably not a prettier map. It is a measured statement of how often the ≤40-node budget holds on real multi-hour rollouts — with those rollouts redacted.

Until then the honest demo is: open the menu bar, pick a session you own, and watch the agent’s last hour become a line with a few branches. That is the point of the thing.
