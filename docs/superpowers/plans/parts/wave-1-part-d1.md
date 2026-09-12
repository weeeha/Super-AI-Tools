### Task 11: Pages, flow-builder shell, flow-index, shortcuts

Deliverable: `app/page.tsx`, `app/flow/[slug]/page.tsx`, `components/builder/flow-builder.tsx`, `components/builder/flow-index.tsx`, `components/builder/use-builder-shortcuts.ts`, all tested, `pnpm build` green with an empty `flows/` directory. Built in dependency order: shortcuts, then the client shell that mounts them, then the index page's own component, then the two server pages that tie everything to routes.

#### Task 11a: Builder keyboard shortcuts

**Files:**
- Create: `components/builder/use-builder-shortcuts.ts`
- Test: `components/builder/use-builder-shortcuts.test.ts`

**Interfaces:**
- Consumes:
  ```ts
  // lib/flow-store.ts
  export function useFlowStore<T>(selector: (s: FlowState) => T): T;
  export function useFlowStoreApi(): FlowStore;
  export function useFlowTemporal(): { undo: () => void; redo: () => void; canUndo: boolean; canRedo: boolean };
  // FlowState.removeSelection(): void;
  // FlowState.duplicateSelection(): void;
  // FlowState.setTool(tool: Tool): void;
  // FlowState.applyNodeChanges(changes: NodeChange<BuilderNode>[]): void;
  ```
  plus `NodeChange` from `@xyflow/react` (package) and `BuilderNode` from `lib/flow-convert.ts` (Task 3).
- Produces:
  ```ts
  // components/builder/use-builder-shortcuts.ts
  export function useBuilderShortcuts(): void;   // Delete/Backspace removeSelection · Mod+Z undo · Shift+Mod+Z redo · Mod+D duplicate · Escape deselect · V select tool · H hand tool; ignored while focus is in input, textarea, select, or contenteditable; no-op in view mode
  ```
  Mounted by `flow-builder.tsx` in Task 11b.

Escape deselect is implemented as an `applyNodeChanges` batch of `{ type: "select", selected: false }` entries, one per currently selected node, rather than a dedicated store action, matching the skeleton's note that Escape "is implemented as an applyNodeChanges select:false batch."

- [ ] **Step 1: Write the failing test**

```ts
// components/builder/use-builder-shortcuts.test.ts
import { describe, expect, it } from "vitest";
import { renderHook } from "@testing-library/react";
import * as React from "react";
import { createFlowStore, FlowStoreProvider, type FlowStore } from "@/lib/flow-store";
import { fixtureFlow } from "@/lib/test-fixtures";
import { useBuilderShortcuts } from "./use-builder-shortcuts";

function wrapperFor(store: FlowStore) {
  return function Wrapper({ children }: { children: React.ReactNode }) {
    return React.createElement(FlowStoreProvider, { store }, children);
  };
}

function dispatchKey(key: string, opts: KeyboardEventInit = {}, target: EventTarget = window) {
  target.dispatchEvent(new KeyboardEvent("keydown", { key, bubbles: true, cancelable: true, ...opts }));
}

describe("useBuilderShortcuts", () => {
  it("Mod+Z undoes the last store action", () => {
    const store = createFlowStore(fixtureFlow());
    const id = store.getState().addNode("agent", { x: 0, y: 0 });
    renderHook(() => useBuilderShortcuts(), { wrapper: wrapperFor(store) });

    dispatchKey("z", { metaKey: true });

    expect(store.getState().nodes.some((n) => n.id === id)).toBe(false);
  });

  it("Delete removes the selected node", () => {
    const store = createFlowStore(fixtureFlow());
    const target = store.getState().nodes[0];
    store.setState((s) => ({ nodes: s.nodes.map((n) => ({ ...n, selected: n.id === target.id })) }));
    renderHook(() => useBuilderShortcuts(), { wrapper: wrapperFor(store) });

    dispatchKey("Delete");

    expect(store.getState().nodes.some((n) => n.id === target.id)).toBe(false);
  });

  it("ignores a keystroke dispatched from an input element", () => {
    const store = createFlowStore(fixtureFlow());
    const before = store.getState().nodes.length;
    const target = store.getState().nodes[0];
    store.setState((s) => ({ nodes: s.nodes.map((n) => ({ ...n, selected: n.id === target.id })) }));
    renderHook(() => useBuilderShortcuts(), { wrapper: wrapperFor(store) });
    const input = document.createElement("input");
    document.body.appendChild(input);

    dispatchKey("Delete", {}, input);
    input.remove();

    expect(store.getState().nodes.length).toBe(before);
  });

  it("does nothing when mode is view", () => {
    const store = createFlowStore(fixtureFlow(), { mode: "view" });
    const before = store.getState().nodes.length;
    const target = store.getState().nodes[0];
    store.setState((s) => ({ nodes: s.nodes.map((n) => ({ ...n, selected: n.id === target.id })) }));
    renderHook(() => useBuilderShortcuts(), { wrapper: wrapperFor(store) });

    dispatchKey("Delete");

    expect(store.getState().nodes.length).toBe(before);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test -- components/builder/use-builder-shortcuts.test.ts`
Expected: FAIL, `Cannot find module './use-builder-shortcuts'`.

- [ ] **Step 3: Implement the hook**

