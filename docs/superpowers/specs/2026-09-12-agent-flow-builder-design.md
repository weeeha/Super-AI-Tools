# Agent Flow Builder, design spec

_2026-09-12 · status: draft for Nick's review · repo: weeeha/Super-AI-Tools · local path: `~/ClaudeCode Projects/Super AI Tools`_

## Summary

A node canvas for Nick's agent loops and workflows, in the shape of OpenAI's Agent Builder: a palette of agent-workflow node kinds (Start, Agent, Classify, If/else, While, User approval, MCP, Guardrails, File search, Transform, Set state, Note, End), a dot-grid canvas, sticky notes, an inspector, undo/redo, and a chrome-free view mode for sharing.

Three waves:

1. **Draw** (this spec): hand-authored flows saved as JSON, edited in the browser, shared as unlisted Vercel pages.
2. **Read**: adapters that generate flows from real agents (Claude Code skills and plugins, OpenClaw agents).
3. **Run**: stubbed execution that lights nodes up through the existing `useFlowRunner`.

This document fully specifies wave 1 and fixes the two contracts waves 2 and 3 depend on: the Flow JSON schema and the node-kind catalog. Waves 2 and 3 get their own brainstorm and spec when they start.

## Decisions (locked 2026-09-12)

| Decision | Choice |
| --- | --- |
| Roadmap | 1 Draw, then 2 Read, then 3 Run |
| Home | A standalone Next.js app in Super-AI-Tools. The design system (VV-DSGN-INC/Super-AI-Components) cut node-builder UI under decision D9 and dropped `@xyflow/react`, so the builder lives here and consumes Flow Kit rather than living inside it |
| Flow Kit reuse | Vendor (copy) `flow-types`, `flow-tokens.css`, `typed-handle`, `typed-edge` and their tests from `flow-kit-worktree` at commit `b414ac9` into `components/flow-kit/`, each file carrying a provenance header. Wave 3 vendors `node-status`, `use-flow-runner`, `run-button` the same way |
| Persistence | Flows are JSON files in `flows/`, committed and statically built. Browser edits are drafts in localStorage. Export and import JSON. No database, no auth |
| Loops | `while` is a container (React Flow group node) holding its body. The graph never contains cyclic edges |
| Look | shadcn tokens (style `base-nova`, base color neutral), light and dark, OpenAI-style compact nodes |
| Project mode | Personal exploration (new-project light path). Every deploy is an unlisted Vercel preview until Nick says otherwise |

## Reference

OpenAI Agent Builder, screenshot taken 2026-09-12:

- Left palette in groups: Core (Agent, Classify, End, Note), Tools (File search, Guardrails, MCP), Logic (If/else, While, User approval), Data (Transform, Set state). A Start node opens every flow.
- Condition node lists one row per expression (`input.output_parsed.has_all_details == true`) plus an Else row, each row with its own output port.
- Agent nodes: icon tile, name, "Agent" subtitle, output port on the right edge.
- Yellow sticky notes sit behind the graph.
- Top bar: back, flow name, Draft badge, edit and preview toggle, overflow, settings, Evaluate, Code.
- Bottom-center toolbar: hand, select, undo, redo.

Earlier work this builds on: the Flow Kit spec in `AI Components/docs/superpowers/specs/2026-06-11-flow-kit-design.md`, the Flow Kit inventory, and the Agent Builder spec's Flows section (read-only per-agent graph with a slide-over), which wave 2 revisits.

## Stack

Pinned to match the Flow Kit worktree so vendored sources compile unchanged.

| Package | Version |
| --- | --- |
| next | 16.2.9 |
| react, react-dom | 19.2.4 |
| @xyflow/react | ^12.11.0 |
| @base-ui/react (via shadcn base-nova) | ^1.5.0 |
| tailwindcss, @tailwindcss/postcss | ^4 |
| zustand | latest 5.x |
| zundo (temporal undo for zustand) | latest |
| zod | latest 4.x |
| lucide-react | ^1.17.0 |
| vitest, @testing-library/react, jsdom | vitest ^4.1 |
| @playwright/test | ^1.60 |
| typescript | ^5 |

Package manager pnpm. Node 24 in CI (Vercel default).

## Wave 1, Draw

### Routes