```ts
// components/builder/use-builder-shortcuts.ts
"use client";

import * as React from "react";
import type { NodeChange } from "@xyflow/react";
import { useFlowStore, useFlowStoreApi, useFlowTemporal } from "@/lib/flow-store";
import type { BuilderNode } from "@/lib/flow-convert";

const EDITABLE_TAGS = new Set(["INPUT", "TEXTAREA", "SELECT"]);

function isEditableTarget(target: EventTarget | null): boolean {
  if (!(target instanceof HTMLElement)) return false;
  return EDITABLE_TAGS.has(target.tagName) || target.isContentEditable;
}

export function useBuilderShortcuts(): void {
  const mode = useFlowStore((s) => s.mode);
  const storeApi = useFlowStoreApi();
  const { undo, redo } = useFlowTemporal();

  React.useEffect(() => {
    if (mode !== "edit") return;

    function onKeyDown(event: KeyboardEvent) {
      if (isEditableTarget(event.target)) return;
      const mod = event.metaKey || event.ctrlKey;
      const key = event.key.toLowerCase();

      if (event.key === "Delete" || event.key === "Backspace") {
        event.preventDefault();
        storeApi.getState().removeSelection();
        return;
      }
      if (mod && event.shiftKey && key === "z") {
        event.preventDefault();
        redo();
        return;
      }
      if (mod && key === "z") {
        event.preventDefault();
        undo();
        return;
      }
      if (mod && key === "d") {
        event.preventDefault();
        storeApi.getState().duplicateSelection();
        return;
      }
      if (event.key === "Escape") {
        event.preventDefault();
        const { nodes, applyNodeChanges } = storeApi.getState();
        const changes: NodeChange<BuilderNode>[] = nodes
          .filter((n) => n.selected)
          .map((n) => ({ id: n.id, type: "select", selected: false }));
        if (changes.length > 0) applyNodeChanges(changes);
        return;
      }
      if (key === "v") {
        storeApi.getState().setTool("select");
        return;
      }
      if (key === "h") {
        storeApi.getState().setTool("hand");
      }
    }

    window.addEventListener("keydown", onKeyDown);
    return () => window.removeEventListener("keydown", onKeyDown);
  }, [mode, storeApi, undo, redo]);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/use-builder-shortcuts.test.ts`
Expected: PASS, 4 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/use-builder-shortcuts.ts components/builder/use-builder-shortcuts.test.ts
git commit -m "$(cat <<'EOF'
feat(builder): add keyboard shortcut bindings

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
)"
```

---

#### Task 11b: Flow-builder client shell

**Files:**
- Create: `components/builder/flow-builder.tsx`
- Test: `components/builder/flow-builder.test.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // lib/flow-schema.ts
  export type Flow = { /* ...as in the skeleton... */ };

  // lib/storage.ts
  export function loadDraft(slug: string): Flow | null;

  // lib/draft-persistence.ts
  export function bindDraftPersistence(store: FlowStore, slug: string, opts?: { debounceMs?: number }): () => void;

  // lib/flow-store.ts
  export type Mode = "edit" | "view";
  export type FlowStore = StoreApi<FlowState> & { temporal: StoreApi<TemporalState<Partial<FlowState>>> };
  export function createFlowStore(initial: Flow, opts?: { file?: Flow | null; mode?: Mode }): FlowStore;
  export function FlowStoreProvider(props: { store: FlowStore; children: React.ReactNode }): JSX.Element;
  export function useFlowStore<T>(selector: (s: FlowState) => T): T;
  ```
  plus `useBuilderShortcuts` (Task 11a) and, from Part C: `TopBar(props: { slug: string; file: Flow | null })`, `NodePalette(props: { className?: string })`, `NodeInspector(props: { className?: string })`, `CanvasToolbar(props: { className?: string })`, `FlowCanvas(props: { className?: string; children?: React.ReactNode })`.
- Produces:
  ```ts
  // components/builder/flow-builder.tsx
  export function FlowBuilder(props: { slug: string; file: Flow | null; initialMode: Mode }): JSX.Element;
  // resolves initial flow = loadDraft(slug) ?? file; if both null renders <FlowNotFound slug/> (a link back to "/"); creates the store once (useState initializer), wraps in FlowStoreProvider, binds persistence in an effect (edit mode only), mounts useBuilderShortcuts, renders TopBar + NodePalette + FlowCanvas + CanvasToolbar + NodeInspector in edit mode and only FlowCanvas in view mode.
  ```
  Rendered by `app/flow/[slug]/page.tsx` in Task 11d.

Two points the skeleton states as outcomes but does not spell out as mechanism, settled here:

1. **Hydration safety.** The server renders from `file` alone, since it has no access to the browser's `localStorage`. Reading `loadDraft` during the initial render would make the client's first paint disagree with whatever the server sent down, and React reports that mismatch as an error. So the draft lookup happens only inside a `useEffect` (which never runs on the server and never runs during the render that produces the first client paint), and the component renders exactly what the server rendered until that effect resolves.
2. **A store built from nothing.** `createFlowStore` requires a real `Flow`, not `null`. When `file` is `null` (an unknown slug) there is nothing to build a store from until the mount effect checks `localStorage`, so the "pending" state renders nothing at all rather than attempting a store with no content. Once the effect resolves, a `<BuilderShell>` keyed differently from the pending one forces a fresh `useState` initializer, so a draft found after mount is what the store ends up holding, even when `file` was `null` throughout.

- [ ] **Step 1: Write the failing tests**

```tsx
// components/builder/flow-builder.test.tsx
import { describe, expect, it } from "vitest";
import { render, screen, waitFor } from "@testing-library/react";
import { saveDraft } from "@/lib/storage";
import { fixtureFlow } from "@/lib/test-fixtures";
import { FlowBuilder } from "./flow-builder";