| Route | Behavior |
| --- | --- |
| `/` | Flow index. Cards for committed flows (name, description, node count, updated date) plus a "Drafts in this browser" section read from localStorage, plus a New flow button that creates a draft with slug `draft-<6 random lowercase alphanumerics>` and opens it |
| `/flow/[slug]` | The builder. A server component reads `flows/<slug>.json` (`generateStaticParams` over the directory). For an unknown slug the client checks localStorage for a draft with that slug; if none, 404 |
| `/flow/[slug]?mode=view` | View mode, the share link. Palette, inspector and bottom toolbar hidden. Canvas locked: no drag, connect, or delete. `fitView` on load, pan and zoom and minimap only |

### Layout (edit mode)

- **Top bar:** back to index · flow name, inline editable · status badge (`Saved` when the store equals the committed file, `Draft` when localStorage differs, `Unsaved draft` for flows that exist only in the browser) · mode toggle (edit, view) · overflow menu with Export JSON (downloads `<slug>.json`), Import JSON (file picker or pasted text), Copy share link (the `?mode=view` URL of the current page), Revert to file, Delete draft.
- **Left floating palette:** grouped list as in the reference. Click adds the node at the viewport center, selected. Drag and drop places it at the cursor. Dropping onto a `while` frame parents the node to it.
- **Canvas:** React Flow with dot background, minimap bottom-right, 16px snap grid, selection on drag in select tool, pan on drag in hand tool.
- **Bottom-center toolbar:** hand, select, undo, redo.
- **Right inspector:** shown when exactly one node is selected. Kind-specific form (see catalog), then Duplicate and Delete. Multi-select shows the count and Delete. Nothing selected: hidden.
- **Keyboard:** Delete and Backspace remove the selection · ⌘Z and ⇧⌘Z undo and redo · ⌘D duplicate · Esc deselect · Space plus drag pans · V select tool · H hand tool.

### Node kinds (catalog)

The catalog lives in one file, `components/builder/node-catalog.ts`, and drives the palette, the inspector forms, default data, and the handle list. Adding a kind means adding one entry plus one card renderer.

| Kind | Group | Handles | Data (inspector fields) | Card |
| --- | --- | --- | --- | --- |
| `start` | core | out | `trigger`: manual, schedule, webhook, event · `triggerDetail?` (cron text, URL, event name) | Rounded pill "Start" with the trigger as subtitle |
| `agent` | core | in, out | `instructions` (textarea) · `model?` · `tools?: string[]` (chips) | Icon tile, label, "Agent" subtitle |
| `classify` | core | in, one out per category | `categories: string[]`, min 1 | Card listing categories, each a labeled port |
| `end` | core | in | `outputLabel?` | Rounded pill "End" |
| `note` | core | none | `text` · `color`: yellow, blue, green, pink | Sticky, resizable, rendered behind nodes, no ports |
| `file-search` | tools | in, out | `store` · `topK?` | Icon tile, label, store as subtitle |
| `guardrails` | tools | in, out `pass`, out `fail` | `checks: string[]` | Card listing checks, two ports Pass and Fail |
| `mcp` | tools | in, out | `server` · `tools?: string[]` | Icon tile, label, server as subtitle |
| `if-else` | logic | in, one out per condition, out `else` | `conditions: { label, expression }[]`, min 1 | One row per condition plus an Else row, matching the reference |
| `while` | logic | in, out | `condition` · `maxIterations?` | Group frame: header "While {condition}", body region with "Drop nodes here" empty state, resizable |
| `user-approval` | logic | in, out `approve`, out `reject` | `prompt` | Icon tile, label, prompt preview |
| `transform` | data | in, out | `expression` (textarea) | Icon tile, label |
| `set-state` | data | in, out | `assignments: { key, value }[]` | Icon tile, label, key list |

Every kind also carries `label` (defaults to the kind's display name) and `description?` (inspector only, shown as a tooltip on the card).

### Handles and connection rules

- All ports use the Flow Kit handle type `control`, registered at module scope with `registerHandleType("control", { label: "Control" })`. One data type is enough for wave 1; wave 3 may add `state`.
- Handle ids extend the Flow Kit codec with an optional port: `{nodeId}:control:in`, `{nodeId}:control:out`, `{nodeId}:control:out@{port}`. Port names: `c0`, `c1`, … and `else` for `if-else`; `cat-0`, `cat-1`, … for `classify`; `pass`, `fail`; `approve`, `reject`. The vendored `flow-types.ts` gains the `@port` suffix in `handleId` and `parseHandleId`; this is the one deliberate deviation from the copied source and is recorded in the provenance header.
- A connection is valid when: source direction is `out` and target is `in` · source and target are different nodes · both nodes share the same `parentId` (both top-level, or both inside the same `while` body). A `while` node's own handles are top-level ports; nothing crosses its boundary except through them.
- A target port accepts many edges. A source port allows many edges (fan-out). Execution semantics of fan-out belong to wave 3.
- Edges render with the vendored `typed-edge` and take the neutral `--flow-control` color. Edges leaving a named port show the port label (the condition label, category, Pass, Fail, Approve, Reject, Else) as an edge label near the source.

### Flow JSON schema (contract for waves 2 and 3)

```ts
type Flow = {
  schemaVersion: 1;
  id: string;                 // slug, [a-z0-9-]+
  name: string;
  description?: string;
  source: "manual" | "claude-code" | "openclaw";
  sourceRef?: string;         // path or agent id the flow was generated from
  createdAt: string;          // ISO 8601
  updatedAt: string;
  nodes: FlowNode[];
  edges: FlowEdge[];
  viewport?: { x: number; y: number; zoom: number };
};

type FlowNode = {
  id: string;
  kind: NodeKind;             // the 13 kinds above
  position: { x: number; y: number };   // relative to parent when parentId is set
  parentId?: string;          // only a `while` node may be a parent
  width?: number;             // note and while only
  height?: number;
  data: KindData[kind] & { label: string; description?: string };
};

type FlowEdge = {
  id: string;
  source: string;
  target: string;
  sourceHandle: string;       // full handle id, port included
  targetHandle: string;
  label?: string;
};
```

- `lib/flow-schema.ts` holds the zod schema: a discriminated union on `kind`. `parseFlow(json)` returns either a typed `Flow` or a list of readable errors. Unknown kinds fail loudly; nothing is silently dropped.
- `migrateFlow(json)` dispatches on `schemaVersion`. Only version 1 exists today; the function exists so waves 2 and 3 can add versions without touching callers.
- Node positions are stored exactly as React Flow holds them (child positions relative to the parent) so export and import round-trip byte-for-byte after key ordering.

### State

- `lib/flow-store.ts`: one zustand store wrapped in `zundo` temporal middleware. State: `nodes`, `edges`, `selection`, `mode` (edit, view), `tool` (select, hand), `dirty`. Actions: `addNode(kind, position, parentId?)`, `updateNodeData(id, patch)`, `applyNodeChanges`, `applyEdgeChanges`, `connect(connection)`, `removeSelection()`, `duplicateSelection()`, `setParent(id, parentId)`, `importFlow(flow)`, `exportFlow(): Flow`, `resetToFile()`.
- Undo history capped at 100 entries. Drag moves coalesce into one entry per drag end. Selection changes are not history entries.
- Drafts: a 500ms debounced subscriber writes `exportFlow()` to localStorage under `sat:flow:<slug>`. The index page lists keys with that prefix. `Saved` means a structural equality between the store's export and the committed file.
- Components never read storage directly; the store owns persistence through one subscriber.

### Seed flows

Three committed flows, each derived from a source Nick already wrote, so nothing is invented. The plan's seed task reads the source file and draws it; between them the three exercise all 13 kinds.

| Slug | Source | Kinds it must exercise |
| --- | --- | --- |
| `ship` | `~/.claude/me-skills/skills/ship/SKILL.md` | start (manual), if-else (clean tree, on main, PR exists), agent (commit, deploy), mcp for the `gh` and `vercel` CLI steps (push, PR create or update, preview deploy), end (link) |
| `new-project` | `~/.claude/me-skills/skills/new-project/SKILL.md` | start, classify (personal, client, promotion), file-search (the design-practice-kit template and PLAYBOOK the skill reads), agent per step, user-approval (kickoff scope commitments), set-state (identity map), note (guardrails), end |
| `subagent-build-loop` | `~/.claude/plugins/cache/claude-plugins-official/superpowers/6.3.0/skills/subagent-driven-development/SKILL.md` | start, while (tasks remain) containing agent (implementer), agent (spec reviewer), agent (quality reviewer), if-else (review passed), transform (task context handed to the next implementer), guardrails (the skill's stop conditions), user-approval (finish branch), end |

Coverage check: the three seeds together use all 13 kinds (`ship`: start, if-else, agent, mcp, end · `new-project`: classify, file-search, user-approval, set-state, note · `subagent-build-loop`: while, transform, guardrails).

### Visual language

- Tokens: shadcn neutral set plus, in `components/flow-kit/flow-tokens.css`: `--flow-control` (edges and ports, maps to `--muted-foreground`), group accents `--flow-core`, `--flow-tools`, `--flow-logic`, `--flow-data` for icon tiles, and `--flow-note-yellow`, `--flow-note-blue`, `--flow-note-green`, `--flow-note-pink` for sticky fills. Light and dark values for each.
- Node card: 260px wide, 16px radius, `--card` background, 1px `--border`, selected state 2px `--ring` plus a soft shadow. Icon tile 36px with the group accent as background. Title 15px medium, subtitle 12px `--muted-foreground`.
- Ports: 10px circles centered on the card edge in `--flow-control`, scaling up on hover and while a connection drags.
- `while` frame: 2px dashed `--border`, header strip with the condition, body region tinted `--muted` at low opacity.
- Sticky note: group accent removed, fill from the note token, 8px radius, 13px text, resize handle bottom-right.
- No raw colors in component sources. The `check:tokens` script is ported from Super-AI-Components and runs in CI.

### Repo layout

```
app/
  layout.tsx · globals.css            shadcn base-nova neutral + flow tokens import
  page.tsx                            flow index
  flow/[slug]/page.tsx                server: read flows/<slug>.json, render <FlowBuilder/>
components/
  ui/                                 shadcn primitives (button, dropdown-menu, input, textarea, tooltip, badge)
  flow-kit/                           vendored from flow-kit-worktree@b414ac9 (+ tests)
    flow-types.ts · flow-tokens.css · typed-handle.tsx · typed-edge.tsx
  builder/
    flow-builder.tsx                  shell composing the parts below
    flow-canvas.tsx                   <ReactFlow/> wrapper: nodeTypes, edgeTypes, drop, isValidConnection
    node-palette.tsx · node-inspector.tsx · top-bar.tsx · canvas-toolbar.tsx
    node-catalog.ts                   the 13 kinds: label, group, icon, handles, defaults, inspector fields
    nodes/
      kind-node.tsx                   icon tile + label + subtitle + ports (agent, mcp, file-search, transform, set-state, user-approval)
      terminal-node.tsx               start and end pills
      branch-node.tsx                 if-else, classify, guardrails (rows with a port each)
      while-node.tsx                  group frame with NodeResizer
      note-node.tsx                   sticky with NodeResizer
lib/
  flow-schema.ts · flow-store.ts · flows.server.ts (read flows/ at build) · storage.ts (localStorage adapter) · utils.ts
flows/
  ship.json · new-project.json · subagent-build-loop.json
scripts/check-tokens.mts
e2e/builder.spec.ts
docs/superpowers/specs · docs/superpowers/plans
```

### Conventions

Inherited from Super-AI-Components so the two codebases read the same: `"use client"` first line on interactive files · files kebab-case, exports PascalCase · `data-slot` on every rendered part · `className` passthrough through `cn()` · controlled components with `on*` callbacks · no data fetching and no storage access inside components · status vocabulary, when wave 3 needs it, is the Flow Kit contract `idle | queued | streaming | done | failed | locked`.

### Error handling

- Invalid committed JSON fails the build with the zod error list, never a blank page.
- Invalid imported JSON shows the error list in a dialog and leaves the current flow untouched.
- A localStorage write failure (private mode, quota) flips the badge to `Not saving` with a tooltip; the app keeps working in memory.
- An invalid connection attempt is refused silently by React Flow's validator; the target port shows an invalid state while hovering.

### Testing

- **Vitest + RTL, colocated:** flow-schema (valid, invalid, unknown kind, round-trip) · store (add, connect, remove, duplicate, undo, redo, drag coalescing, parent constraint, import and export equality) · connection validator matrix (direction, self, cross-boundary) · each node renderer shows its data and the right number of ports · inspector edits reach the store · vendored Flow Kit tests keep passing after the port-suffix change.
- **Playwright, Chromium and WebKit:** open `/flow/ship` and assert the seed nodes · add an Agent from the palette · connect it · rename it in the inspector · export JSON and assert it contains the new node · reload and assert the draft restored · open `?mode=view` and assert palette and inspector are absent and the canvas is locked.
- **CI (GitHub Actions):** lint, typecheck, check:tokens, test, build, e2e. Node 24.

### Non-goals (wave 1)

Running flows · live agent data · auto-layout · an AI omnibar · code export · collaboration and comments · auth · cloud saves · mobile layout · palette search · edge waypoints · copy and paste between flows.

### Success criteria

1. `/` lists the three seed flows and opens each; every seed renders its nodes, edges and notes exactly as committed.
2. From the palette a user adds a node, connects it, renames it in the inspector, exports JSON, imports that JSON into a new draft, and gets an identical canvas.
3. A node dropped into a `while` frame moves with the frame and cannot connect to nodes outside it.
4. Undo and redo cover every store action, with one entry per drag.
5. `?mode=view` on any flow shows only the locked canvas, fitted, in both light and dark.
6. Vercel preview URL renders in Chrome and Safari with no console errors.
7. All Vitest and Playwright suites pass in CI.

## Wave 2, Read (sketch; contracts only)

- Adapters emit Flow JSON with `source` set to `claude-code` or `openclaw` and `sourceRef` set to the file path or agent id. Positions come from an auto-layout pass (dagre, left to right) at generation time, so the builder itself stays layout-free. Generated flows are committed into `flows/` as snapshots; the deployed site stays static.
- `claude-code` adapter, a local CLI (`pnpm flows:import claude-code <path>`): reads a SKILL.md (frontmatter, numbered steps, sentences beginning with "If") into start, one agent per step, if-else per branch sentence, user-approval where a step waits for Nick; plugin agents and hooks become agent and guardrails nodes.
- `openclaw` adapter: OpenClaw stores agent config in sqlite (`~/.openclaw/agents/<id>/agent/openclaw-agent.sqlite`) and its flow registry in `~/.openclaw/flows/registry.sqlite*`, so the adapter reads through the OpenClaw CLI or gateway JSON where one exists rather than raw sqlite. An agent becomes start (trigger from its cron), agent, mcp and file-search for its tools, end; sub-agents become agent nodes with a handoff edge.
- Questions for the wave-2 brainstorm: which skills count as workflows, refresh cadence, and whether generated flows are read-only or fork into manual copies.

## Wave 3, Run (sketch)

- Vendor `node-status`, `use-flow-runner`, `run-button`. The runner treats a `while` group as one runner node whose `execute` iterates its body up to `maxIterations`, so the outer graph stays acyclic and the existing cycle detection holds.
- Executors: a stub executor by default (canned outputs, simulated latency, a scripted failure flag); agent nodes call the Vercel AI Gateway when a key is present; conditions run through a safe expression evaluator over a `state` object, never `eval`; user-approval pauses the run behind an Approve and Reject prompt; guardrails run their checks as predicate stubs.
- Chrome: Run flow, Stop, Run from node, a run log panel, status ring and badge on cards, streaming edge animation.
- Candidate, not committed: a Code button exporting a Claude Agent SDK TypeScript skeleton from the flow.

## Risks and mitigations

- **React Flow group nodes** (parent extent, z-index, relative positions, drop-into-parent) are the fiddliest part of wave 1. The plan starts with a spike task that proves drop-into-frame and move-with-frame before any other node work.
- **Draft-only persistence** can surprise. The status badge is always visible and the overflow menu leads with Export.
- **Vendored code drifts** from Flow Kit. The provenance header names the commit; a `scripts/flow-kit-diff.sh` compares the vendored files against the worktree so drift is a choice, not an accident.
- **shadcn base-nova and Base UI**: the vendored Flow Kit files use neither, so wave 1's exposure to the Base UI wrapper contracts (the SAI-02 finding in the design-system review) is limited to the inspector's own inputs.