describe("FlowBuilder", () => {
  it("renders FlowNotFound when there is no file and no draft", async () => {
    render(<FlowBuilder slug="ghost" file={null} initialMode="edit" />);
    expect(await screen.findByText("Flow not found")).toBeInTheDocument();
    expect(screen.getByRole("link", { name: "Back to flows" })).toHaveAttribute("href", "/");
  });

  it("renders the given file's nodes directly in edit mode when no draft exists", async () => {
    const flow = fixtureFlow();
    const { container } = render(<FlowBuilder slug={flow.id} file={flow} initialMode="edit" />);
    await waitFor(() => {
      expect(container.querySelectorAll('[data-slot="node"]')).toHaveLength(flow.nodes.length);
    });
    expect(container.querySelector('[data-slot="node-palette"]')).not.toBeNull();
  });

  it("renders a localStorage draft's nodes when file is null", async () => {
    const draft = fixtureFlow();
    saveDraft(draft);
    const { container } = render(<FlowBuilder slug={draft.id} file={null} initialMode="edit" />);
    await waitFor(() => {
      expect(container.querySelectorAll('[data-slot="node"]')).toHaveLength(draft.nodes.length);
    });
  });

  it("renders only the canvas and an Edit link in view mode, no palette or inspector", async () => {
    const flow = fixtureFlow();
    const { container } = render(<FlowBuilder slug={flow.id} file={flow} initialMode="view" />);
    await waitFor(() => {
      expect(container.querySelectorAll('[data-slot="node"]')).toHaveLength(flow.nodes.length);
    });
    expect(container.querySelector('[data-slot="node-palette"]')).toBeNull();
    expect(container.querySelector('[data-slot="inspector"]')).toBeNull();
    expect(screen.getByRole("link", { name: "Edit" })).toHaveAttribute("href", `/flow/${flow.id}`);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm test -- components/builder/flow-builder.test.tsx`
Expected: FAIL, `Cannot find module './flow-builder'`.

- [ ] **Step 3: Implement the shell**

```tsx
// components/builder/flow-builder.tsx
"use client";

import * as React from "react";
import Link from "next/link";
import type { Flow } from "@/lib/flow-schema";
import { loadDraft } from "@/lib/storage";
import { bindDraftPersistence } from "@/lib/draft-persistence";
import { createFlowStore, FlowStoreProvider, useFlowStore, type FlowStore, type Mode } from "@/lib/flow-store";
import { TopBar } from "@/components/builder/top-bar";
import { NodePalette } from "@/components/builder/node-palette";
import { NodeInspector } from "@/components/builder/node-inspector";
import { CanvasToolbar } from "@/components/builder/canvas-toolbar";
import { FlowCanvas } from "@/components/builder/flow-canvas";
import { useBuilderShortcuts } from "@/components/builder/use-builder-shortcuts";

function FlowNotFound({ slug }: { slug: string }) {
  return (
    <div className="flex h-dvh w-full items-center justify-center">
      <div
        data-slot="flow-not-found"
        className="flex flex-col items-center gap-3 rounded-xl border border-border bg-card p-8 text-center shadow-sm"
      >
        <p className="text-sm font-medium">Flow not found</p>
        <p className="text-xs text-muted-foreground">
          No committed or draft flow matches &quot;{slug}&quot;.
        </p>
        <Link href="/" className="text-sm underline underline-offset-4">
          Back to flows
        </Link>
      </div>
    </div>
  );
}

type Resolution = { status: "pending" } | { status: "resolved"; flow: Flow } | { status: "missing" };

function BuilderChrome({ slug, file }: { slug: string; file: Flow | null }) {
  useBuilderShortcuts();
  const mode = useFlowStore((s) => s.mode);

  return (
    <div data-slot="flow-builder" className="relative flex h-dvh w-full flex-col">
      <TopBar slug={slug} file={file} />
      <FlowCanvas className="flex-1">
        {mode === "edit" ? (
          <>
            <NodePalette />
            <CanvasToolbar />
            <NodeInspector />
          </>
        ) : (
          <Link
            href={`/flow/${slug}`}
            data-slot="edit-link"
            className="absolute top-4 right-4 z-10 rounded-full border border-border bg-card px-3 py-1.5 text-sm shadow-md"
          >
            Edit
          </Link>
        )}
      </FlowCanvas>
    </div>
  );
}

function BuilderShell({
  slug,
  initial,
  file,
  initialMode,
}: {
  slug: string;
  initial: Flow;
  file: Flow | null;
  initialMode: Mode;
}) {
  const [store] = React.useState<FlowStore>(() => createFlowStore(initial, { file, mode: initialMode }));

  React.useEffect(() => {
    if (initialMode !== "edit") return;
    return bindDraftPersistence(store, slug);
  }, [store, slug, initialMode]);

  return (
    <FlowStoreProvider store={store}>
      <BuilderChrome slug={slug} file={file} />
    </FlowStoreProvider>
  );
}

export function FlowBuilder({
  slug,
  file,
  initialMode,
}: {
  slug: string;
  file: Flow | null;
  initialMode: Mode;
}) {
  const [resolution, setResolution] = React.useState<Resolution>({ status: "pending" });

  React.useEffect(() => {
    // The only place this component reads localStorage. Resolving during render would make
    // this first client render disagree with the server's markup, which React reports as a
    // hydration mismatch, so the lookup waits for the mount effect instead.
    const draft = loadDraft(slug);
    const resolved = draft ?? file;
    setResolution(resolved ? { status: "resolved", flow: resolved } : { status: "missing" });
    // `slug` and `file` come from one server-rendered route; this effect is meant to run once
    // per page load, not on every re-render of its own inputs.
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [slug]);

  if (resolution.status === "missing") return <FlowNotFound slug={slug} />;

  if (resolution.status === "pending") {
    // Nothing to paint yet beats a flash of empty chrome; when `file` exists, paint it right
    // away so this render matches the server's exactly.
    if (!file) return null;
    return <BuilderShell key="pending" slug={slug} initial={file} file={file} initialMode={initialMode} />;
  }

  // A different `key` than the "pending" branch forces a fresh mount, and so a fresh `useState`
  // initializer, the moment resolution completes, so a draft found after mount always replaces
  // whatever the "pending" branch rendered, including nothing at all when `file` was null.
  return (
    <BuilderShell key="resolved" slug={slug} initial={resolution.flow} file={file} initialMode={initialMode} />
  );
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm test -- components/builder/flow-builder.test.tsx`
Expected: PASS, 4 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/flow-builder.tsx components/builder/flow-builder.test.tsx
git commit -m "$(cat <<'EOF'
feat(builder): add flow-builder shell with draft resolution

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
)"
```

---

#### Task 11c: Flow index component

**Files:**
- Create: `components/builder/flow-index.tsx`
- Test: `components/builder/flow-index.test.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // lib/flow-schema.ts
  export function newDraftFlow(now?: Date): Flow;   // id `draft-` + 6 lowercase alphanumerics, name "Untitled flow", source "manual", one start node at {x:0,y:0}

  // lib/storage.ts
  export function saveDraft(flow: Flow): void;
  export function listDrafts(): Flow[];   // by updatedAt desc; skips unparsable entries
  ```
  plus `useRouter` from `next/navigation` (package).
- Produces:
  ```ts
  // components/builder/flow-index.tsx
  export function FlowIndex(props: { flows: Flow[] }): JSX.Element;
  ```
  Rendered by `app/page.tsx` in Task 11d.

- [ ] **Step 1: Write the failing tests**

```tsx
// components/builder/flow-index.test.tsx
import { describe, expect, it, vi } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { saveDraft } from "@/lib/storage";
import type { Flow } from "@/lib/flow-schema";
import { FlowIndex } from "./flow-index";

const { mockPush } = vi.hoisted(() => ({ mockPush: vi.fn() }));
vi.mock("next/navigation", () => ({
  useRouter: () => ({ push: mockPush }),
}));

function flow(overrides: Partial<Flow>): Flow {
  return {
    schemaVersion: 1,
    id: "a",
    name: "A flow",
    source: "manual",
    createdAt: "2026-01-01T00:00:00.000Z",
    updatedAt: "2026-03-04T00:00:00.000Z",
    nodes: [],
    edges: [],
    ...overrides,
  };
}

describe("FlowIndex", () => {
  it("renders one card per committed flow with its node count and formatted date", () => {
    const shipNode = { id: "start-1", kind: "start" as const, position: { x: 0, y: 0 }, data: { label: "Start", trigger: "manual" as const } };
    render(<FlowIndex flows={[flow({ id: "ship", name: "Ship", nodes: [shipNode] })]} />);
    expect(screen.getByText("Ship")).toBeInTheDocument();
    expect(screen.getByText("1 nodes")).toBeInTheDocument();
    expect(screen.getByText("2026-03-04")).toBeInTheDocument();
  });

  it("shows No drafts yet when localStorage has none", () => {
    render(<FlowIndex flows={[]} />);
    expect(screen.getByText("No drafts yet")).toBeInTheDocument();
  });

  it("lists a seeded localStorage draft under Drafts in this browser", async () => {
    saveDraft(flow({ id: "draft-ab12cd", name: "My draft" }));
    render(<FlowIndex flows={[]} />);
    expect(await screen.findByText("My draft")).toBeInTheDocument();
  });

  it("New flow saves a draft and navigates to it", async () => {
    const user = userEvent.setup();
    render(<FlowIndex flows={[]} />);
    await user.click(screen.getByRole("button", { name: "New flow" }));
    expect(mockPush).toHaveBeenCalledTimes(1);
    expect(mockPush.mock.calls[0][0]).toMatch(/^\/flow\/draft-[a-z0-9]{6}$/);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm test -- components/builder/flow-index.test.tsx`
Expected: FAIL, `Cannot find module './flow-index'`.

- [ ] **Step 3: Implement the index**

```tsx
// components/builder/flow-index.tsx
"use client";

import * as React from "react";
import Link from "next/link";
import { useRouter } from "next/navigation";
import { Button } from "@/components/ui/button";
import { newDraftFlow, type Flow } from "@/lib/flow-schema";
import { listDrafts, saveDraft } from "@/lib/storage";

function formatDate(iso: string): string {
  // ISO 8601 timestamps already start with YYYY-MM-DD; slicing needs no timezone-sensitive
  // Date parsing.
  return iso.slice(0, 10);
}

function FlowCard({ flow }: { flow: Flow }) {
  return (
    <Link
      href={`/flow/${flow.id}`}
      data-slot="flow-card"
      className="flex flex-col gap-2 rounded-xl border border-border bg-card p-4 shadow-sm transition-shadow hover:shadow-md"
    >
      <div className="text-sm font-medium">{flow.name}</div>
      {flow.description && <div className="text-xs text-muted-foreground">{flow.description}</div>}
      <div className="mt-auto flex items-center justify-between text-xs text-muted-foreground">
        <span>{flow.nodes.length} nodes</span>
        <span>{formatDate(flow.updatedAt)}</span>
      </div>
    </Link>
  );
}

export function FlowIndex({ flows }: { flows: Flow[] }) {
  const router = useRouter();
  const [drafts, setDrafts] = React.useState<Flow[]>([]);

  React.useEffect(() => {
    setDrafts(listDrafts());
  }, []);

  const createFlow = () => {
    const flow = newDraftFlow();
    saveDraft(flow);
    router.push("/flow/" + flow.id);
  };

  return (
    <div data-slot="flow-index" className="mx-auto flex max-w-5xl flex-col gap-8 p-8">
      <div className="flex items-center justify-between">
        <h1 className="text-xl font-semibold">Flows</h1>
        <Button type="button" onClick={createFlow}>
          New flow
        </Button>
      </div>
      <section data-slot="flow-grid" className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
        {flows.map((flow) => (
          <FlowCard key={flow.id} flow={flow} />
        ))}
      </section>
      <section data-slot="drafts-section" className="flex flex-col gap-3">
        <h2 className="text-sm font-medium text-muted-foreground">Drafts in this browser</h2>
        {drafts.length === 0 ? (
          <p className="text-sm text-muted-foreground">No drafts yet</p>
        ) : (
          <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
            {drafts.map((flow) => (
              <FlowCard key={flow.id} flow={flow} />
            ))}
          </div>
        )}
      </section>
    </div>
  );
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm test -- components/builder/flow-index.test.tsx`
Expected: PASS, 4 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/flow-index.tsx components/builder/flow-index.test.tsx
git commit -m "$(cat <<'EOF'
feat(builder): add flow index with drafts section and new flow

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
)"
```

---

#### Task 11d: Server pages

**Files:**
- Create: `app/page.tsx`, `app/flow/[slug]/page.tsx`
- Test: `app/page.test.tsx`, `app/flow/[slug]/page.test.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // lib/flows.server.ts (server only)
  export const FLOWS_DIR: string;
  export async function listFlows(dir?: string): Promise<Flow[]>;
  export async function getFlow(slug: string, dir?: string): Promise<Flow | null>;
  ```
  plus `FlowIndex(props: { flows: Flow[] })` (Task 11c) and `FlowBuilder(props: { slug: string; file: Flow | null; initialMode: Mode })` (Task 11b).
- Produces: the two routes `/` and `/flow/[slug]` the rest of the app links to (`FlowIndex`'s cards, `TopBar`'s back link, `FlowCanvas`'s `?mode=view` share link all already point at these paths). No new exported names beyond the page components and Next's route-level exports (`dynamicParams`, `generateStaticParams`, `generateMetadata`), which are conventionally named and not part of the skeleton's Shared interfaces.

`getFlow` returning `null` for an unrecognized slug does not 404 here: the spec's route table says the client checks `localStorage` for a matching draft before giving up, and only `FlowBuilder` (Task 11b) can see `localStorage`. Calling `notFound()` in this server component would 404 every draft-only flow before the client ever got a chance to look.

`listFlows`/`getFlow` call `node:fs/promises`' `readdir`/`readFile` directly against `flows/`, with no fallback for a missing directory, so an empty index needs the directory to exist, not merely to contain zero flows. Task 12 fills it with the three seed files; until then this task adds a placeholder so `pnpm build` has something to read.

- [ ] **Step 1: Create the flows directory**

```bash
mkdir -p flows
touch flows/.gitkeep
```

Expected: `flows/.gitkeep` exists; `git status` shows it as untracked. Task 12 (seed flows, tracked separately) adds `ship.json`, `new-project.json` and `subagent-build-loop.json` alongside it; `.gitkeep` can stay or go at that point since `listFlows` only reads `*.json`.

- [ ] **Step 2: Write the failing test for the index page**

```tsx
// app/page.test.tsx
import { describe, expect, it, vi } from "vitest";
import { render, screen } from "@testing-library/react";
import type { Flow } from "@/lib/flow-schema";

const flow: Flow = {
  schemaVersion: 1,
  id: "ship",
  name: "Ship",
  source: "manual",
  createdAt: "2026-01-01T00:00:00.000Z",
  updatedAt: "2026-01-01T00:00:00.000Z",
  nodes: [],
  edges: [],
};

vi.mock("@/lib/flows.server", () => ({
  listFlows: vi.fn(async () => [flow]),
}));

import HomePage from "./page";

describe("HomePage", () => {
  it("passes listFlows() through to FlowIndex", async () => {
    render(await HomePage());
    expect(screen.getByText("Ship")).toBeInTheDocument();
  });
});
```

- [ ] **Step 3: Run to verify it fails**

Run: `pnpm test -- app/page.test.tsx`
Expected: FAIL, `Cannot find module './page'`.

- [ ] **Step 4: Implement app/page.tsx**

```tsx
// app/page.tsx
import { listFlows } from "@/lib/flows.server";
import { FlowIndex } from "@/components/builder/flow-index";

export default async function HomePage() {
  const flows = await listFlows();
  return <FlowIndex flows={flows} />;
}
```

- [ ] **Step 5: Run to verify it passes**

Run: `pnpm test -- app/page.test.tsx`
Expected: PASS, 1 test.

- [ ] **Step 6: Write the failing tests for the flow page**

```tsx
// app/flow/[slug]/page.test.tsx
import { describe, expect, it, vi } from "vitest";
import { render, screen } from "@testing-library/react";
import type { Flow } from "@/lib/flow-schema";

const shipFlow: Flow = {
  schemaVersion: 1,
  id: "ship",
  name: "Ship",
  source: "manual",
  createdAt: "2026-01-01T00:00:00.000Z",
  updatedAt: "2026-01-01T00:00:00.000Z",
  nodes: [],
  edges: [],
};

const { getFlowMock, listFlowsMock } = vi.hoisted(() => ({
  getFlowMock: vi.fn(),
  listFlowsMock: vi.fn(),
}));
vi.mock("@/lib/flows.server", () => ({
  getFlow: getFlowMock,
  listFlows: listFlowsMock,
}));
vi.mock("@/components/builder/flow-builder", () => ({
  FlowBuilder: (props: { slug: string; file: Flow | null; initialMode: string }) => (
    <div data-testid="flow-builder-stub">
      {props.slug}:{props.file ? "file" : "no-file"}:{props.initialMode}
    </div>
  ),
}));

import FlowPage, { dynamicParams, generateMetadata, generateStaticParams } from "./page";

describe("flow/[slug] page", () => {
  it("sets dynamicParams to true", () => {
    expect(dynamicParams).toBe(true);
  });

  it("generateStaticParams maps listFlows() to slugs", async () => {
    listFlowsMock.mockResolvedValue([shipFlow]);
    expect(await generateStaticParams()).toEqual([{ slug: "ship" }]);
  });

  it("generateMetadata uses the flow's name, falling back to Flow", async () => {
    getFlowMock.mockResolvedValue(shipFlow);
    expect(await generateMetadata({ params: Promise.resolve({ slug: "ship" }) })).toEqual({ title: "Ship" });

    getFlowMock.mockResolvedValue(null);
    expect(await generateMetadata({ params: Promise.resolve({ slug: "ghost" }) })).toEqual({ title: "Flow" });
  });

  it("passes the resolved flow and edit mode by default", async () => {
    getFlowMock.mockResolvedValue(shipFlow);
    render(
      await FlowPage({ params: Promise.resolve({ slug: "ship" }), searchParams: Promise.resolve({}) }),
    );
    expect(screen.getByTestId("flow-builder-stub")).toHaveTextContent("ship:file:edit");
  });

  it("passes a null file and view mode for an unknown slug with ?mode=view, without 404ing", async () => {
    getFlowMock.mockResolvedValue(null);
    render(
      await FlowPage({
        params: Promise.resolve({ slug: "ghost" }),
        searchParams: Promise.resolve({ mode: "view" }),
      }),
    );
    expect(screen.getByTestId("flow-builder-stub")).toHaveTextContent("ghost:no-file:view");
  });
});
```

- [ ] **Step 7: Run to verify it fails**

Run: `pnpm test -- "app/flow/[slug]/page.test.tsx"`
Expected: FAIL, `Cannot find module './page'`.

- [ ] **Step 8: Implement app/flow/[slug]/page.tsx**

```tsx
// app/flow/[slug]/page.tsx
import type { Metadata } from "next";
import { getFlow, listFlows } from "@/lib/flows.server";
import { FlowBuilder } from "@/components/builder/flow-builder";

export const dynamicParams = true;

export async function generateStaticParams() {
  const flows = await listFlows();
  return flows.map((flow) => ({ slug: flow.id }));
}

type Params = Promise<{ slug: string }>;
type SearchParams = Promise<{ [key: string]: string | string[] | undefined }>;

export async function generateMetadata({ params }: { params: Params }): Promise<Metadata> {
  const { slug } = await params;
  const flow = await getFlow(slug);
  return { title: flow ? flow.name : "Flow" };
}

export default async function FlowPage({
  params,
  searchParams,
}: {
  params: Params;
  searchParams: SearchParams;
}) {
  const { slug } = await params;
  const sp = await searchParams;
  const flow = await getFlow(slug);

  // An unrecognized slug still renders: the flow may exist only as a browser draft, which only
  // the client can see. Calling notFound() here would 404 every draft-only flow.
  return <FlowBuilder slug={slug} file={flow} initialMode={sp.mode === "view" ? "view" : "edit"} />;
}
```

- [ ] **Step 9: Run to verify it passes**

Run: `pnpm test -- "app/flow/[slug]/page.test.tsx"`
Expected: PASS, 5 tests.

- [ ] **Step 10: Run the full verification suite**

```bash
pnpm lint
pnpm typecheck
pnpm check:tokens
pnpm test
pnpm build
```

Expected, in order:
- `pnpm lint` and `pnpm typecheck` exit 0.
- `pnpm check:tokens` unchanged in count from Part C's last task (this task adds no `.tsx` under `components/**`, only under `app/**`, which the glob does not cover).
- `pnpm test` passes every test file from Tasks 1 through 11, including the four new ones in this task.
- `pnpm build` compiles and prints the route table with `○ /` (static) and `● /flow/[slug]` (or equivalent SSG/dynamic marker) with zero pre-rendered slugs, since `flows/` holds only `.gitkeep`; exits 0. This is the "empty index is valid" deliverable.

- [ ] **Step 11: Commit**

```bash
git add app/page.tsx app/page.test.tsx "app/flow/[slug]/page.tsx" "app/flow/[slug]/page.test.tsx" flows/.gitkeep
git commit -m "$(cat <<'EOF'
feat(app): add flow index and flow builder routes

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
)"
```

Expected: commit succeeds.

---

### Task 13: Playwright journey and CI e2e

Deliverable: `playwright.config.ts`, `e2e/builder.spec.ts`, and a CI `e2e` job. Split in two: the config and journey spec are locally runnable and reviewable on their own (Task 13a); the CI wiring is an infrastructure change a reviewer might want revised independently (Task 13b).

#### Task 13a: Playwright config and the builder journey spec

**Files:**
- Create: `playwright.config.ts`
- Create: `e2e/builder.spec.ts`

**Interfaces:**
- Consumes: no TypeScript imports from the app; the spec drives the built app over HTTP and asserts on the Shared DOM contract slots (`node`, `node-title`, `node-palette`, `inspector`, `status-badge`, `flow-canvas` with `data-mode`) and the handle-id codec (`{nodeId}:control:in`, `{nodeId}:control:out`, `{nodeId}:control:out@{port}`) fixed in the skeleton. It also reads `flows/ship.json` directly with `node:fs` (Task 12's deliverable, already committed by the time this task runs) to get the seed's node count rather than hardcoding one.
- Produces: content for the `pnpm e2e` script (already defined in Task 1's `package.json`); consumed by Task 13b's CI job and by Task 14's cross-browser verification run.

This task has no red/green cycle in the usual sense: the app it drives is already fully built by Tasks 1 through 12, so there is no failing implementation to make pass. Its "run to verify" steps instead confirm the suite is real and green against the finished app.

- [ ] **Step 1: Write playwright.config.ts**

```ts
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

const PORT = 3100;
const baseURL = process.env.PLAYWRIGHT_BASE_URL ?? `http://127.0.0.1:${PORT}`;

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  reporter: [["html", { outputFolder: "playwright-report", open: "never" }]],
  use: {
    baseURL,
    trace: "on-first-retry",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "webkit", use: { ...devices["Desktop Safari"] } },
  ],
  // Task 14 points this at a live Vercel preview by setting PLAYWRIGHT_BASE_URL. A deployed
  // preview already runs its own server, so this config must not also try to spawn one; the
  // local default keeps the skeleton's webServer block for every other run.
  ...(process.env.PLAYWRIGHT_BASE_URL
    ? {}
    : {
        webServer: {
          command: "pnpm start --port 3100",
          url: baseURL,
          reuseExistingServer: !process.env.CI,
        },
      }),
});
```

- [ ] **Step 2: Write the journey spec**

```ts
// e2e/builder.spec.ts
import { test, expect, type Page } from "@playwright/test";
import { readFileSync } from "node:fs";
import path from "node:path";

function readSeed(slug: string): { nodes: unknown[] } {
  const raw = readFileSync(path.join(process.cwd(), "flows", `${slug}.json`), "utf8");
  return JSON.parse(raw) as { nodes: unknown[] };
}

async function openFlow(page: Page, url: string) {
  await page.addInitScript(() => window.localStorage.clear());
  await page.goto(url);
}

test.describe("agent flow builder journey", () => {
  test("renders the ship seed's nodes", async ({ page }) => {
    const seed = readSeed("ship");
    await openFlow(page, "/flow/ship");
    await expect(page.locator('[data-slot="node"]')).toHaveCount(seed.nodes.length);
  });

  test("adding an Agent from the palette increments the node count", async ({ page }) => {
    const seed = readSeed("ship");
    await openFlow(page, "/flow/ship");
    await page.locator('[data-slot="node-palette"]').getByRole("button", { name: "Add Agent" }).click();
    await expect(page.locator('[data-slot="node"]')).toHaveCount(seed.nodes.length + 1);
  });

  test("connecting the new node increments the edge count", async ({ page }) => {
    await openFlow(page, "/flow/ship");
    await page.locator('[data-slot="node-palette"]').getByRole("button", { name: "Add Agent" }).click();

    const newNodeId = await page.locator('[data-slot="node"]').last().getAttribute("data-id");
    const source = page.locator('[data-kind="start"] [data-handleid$=":control:out"]');
    const target = page.locator(`[data-handleid="${newNodeId}:control:in"]`);
    const edgesBefore = await page.locator(".react-flow__edge").count();

    const sourceBox = await source.boundingBox();
    const targetBox = await target.boundingBox();
    if (!sourceBox || !targetBox) throw new Error("handle not visible");

    // React Flow tracks pointer-move events to draw the in-progress connection line and to
    // resolve the drop target on release. A single teleport from mouse.down straight to
    // mouse.up never fires those intermediate events, so the connection silently fails to
    // register without the stepped move.
    await page.mouse.move(sourceBox.x + sourceBox.width / 2, sourceBox.y + sourceBox.height / 2);
    await page.mouse.down();
    await page.mouse.move(targetBox.x + targetBox.width / 2, targetBox.y + targetBox.height / 2, { steps: 10 });
    await page.mouse.up();

    await expect(page.locator(".react-flow__edge")).toHaveCount(edgesBefore + 1);
  });

  test("renaming the new node via the inspector updates its card title", async ({ page }) => {
    await openFlow(page, "/flow/ship");
    await page.locator('[data-slot="node-palette"]').getByRole("button", { name: "Add Agent" }).click();
    const newNode = page.locator('[data-slot="node"]').last();
    await newNode.click();

    await page.locator('[data-slot="inspector"]').getByLabel("Label").fill("Renamed via inspector");

    await expect(newNode.locator('[data-slot="node-title"]')).toHaveText("Renamed via inspector");
  });

  test("exports JSON containing the renamed node", async ({ page }) => {
    await openFlow(page, "/flow/ship");
    await page.locator('[data-slot="node-palette"]').getByRole("button", { name: "Add Agent" }).click();
    const newNode = page.locator('[data-slot="node"]').last();
    await newNode.click();
    await page.locator('[data-slot="inspector"]').getByLabel("Label").fill("Exported label");

    await page.locator('[data-slot="top-bar"]').getByRole("button", { name: "More options" }).click();
    const downloadPromise = page.waitForEvent("download");
    await page.getByRole("menuitem", { name: "Export JSON" }).click();
    const download = await downloadPromise;
    const filePath = await download.path();
    if (!filePath) throw new Error("download did not save to disk");
    const exported = JSON.parse(readFileSync(filePath, "utf8")) as { nodes: { data: { label: string } }[] };

    expect(exported.nodes.some((n) => n.data.label === "Exported label")).toBe(true);
  });

  test("reload restores the added node and shows a Draft status", async ({ page }) => {
    const seed = readSeed("ship");
    await openFlow(page, "/flow/ship");
    await page.locator('[data-slot="node-palette"]').getByRole("button", { name: "Add Agent" }).click();
    await expect(page.locator('[data-slot="node"]')).toHaveCount(seed.nodes.length + 1);

    // Drafts persist 500ms after the last change (lib/draft-persistence.ts). Reloading before
    // that write lands would restore nothing, and this assertion would see the committed file
    // instead of the draft.
    await page.waitForTimeout(700);
    await page.reload();

    await expect(page.locator('[data-slot="node"]')).toHaveCount(seed.nodes.length + 1);
    await expect(page.locator('[data-slot="status-badge"]')).toHaveText("Draft");
  });

  test("?mode=view hides the palette and inspector, locks dragging, matches the node count", async ({ page }) => {
    const seed = readSeed("ship");
    await openFlow(page, "/flow/ship?mode=view");

    await expect(page.locator('[data-slot="flow-canvas"]')).toHaveAttribute("data-mode", "view");
    await expect(page.locator('[data-slot="node-palette"]')).toHaveCount(0);
    await expect(page.locator('[data-slot="inspector"]')).toHaveCount(0);
    await expect(page.locator('[data-slot="node"]')).toHaveCount(seed.nodes.length);

    const nodeWrapper = page.locator(".react-flow__node").first();
    const before = await nodeWrapper.evaluate((el) => getComputedStyle(el).transform);
    const box = await nodeWrapper.boundingBox();
    if (!box) throw new Error("node not visible");
    await page.mouse.move(box.x + box.width / 2, box.y + box.height / 2);
    await page.mouse.down();
    await page.mouse.move(box.x + box.width / 2 + 120, box.y + box.height / 2 + 80, { steps: 10 });
    await page.mouse.up();
    const after = await nodeWrapper.evaluate((el) => getComputedStyle(el).transform);

    expect(after).toBe(before);
  });
});
```

- [ ] **Step 3: Install Playwright browsers locally**

```bash
pnpm exec playwright install chromium webkit
```

Expected: downloads and installs both browser binaries, exits 0.

- [ ] **Step 4: Build the app so a production server exists to drive**

```bash
pnpm build
```

Expected: exits 0 (this is Task 12's seeded `flows/`, so the route table now lists the three prerendered `/flow/[slug]` pages alongside `/`).

- [ ] **Step 5: Run the suite**

```bash
pnpm e2e
```

Expected: Playwright starts `pnpm start --port 3100` itself (no `PLAYWRIGHT_BASE_URL` set locally), runs the 7 journey tests on both `chromium` and `webkit`, `14 passed`.

- [ ] **Step 6: Commit**

```bash
git add playwright.config.ts e2e/builder.spec.ts
git commit -m "$(cat <<'EOF'
test(e2e): add Playwright config and the builder journey spec

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
)"
```

Expected: commit succeeds.

---

#### Task 13b: CI e2e job

**Files:**
- Modify: `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: the `verify` job Task 1 wrote (lint, typecheck, check:tokens, test, build) and the `pnpm e2e` script Task 1 defined, now runnable thanks to Task 13a.
- Produces: a second job, `e2e`, that gates merges on the Playwright suite passing on both browsers.

- [ ] **Step 1: Append the e2e job**

Show the full file; only the `e2e` job at the bottom is new, the `verify` job is unchanged from Task 1.

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 11.1.0 }
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm check:tokens
      - run: pnpm test
      - run: pnpm build

  e2e:
    needs: verify
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 11.1.0 }
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps chromium webkit
      - run: pnpm build
      - run: pnpm e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

- [ ] **Step 2: Confirm the file reads back as written**

```bash
cat .github/workflows/ci.yml
```

Expected: prints exactly the two jobs above. This repository has no local YAML linter in its toolchain, so the definitive syntax check is GitHub Actions itself running this file once Task 14 pushes the branch; a bad indent or key would show up there as a workflow parse failure, not here.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "$(cat <<'EOF'
ci: run the Playwright e2e suite on Chromium and WebKit after build

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
)"
```

Expected: commit succeeds.

---
