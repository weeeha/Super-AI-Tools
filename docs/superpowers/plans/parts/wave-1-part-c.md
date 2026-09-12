### Task 7a: Kind-node and terminal-node renderers

**Files:**
- Create: `components/builder/nodes/kind-node.tsx`
- Create: `components/builder/nodes/kind-node.test.tsx`
- Create: `components/builder/nodes/terminal-node.tsx`
- Create: `components/builder/nodes/terminal-node.test.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // components/flow-kit/typed-handle.tsx
  export interface TypedHandleProps extends Omit<HandleProps, "type" | "position" | "id" | "isValidConnection"> {
    nodeId?: string;
    dataType: string;
    type: "source" | "target";
    position?: Position;
    top?: number;
    className?: string;
    port?: string;
    validate?: (c: Connection | Edge) => boolean;
  }
  export function TypedHandle(props: TypedHandleProps): JSX.Element;

  // lib/flow-handles.ts
  export const CONTROL = "control" as const;

  // lib/flow-convert.ts
  export type BuilderNode = Node<AnyKindData, NodeKind>;

  // components/builder/node-catalog.ts
  export type PortSpec = { dir: HandleDir; port?: string; label?: string };
  export interface CatalogEntry<K extends NodeKind = NodeKind> {
    kind: K; label: string; group: NodeGroup; icon: LucideIcon; renderer: Renderer;
    subtitle?: (data: KindData[K]) => string | undefined;
    defaultData: () => KindData[K];
    ports: (data: KindData[K]) => PortSpec[];
    fields: FieldDescriptor[];
    defaultSize?: { width: number; height: number };
  }
  export function catalogFor(kind: NodeKind): CatalogEntry;
  export function portsFor(node: BuilderNode): PortSpec[];

  // components/builder/use-connection-validator.ts (Task 6 stub)
  export function useConnectionValidator(): (c: Connection | Edge) => boolean;
  ```
- Produces:
  ```ts
  // components/builder/nodes/kind-node.tsx
  export function KindNode(props: NodeProps<BuilderNode>): JSX.Element;
  // components/builder/nodes/terminal-node.tsx
  export function TerminalNode(props: NodeProps<BuilderNode>): JSX.Element;
  ```
  Both are consumed by `components/builder/nodes/node-types.ts` in Task 7c.

Ports per kind (from the skeleton's node-catalog section): `agent`, `mcp`, `file-search`, `transform`, `set-state` each have exactly one unported `in` and one unported `out`. `user-approval` has one unported `in` plus two named out ports `approve` ("Approve") and `reject` ("Reject"). It uses `KindNode`, not `branch-node`, because its card is an icon tile with no visible per-port rows (only `guardrails`, `if-else` and `classify` render port rows, in Task 7b). `start` has one unported `out` only; `end` has one unported `in` only.

- [ ] **Step 1: Write the failing tests**

```tsx
// components/builder/nodes/kind-node.test.tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { ReactFlowProvider, type NodeProps } from "@xyflow/react";
import { createFlowStore, FlowStoreProvider } from "@/lib/flow-store";
import type { BuilderNode } from "@/lib/flow-convert";
import type { Flow, KindData, NodeKind } from "@/lib/flow-schema";
import { KindNode } from "./kind-node";

function baseNodeProps<K extends NodeKind>(overrides: {
  id: string;
  type: K;
  data: KindData[K];
  selected?: boolean;
}): NodeProps<BuilderNode> {
  return {
    ...overrides,
    selected: overrides.selected ?? false,
    dragging: false,
    zIndex: 0,
    selectable: true,
    deletable: true,
    draggable: true,
    isConnectable: true,
    positionAbsoluteX: 0,
    positionAbsoluteY: 0,
  } as NodeProps<BuilderNode>;
}

const emptyFlow: Flow = {
  schemaVersion: 1,
  id: "kind-node-fixture",
  name: "Kind node fixture",
  source: "manual",
  createdAt: "2026-01-01T00:00:00.000Z",
  updatedAt: "2026-01-01T00:00:00.000Z",
  nodes: [],
  edges: [],
};

function renderNode(props: NodeProps<BuilderNode>) {
  const store = createFlowStore(emptyFlow);
  return render(
    <FlowStoreProvider store={store}>
      <ReactFlowProvider>
        <KindNode {...props} />
      </ReactFlowProvider>
    </FlowStoreProvider>,
  );
}

describe("KindNode", () => {
  it("renders an agent node with one in and one out port", () => {
    renderNode(
      baseNodeProps({
        id: "agent-1",
        type: "agent",
        data: { label: "Researcher", instructions: "Find things" },
      }),
    );
    expect(screen.getByText("Researcher")).toBeInTheDocument();
    expect(document.querySelector('[data-handleid="agent-1:control:in"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="agent-1:control:out"]')).toBeTruthy();
  });

  it("shows the mcp server as the subtitle", () => {
    renderNode(baseNodeProps({ id: "mcp-1", type: "mcp", data: { label: "GitHub", server: "gh" } }));
    expect(screen.getByText("gh")).toBeInTheDocument();
  });

  it("renders file-search, transform and set-state with one in and one out port", () => {
    renderNode(baseNodeProps({ id: "fs-1", type: "file-search", data: { label: "Docs", store: "docs" } }));
    expect(document.querySelector('[data-handleid="fs-1:control:in"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="fs-1:control:out"]')).toBeTruthy();

    renderNode(
      baseNodeProps({ id: "tf-1", type: "transform", data: { label: "Shape", expression: "x => x.id" } }),
    );
    expect(document.querySelector('[data-handleid="tf-1:control:in"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="tf-1:control:out"]')).toBeTruthy();

    renderNode(
      baseNodeProps({ id: "ss-1", type: "set-state", data: { label: "Remember", assignments: [] } }),
    );
    expect(document.querySelector('[data-handleid="ss-1:control:in"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="ss-1:control:out"]')).toBeTruthy();
  });

  it("renders user-approval with an in port and approve/reject out ports, no visible rows", () => {
    renderNode(
      baseNodeProps({
        id: "ua-1",
        type: "user-approval",
        data: { label: "Ship it?", prompt: "Ready to deploy?" },
      }),
    );
    expect(document.querySelector('[data-handleid="ua-1:control:in"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="ua-1:control:out@approve"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="ua-1:control:out@reject"]')).toBeTruthy();
    expect(screen.getByLabelText("Approve output port")).toBeInTheDocument();
    expect(screen.getByLabelText("Reject output port")).toBeInTheDocument();
    expect(screen.queryByText("Approve")).not.toBeInTheDocument();
  });

  it("shows a ring when selected", () => {
    renderNode(
      baseNodeProps({
        id: "agent-2",
        type: "agent",
        data: { label: "Researcher", instructions: "Find things" },
        selected: true,
      }),
    );
    const card = document.querySelector('[data-slot="node"][data-kind="agent"]');
    expect(card?.className).toContain("ring-2");
  });

  it("carries the node-title slot and a description tooltip", () => {
    renderNode(
      baseNodeProps({
        id: "agent-3",
        type: "agent",
        data: { label: "Researcher", description: "Runs web research", instructions: "Find things" },
      }),
    );
    expect(document.querySelector('[data-slot="node-title"]')?.textContent).toBe("Researcher");
    expect(document.querySelector('[data-slot="node"]')).toHaveAttribute("title", "Runs web research");
  });
});
```

```tsx
// components/builder/nodes/terminal-node.test.tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { ReactFlowProvider, type NodeProps } from "@xyflow/react";
import { createFlowStore, FlowStoreProvider } from "@/lib/flow-store";
import type { BuilderNode } from "@/lib/flow-convert";
import type { Flow, KindData, NodeKind } from "@/lib/flow-schema";
import { TerminalNode } from "./terminal-node";

function baseNodeProps<K extends NodeKind>(overrides: { id: string; type: K; data: KindData[K] }): NodeProps<BuilderNode> {
  return {
    ...overrides,
    selected: false,
    dragging: false,
    zIndex: 0,
    selectable: true,
    deletable: true,
    draggable: true,
    isConnectable: true,
    positionAbsoluteX: 0,
    positionAbsoluteY: 0,
  } as NodeProps<BuilderNode>;
}

const emptyFlow: Flow = {
  schemaVersion: 1,
  id: "terminal-fixture",
  name: "Terminal fixture",
  source: "manual",
  createdAt: "2026-01-01T00:00:00.000Z",
  updatedAt: "2026-01-01T00:00:00.000Z",
  nodes: [],
  edges: [],
};

function renderNode(props: NodeProps<BuilderNode>) {
  const store = createFlowStore(emptyFlow);
  return render(
    <FlowStoreProvider store={store}>
      <ReactFlowProvider>
        <TerminalNode {...props} />
      </ReactFlowProvider>
    </FlowStoreProvider>,
  );
}

describe("TerminalNode", () => {
  it("renders start with a trigger subtitle and one out port only", () => {
    renderNode(baseNodeProps({ id: "start-1", type: "start", data: { label: "Start", trigger: "manual" } }));
    expect(screen.getByText("Start")).toBeInTheDocument();
    expect(document.querySelector('[data-handleid="start-1:control:out"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="start-1:control:in"]')).toBeNull();
  });

  it("renders end with one in port only and no subtitle", () => {
    renderNode(baseNodeProps({ id: "end-1", type: "end", data: { label: "End" } }));
    expect(screen.getByText("End")).toBeInTheDocument();
    expect(document.querySelector('[data-handleid="end-1:control:in"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="end-1:control:out"]')).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm test -- components/builder/nodes/kind-node.test.tsx components/builder/nodes/terminal-node.test.tsx`
Expected: FAIL with `Cannot find module './kind-node'` and `Cannot find module './terminal-node'`.

- [ ] **Step 3: Implement the renderers**

```tsx
// components/builder/nodes/kind-node.tsx
"use client";

import type { NodeProps } from "@xyflow/react";
import { CONTROL } from "@/lib/flow-handles";
import { catalogFor, portsFor } from "@/components/builder/node-catalog";
import { useConnectionValidator } from "@/components/builder/use-connection-validator";
import { TypedHandle } from "@/components/flow-kit/typed-handle";
import type { BuilderNode } from "@/lib/flow-convert";
import { cn } from "@/lib/utils";

const OUT_PORT_SPACING = 22;
const OUT_PORT_CENTER = 36;

function stackedTop(index: number, count: number): number | undefined {
  if (count <= 1) return undefined;
  return OUT_PORT_CENTER - ((count - 1) * OUT_PORT_SPACING) / 2 + index * OUT_PORT_SPACING;
}

export function KindNode({ id, data, selected, type }: NodeProps<BuilderNode>) {
  const validate = useConnectionValidator();
  const entry = catalogFor(type);
  const Icon = entry.icon;
  const node = { id, type, data, position: { x: 0, y: 0 } } as BuilderNode;
  const outPorts = portsFor(node).filter((port) => port.dir === "out");
  const subtitle = entry.subtitle?.(data);

  return (
    <div
      data-slot="node"
      data-kind={type}
      title={data.description}
      style={{ width: 260 }}
      className={cn(
        "flex items-center gap-2.5 rounded-2xl border border-border bg-card px-3 py-2.5 text-card-foreground shadow-sm",
        selected && "ring-2 ring-ring",
      )}
    >
      <div
        className="flex size-9 shrink-0 items-center justify-center rounded-lg text-white"
        style={{ background: `var(--flow-${entry.group})` }}
      >
        <Icon className="size-4" />
      </div>
      <div className="min-w-0 flex-1">
        <div data-slot="node-title" className="truncate text-[15px] font-medium">
          {data.label}
        </div>
        {subtitle && <div className="truncate text-xs text-muted-foreground">{subtitle}</div>}
      </div>
      <TypedHandle
        nodeId={id}
        dataType={CONTROL}
        type="target"
        validate={validate}
        aria-label="Control input port"
      />
      {outPorts.map((port, index) => (
        <TypedHandle
          key={port.port ?? "out"}
          nodeId={id}
          dataType={CONTROL}
          type="source"
          port={port.port}
          validate={validate}
          top={stackedTop(index, outPorts.length)}
          aria-label={`${port.label ?? "Control"} output port`}
        />
      ))}
    </div>
  );
}
```

```tsx
// components/builder/nodes/terminal-node.tsx
"use client";

import type { NodeProps } from "@xyflow/react";
import { CONTROL } from "@/lib/flow-handles";
import { catalogFor } from "@/components/builder/node-catalog";
import { useConnectionValidator } from "@/components/builder/use-connection-validator";
import { TypedHandle } from "@/components/flow-kit/typed-handle";
import type { BuilderNode } from "@/lib/flow-convert";
import { cn } from "@/lib/utils";

export function TerminalNode({ id, data, selected, type }: NodeProps<BuilderNode>) {
  const validate = useConnectionValidator();
  const entry = catalogFor(type);
  const Icon = entry.icon;
  const isStart = type === "start";
  const subtitle = isStart ? entry.subtitle?.(data) : undefined;

  return (
    <div
      data-slot="node"
      data-kind={type}
      title={data.description}
      style={{ width: 260 }}
      className={cn(
        "flex items-center gap-2 rounded-full border border-border bg-card px-4 py-2.5 text-card-foreground shadow-sm",
        selected && "ring-2 ring-ring",
      )}
    >
      <Icon className="size-4 shrink-0 text-muted-foreground" />
      <div className="min-w-0 flex-1">
        <div data-slot="node-title" className="truncate text-[15px] font-medium">
          {data.label}
        </div>
        {subtitle && <div className="truncate text-xs text-muted-foreground">{subtitle}</div>}
      </div>
      {isStart ? (
        <TypedHandle
          nodeId={id}
          dataType={CONTROL}
          type="source"
          validate={validate}
          aria-label="Control output port"
        />
      ) : (
        <TypedHandle
          nodeId={id}
          dataType={CONTROL}
          type="target"
          validate={validate}
          aria-label="Control input port"
        />
      )}
    </div>
  );
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm test -- components/builder/nodes/kind-node.test.tsx components/builder/nodes/terminal-node.test.tsx`
Expected: PASS, 6 tests in `kind-node.test.tsx`, 2 tests in `terminal-node.test.tsx`.

- [ ] **Step 5: Commit**

```bash
git add components/builder/nodes/kind-node.tsx components/builder/nodes/kind-node.test.tsx \
  components/builder/nodes/terminal-node.tsx components/builder/nodes/terminal-node.test.tsx
git commit -m "feat(builder): add kind-node and terminal-node renderers

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 7b: Branch-node renderer

**Files:**
- Create: `components/builder/nodes/branch-node.tsx`
- Create: `components/builder/nodes/branch-node.test.tsx`

**Interfaces:**
- Consumes: `TypedHandle`/`TypedHandleProps` and `CONTROL`/`BuilderNode`/`catalogFor`/`portsFor`/`PortSpec` (verbatim, same as Task 7a) plus:
  ```ts
  // lib/flow-schema.ts
  export type IfElseData = Base & { label: string; description?: string; conditions: { label: string; expression: string }[] };
  export type GuardrailsData = Base & { label: string; description?: string; checks: string[] };
  export type ClassifyData = Base & { label: string; description?: string; categories: string[] };
  ```
- Produces:
  ```ts
  // components/builder/nodes/branch-node.tsx
  export function BranchNode(props: NodeProps<BuilderNode>): JSX.Element;
  ```
  Consumed by `components/builder/nodes/node-types.ts` in Task 7c.

`portsFor` for `guardrails` returns exactly two out ports (`pass` "Pass", `fail` "Fail") regardless of how many checks exist. The individual check strings are informational text above the port rows, not one row per check. `classify` returns one out port per category (`cat-{i}`, label = the category text). `if-else` returns one out port per condition (`c{i}`, label = the condition's own label) plus a trailing `else` port labelled "Else".

- [ ] **Step 1: Write the failing test**

```tsx
// components/builder/nodes/branch-node.test.tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { ReactFlowProvider, type NodeProps } from "@xyflow/react";
import { createFlowStore, FlowStoreProvider } from "@/lib/flow-store";
import type { BuilderNode } from "@/lib/flow-convert";
import type { Flow, KindData, NodeKind } from "@/lib/flow-schema";
import { BranchNode } from "./branch-node";

function baseNodeProps<K extends NodeKind>(overrides: { id: string; type: K; data: KindData[K] }): NodeProps<BuilderNode> {
  return {
    ...overrides,
    selected: false,
    dragging: false,
    zIndex: 0,
    selectable: true,
    deletable: true,
    draggable: true,
    isConnectable: true,
    positionAbsoluteX: 0,
    positionAbsoluteY: 0,
  } as NodeProps<BuilderNode>;
}

const emptyFlow: Flow = {
  schemaVersion: 1,
  id: "branch-node-fixture",
  name: "Branch node fixture",
  source: "manual",
  createdAt: "2026-01-01T00:00:00.000Z",
  updatedAt: "2026-01-01T00:00:00.000Z",
  nodes: [],
  edges: [],
};

function renderNode(props: NodeProps<BuilderNode>) {
  const store = createFlowStore(emptyFlow);
  return render(
    <FlowStoreProvider store={store}>
      <ReactFlowProvider>
        <BranchNode {...props} />
      </ReactFlowProvider>
    </FlowStoreProvider>,
  );
}

describe("BranchNode", () => {
  it("renders one row per if-else condition plus a trailing Else row", () => {
    renderNode(
      baseNodeProps({
        id: "if-1",
        type: "if-else",
        data: {
          label: "Check details",
          conditions: [
            { label: "Has details", expression: "input.output_parsed.has_all_details == true" },
            { label: "Needs review", expression: "input.output_parsed.needs_review == true" },
          ],
        },
      }),
    );
    expect(screen.getByText("Has details")).toBeInTheDocument();
    expect(screen.getByText("input.output_parsed.has_all_details == true")).toBeInTheDocument();
    expect(screen.getByText("Needs review")).toBeInTheDocument();
    expect(screen.getByText("Else")).toBeInTheDocument();
    expect(document.querySelector('[data-handleid="if-1:control:in"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="if-1:control:out@c0"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="if-1:control:out@c1"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="if-1:control:out@else"]')).toBeTruthy();
    expect(document.querySelectorAll('[data-slot="branch-row"]')).toHaveLength(3);
  });

  it("renders one row per classify category", () => {
    renderNode(
      baseNodeProps({
        id: "classify-1",
        type: "classify",
        data: { label: "Route", categories: ["personal", "client", "promotion"] },
      }),
    );
    expect(screen.getByText("personal")).toBeInTheDocument();
    expect(screen.getByText("client")).toBeInTheDocument();
    expect(screen.getByText("promotion")).toBeInTheDocument();
    expect(document.querySelector('[data-handleid="classify-1:control:out@cat-0"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="classify-1:control:out@cat-2"]')).toBeTruthy();
    expect(document.querySelectorAll('[data-slot="branch-row"]')).toHaveLength(3);
  });

  it("lists checks as text and renders Pass/Fail rows for guardrails", () => {
    renderNode(
      baseNodeProps({
        id: "guardrails-1",
        type: "guardrails",
        data: { label: "Stop conditions", checks: ["scope creep", "tests failing"] },
      }),
    );
    expect(screen.getByText("scope creep", { exact: false })).toBeInTheDocument();
    expect(screen.getByText("tests failing", { exact: false })).toBeInTheDocument();
    expect(screen.getByText("Pass")).toBeInTheDocument();
    expect(screen.getByText("Fail")).toBeInTheDocument();
    expect(document.querySelector('[data-handleid="guardrails-1:control:in"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="guardrails-1:control:out@pass"]')).toBeTruthy();
    expect(document.querySelector('[data-handleid="guardrails-1:control:out@fail"]')).toBeTruthy();
    expect(document.querySelectorAll('[data-slot="branch-row"]')).toHaveLength(2);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test -- components/builder/nodes/branch-node.test.tsx`
Expected: FAIL with `Cannot find module './branch-node'`.

- [ ] **Step 3: Implement the renderer**

```tsx
// components/builder/nodes/branch-node.tsx
"use client";

import type { NodeProps } from "@xyflow/react";
import { CONTROL } from "@/lib/flow-handles";
import { catalogFor, portsFor } from "@/components/builder/node-catalog";
import { useConnectionValidator } from "@/components/builder/use-connection-validator";
import { TypedHandle } from "@/components/flow-kit/typed-handle";
import type { BuilderNode } from "@/lib/flow-convert";
import type { GuardrailsData, IfElseData } from "@/lib/flow-schema";
import { cn } from "@/lib/utils";

const HEADER_HEIGHT = 48;
const ROW_HEIGHT = 36;
const CHECK_ROW_HEIGHT = 16;
const CHECKS_PADDING = 20;

export function BranchNode({ id, data, selected, type }: NodeProps<BuilderNode>) {
  const validate = useConnectionValidator();
  const entry = catalogFor(type);
  const Icon = entry.icon;
  const node = { id, type, data, position: { x: 0, y: 0 } } as BuilderNode;
  const outPorts = portsFor(node).filter((port) => port.dir === "out");
  const conditions = type === "if-else" ? (data as IfElseData).conditions : undefined;
  const checks = type === "guardrails" ? (data as GuardrailsData).checks : undefined;
  const checksHeight = checks && checks.length > 0 ? CHECKS_PADDING + checks.length * CHECK_ROW_HEIGHT : 0;

  return (
    <div
      data-slot="node"
      data-kind={type}
      title={data.description}
      style={{ width: 260 }}
      className={cn(
        "overflow-hidden rounded-2xl border border-border bg-card text-card-foreground shadow-sm",
        selected && "ring-2 ring-ring",
      )}
    >
      <div className="flex items-center gap-2 px-3" style={{ height: HEADER_HEIGHT }}>
        <div
          className="flex size-9 shrink-0 items-center justify-center rounded-lg text-white"
          style={{ background: `var(--flow-${entry.group})` }}
        >
          <Icon className="size-4" />
        </div>
        <div data-slot="node-title" className="truncate text-[15px] font-medium">
          {data.label}
        </div>
      </div>
      {checks && checks.length > 0 && (
        <ul className="flex flex-col gap-0.5 border-t border-border px-3 py-2 text-xs text-muted-foreground">
          {checks.map((check, index) => (
            <li key={index} className="truncate">
              &bull; {check}
            </li>
          ))}
        </ul>
      )}
      <div className="border-t border-border">
        {outPorts.map((port, index) => {
          const isElse = port.port === "else";
          const condition = conditions?.[index];
          return (
            <div
              key={port.port ?? index}
              data-slot="branch-row"
              className={cn(
                "relative flex min-w-0 items-center border-b border-border px-3 text-sm last:border-b-0",
                isElse && "justify-end",
              )}
              style={{ height: ROW_HEIGHT }}
            >
              {isElse ? (
                <span className="text-muted-foreground">Else</span>
              ) : type === "if-else" && condition ? (
                <div className="flex min-w-0 flex-col justify-center leading-tight">
                  <span className="truncate">{port.label}</span>
                  <span className="truncate font-mono text-[11px] text-muted-foreground">
                    {condition.expression}
                  </span>
                </div>
              ) : (
                <span className="truncate">{port.label}</span>
              )}
              <TypedHandle
                nodeId={id}
                dataType={CONTROL}
                type="source"
                port={port.port}
                validate={validate}
                top={HEADER_HEIGHT + checksHeight + index * ROW_HEIGHT + ROW_HEIGHT / 2}
                aria-label={`${port.label ?? "Control"} output port`}
              />
            </div>
          );
        })}
      </div>
      <TypedHandle
        nodeId={id}
        dataType={CONTROL}
        type="target"
        validate={validate}
        top={HEADER_HEIGHT / 2}
        aria-label="Control input port"
      />
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/nodes/branch-node.test.tsx`
Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/nodes/branch-node.tsx components/builder/nodes/branch-node.test.tsx
git commit -m "feat(builder): add branch-node renderer for if-else, classify and guardrails

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 7c: Note-node renderer and node-types map

**Files:**
- Create: `components/builder/nodes/note-node.tsx`
- Create: `components/builder/nodes/note-node.test.tsx`
- Create: `components/builder/nodes/node-types.ts`

**Interfaces:**
- Consumes: `BuilderNode` (from `lib/flow-convert.ts`), `NodeKind` (from `lib/flow-schema.ts`), `NoteData` (from `lib/flow-schema.ts`), `KindNode`/`TerminalNode` (Task 7a), `BranchNode` (Task 7b), `WhileNode` from `components/builder/nodes/while-node.tsx` (Task 6, already built with the renderer contract: root `data-slot="node"` `data-kind="while"`, body `data-slot="while-body"`).
- Produces:
  ```ts
  // components/builder/nodes/note-node.tsx
  export function NoteNode(props: NodeProps<BuilderNode>): JSX.Element;
  // components/builder/nodes/node-types.ts
  export const nodeTypes: Record<NodeKind, React.ComponentType<NodeProps<BuilderNode>>>;
  ```
  `nodeTypes` is consumed by `components/builder/flow-canvas.tsx` in Task 8a.

`node-types.ts` has no dedicated test file in the skeleton's file structure, its correctness is exercised indirectly by `flow-canvas.test.tsx` (Task 8) and the Playwright journey (Task 13). Its own verification step here is a typecheck, since a wrong or missing kind would surface as a `Record<NodeKind, ...>` type error.

- [ ] **Step 1: Write the failing test**

```tsx
// components/builder/nodes/note-node.test.tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { ReactFlowProvider, type NodeProps } from "@xyflow/react";
import { createFlowStore, FlowStoreProvider } from "@/lib/flow-store";
import type { BuilderNode } from "@/lib/flow-convert";
import type { Flow } from "@/lib/flow-schema";
import { NoteNode } from "./note-node";

function baseNodeProps(overrides: Partial<NodeProps<BuilderNode>> & Pick<NodeProps<BuilderNode>, "id" | "data">): NodeProps<BuilderNode> {
  return {
    type: "note",
    selected: false,
    dragging: false,
    zIndex: -1,
    selectable: true,
    deletable: true,
    draggable: true,
    isConnectable: false,
    positionAbsoluteX: 0,
    positionAbsoluteY: 0,
    ...overrides,
  } as NodeProps<BuilderNode>;
}

const emptyFlow: Flow = {
  schemaVersion: 1,
  id: "note-fixture",
  name: "Note fixture",
  source: "manual",
  createdAt: "2026-01-01T00:00:00.000Z",
  updatedAt: "2026-01-01T00:00:00.000Z",
  nodes: [],
  edges: [],
};

function renderNote(props: NodeProps<BuilderNode>) {
  const store = createFlowStore(emptyFlow);
  return render(
    <FlowStoreProvider store={store}>
      <ReactFlowProvider>
        <NoteNode {...props} />
      </ReactFlowProvider>
    </FlowStoreProvider>,
  );
}

describe("NoteNode", () => {
  it("renders its text and color token with no ports", () => {
    renderNote(
      baseNodeProps({
        id: "note-1",
        data: { label: "Note", text: "Remember to branch first", color: "yellow" },
      }),
    );
    expect(screen.getByText("Remember to branch first")).toBeInTheDocument();
    const card = document.querySelector('[data-slot="node"][data-kind="note"]') as HTMLElement;
    expect(card.style.background).toBe("var(--flow-note-yellow)");
    expect(document.querySelectorAll('[data-slot="typed-handle"]')).toHaveLength(0);
  });

  it("shows the resize handles only when selected", () => {
    const { rerender } = renderNote(
      baseNodeProps({ id: "note-2", data: { label: "Note", text: "Unselected", color: "blue" } }),
    );
    expect(document.querySelector(".react-flow__resize-control")).toBeNull();
    rerender(
      <FlowStoreProvider store={createFlowStore(emptyFlow)}>
        <ReactFlowProvider>
          <NoteNode
            {...baseNodeProps({
              id: "note-2",
              data: { label: "Note", text: "Selected", color: "blue" },
              selected: true,
            })}
          />
        </ReactFlowProvider>
      </FlowStoreProvider>,
    );
    expect(document.querySelector(".react-flow__resize-control")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test -- components/builder/nodes/note-node.test.tsx`
Expected: FAIL with `Cannot find module './note-node'`.

- [ ] **Step 3: Implement the renderer and the node-types map**

```tsx
// components/builder/nodes/note-node.tsx
"use client";

import { NodeResizer, type NodeProps } from "@xyflow/react";
import type { BuilderNode } from "@/lib/flow-convert";
import type { NoteData } from "@/lib/flow-schema";
import { cn } from "@/lib/utils";

export function NoteNode({ data, selected }: NodeProps<BuilderNode>) {
  const note = data as NoteData;

  return (
    <div
      data-slot="node"
      data-kind="note"
      title={data.description}
      className={cn("size-full overflow-auto p-3 text-[13px]", selected && "ring-2 ring-ring")}
      style={{ background: `var(--flow-note-${note.color})`, borderRadius: 8 }}
    >
      <NodeResizer minWidth={160} minHeight={100} isVisible={selected} />
      <div data-slot="node-title" className="sr-only">
        {note.label}
      </div>
      <p className="whitespace-pre-wrap break-words">{note.text}</p>
    </div>
  );
}
```

```ts
// components/builder/nodes/node-types.ts
import type { ComponentType } from "react";
import type { NodeProps } from "@xyflow/react";
import type { BuilderNode } from "@/lib/flow-convert";
import type { NodeKind } from "@/lib/flow-schema";
import { KindNode } from "./kind-node";
import { TerminalNode } from "./terminal-node";
import { BranchNode } from "./branch-node";
import { WhileNode } from "./while-node";
import { NoteNode } from "./note-node";

export const nodeTypes: Record<NodeKind, ComponentType<NodeProps<BuilderNode>>> = {
  start: TerminalNode,
  end: TerminalNode,
  agent: KindNode,
  "file-search": KindNode,
  mcp: KindNode,
  transform: KindNode,
  "set-state": KindNode,
  "user-approval": KindNode,
  classify: BranchNode,
  "if-else": BranchNode,
  guardrails: BranchNode,
  while: WhileNode,
  note: NoteNode,
};
```

- [ ] **Step 4: Run test to verify it passes, then typecheck the map**

Run: `pnpm test -- components/builder/nodes/note-node.test.tsx`
Expected: PASS, 2 tests.

Run: `pnpm typecheck`
Expected: no errors. If a kind is missing from `nodeTypes`, `tsc` reports `Property '<kind>' is missing in type ...` against the `Record<NodeKind, ...>` annotation, that failure mode is `node-types.ts`'s only safety net, so do not weaken the annotation to `Partial<Record<...>>`.

- [ ] **Step 5: Commit**

```bash
git add components/builder/nodes/note-node.tsx components/builder/nodes/note-node.test.tsx \
  components/builder/nodes/node-types.ts
git commit -m "feat(builder): add note-node renderer and the node-types map

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 7d: Builder-edge with port label pills

**Files:**
- Create: `components/builder/builder-edge.tsx`
- Create: `components/builder/builder-edge.test.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // components/flow-kit/typed-edge.tsx
  export type TypedEdgeType = Edge<{ streaming?: boolean }, "typed">;
  export function TypedEdge(props: EdgeProps<TypedEdgeType>): JSX.Element;

  // lib/flow-convert.ts
  export type BuilderEdge = Edge<{ port?: string; streaming?: boolean }, "typed">;

  // components/builder/node-catalog.ts
  export function portLabel(node: BuilderNode, port: string | undefined): string | undefined;

  // lib/flow-store.ts
  export function useFlowStore<T>(selector: (s: FlowState) => T): T;
  ```
  plus `KindNode` (Task 7a) and `BranchNode` (Task 7b) for the DOM smoke test below.
- Produces:
  ```ts
  // components/builder/builder-edge.tsx
  export function BuilderEdge(props: EdgeProps<BuilderEdge>): JSX.Element;
  ```
  Consumed as `edgeTypes.typed` by `components/builder/flow-canvas.tsx` in Task 8a.

`EdgeLabelRenderer` portals its children into a DOM node (`.react-flow__edgelabel-renderer`) that only the real `<ReactFlow>` component creates on mount, `<ReactFlowProvider>` alone does not create it, so the pill only appears in the DOM once a `<ReactFlow>` instance is actually mounted. The test below mounts a minimal, real `<ReactFlow>` (reusing `KindNode`/`BranchNode` from earlier in this task so the rendered handles match real handle ids) instead of invoking `BuilderEdge` as a bare function component.

- [ ] **Step 1: Write the failing test**

```tsx
// components/builder/builder-edge.test.tsx
import { describe, expect, it } from "vitest";
import { render, screen } from "@testing-library/react";
import { ReactFlow, ReactFlowProvider } from "@xyflow/react";
import { createFlowStore, FlowStoreProvider } from "@/lib/flow-store";
import { portHandleId } from "@/lib/flow-handles";
import type { BuilderEdge as BuilderEdgeType, BuilderNode } from "@/lib/flow-convert";
import type { Flow } from "@/lib/flow-schema";
import { BuilderEdge } from "./builder-edge";
import { KindNode } from "./nodes/kind-node";
import { BranchNode } from "./nodes/branch-node";

const flow: Flow = {
  schemaVersion: 1,
  id: "edge-fixture",
  name: "Edge fixture",
  source: "manual",
  createdAt: "2026-01-01T00:00:00.000Z",
  updatedAt: "2026-01-01T00:00:00.000Z",
  nodes: [
    {
      id: "if-1",
      kind: "if-else",
      position: { x: 0, y: 0 },
      data: { label: "Check", conditions: [{ label: "Ready", expression: "state.ready == true" }] },
    },
    { id: "agent-1", kind: "agent", position: { x: 320, y: 0 }, data: { label: "Agent", instructions: "Do the thing" } },
    { id: "agent-2", kind: "agent", position: { x: 640, y: 0 }, data: { label: "Second agent", instructions: "Then this" } },
  ],
  edges: [],
};

function renderGraph() {
  const store = createFlowStore(flow);
  const nodes: BuilderNode[] = [
    { id: "if-1", type: "if-else", position: { x: 0, y: 0 }, data: flow.nodes[0].data as never },
    { id: "agent-1", type: "agent", position: { x: 320, y: 0 }, data: flow.nodes[1].data as never },
    { id: "agent-2", type: "agent", position: { x: 640, y: 0 }, data: flow.nodes[2].data as never },
  ];
  const edges: BuilderEdgeType[] = [
    {
      id: "e-else",
      source: "if-1",
      target: "agent-1",
      sourceHandle: portHandleId("if-1", "out", "else"),
      targetHandle: portHandleId("agent-1", "in"),
      type: "typed",
      data: { port: "else" },
    },
    {
      id: "e-plain",
      source: "agent-1",
      target: "agent-2",
      sourceHandle: portHandleId("agent-1", "out"),
      targetHandle: portHandleId("agent-2", "in"),
      type: "typed",
      data: {},
    },
  ];
  return render(
    <FlowStoreProvider store={store}>
      <ReactFlowProvider>
        <div style={{ width: 800, height: 300 }}>
          <ReactFlow
            nodes={nodes}
            edges={edges}
            nodeTypes={{ "if-else": BranchNode, agent: KindNode }}
            edgeTypes={{ typed: BuilderEdge }}
          />
        </div>
      </ReactFlowProvider>
    </FlowStoreProvider>,
  );
}

describe("BuilderEdge", () => {
  it("shows the pill text for a named port", () => {
    renderGraph();
    expect(screen.getByText("Else")).toBeInTheDocument();
  });

  it("renders no pill for an edge whose port is undefined", () => {
    renderGraph();
    expect(document.querySelectorAll('[data-slot="edge-port-pill"]')).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test -- components/builder/builder-edge.test.tsx`
Expected: FAIL with `Cannot find module './builder-edge'`.

- [ ] **Step 3: Implement the edge renderer**

```tsx
// components/builder/builder-edge.tsx
"use client";

import { EdgeLabelRenderer, getBezierPath, type EdgeProps } from "@xyflow/react";
import { TypedEdge } from "@/components/flow-kit/typed-edge";
import { portLabel } from "@/components/builder/node-catalog";
import { useFlowStore } from "@/lib/flow-store";
import type { BuilderEdge as BuilderEdgeType } from "@/lib/flow-convert";

export function BuilderEdge(props: EdgeProps<BuilderEdgeType>) {
  const { source, data, sourceX, sourceY, targetX, targetY, sourcePosition, targetPosition } = props;
  const sourceNode = useFlowStore((s) => s.nodes.find((n) => n.id === source));
  const [, labelX, labelY] = getBezierPath({ sourceX, sourceY, targetX, targetY, sourcePosition, targetPosition });
  const label = data?.port && sourceNode ? portLabel(sourceNode, data.port) : undefined;

  return (
    <>
      <TypedEdge {...props} />
      {label && (
        <EdgeLabelRenderer>
          <div
            data-slot="edge-port-pill"
            className="nodrag nopan absolute rounded-full border border-border bg-card px-2 text-xs"
            style={{ transform: `translate(-50%, -50%) translate(${labelX}px, ${labelY}px)` }}
          >
            {label}
          </div>
        </EdgeLabelRenderer>
      )}
    </>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/builder-edge.test.tsx`
Expected: PASS, 2 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/builder-edge.tsx components/builder/builder-edge.test.tsx
git commit -m "feat(builder): add builder-edge with a port-label pill

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 8a: Flow canvas core rendering, tool modes and view lock

**Files:**
- Create: `components/builder/flow-canvas.tsx`
- Create: `components/builder/flow-canvas.test.tsx`
- Modify: `components/builder/use-connection-validator.ts` (finalize the Task 6 stub)

**Interfaces:**
- Consumes:
  ```ts
  // lib/flow-store.ts
  export type Tool = "select" | "hand";
  export type Mode = "edit" | "view";
  export interface FlowState {
    meta: FlowMeta; nodes: BuilderNode[]; edges: BuilderEdge[]; viewport?: FlowViewport;
    mode: Mode; tool: Tool; storageStatus: "ok" | "failing";
    fileSnapshot: string | null;
    setViewport(v: FlowViewport): void;
    applyNodeChanges(changes: NodeChange<BuilderNode>[]): void;
    applyEdgeChanges(changes: EdgeChange<BuilderEdge>[]): void;
    connect(c: Connection): void;
    // ...remaining actions unchanged from the skeleton
  }
  export function useFlowStore<T>(selector: (s: FlowState) => T): T;
  export function useFlowStoreApi(): FlowStore;

  // lib/connection-rules.ts
  export function isValidBuilderConnection(c: Connection | Edge, nodes: BuilderNode[]): boolean;

  // components/builder/nodes/node-types.ts
  export const nodeTypes: Record<NodeKind, React.ComponentType<NodeProps<BuilderNode>>>;
  ```
  plus `BuilderEdge` (component, Task 7d).
- Produces:
  ```ts
  // components/builder/flow-canvas.tsx
  export function FlowCanvas(props: { className?: string }): JSX.Element;
  // components/builder/use-connection-validator.ts
  export function useConnectionValidator(): (c: Connection | Edge) => boolean;
  ```
  `FlowCanvas` gains a `children` prop in Task 9a and is mounted by `flow-builder.tsx` in Task 11. Drag-and-drop (`onDrop`, `onNodeDragStop`) is added in Task 8b.

This task assumes `tool` defaults to `"select"` in `createFlowStore`. The skeleton fixes the store's action names but not this initial value, so the toolbar's default active button (Task 9b) and the test below both key off `"select"`.

- [ ] **Step 1: Write the failing test**

```tsx
// components/builder/flow-canvas.test.tsx
import { describe, expect, it } from "vitest";
import { createFlowStore } from "@/lib/flow-store";
import { fixtureFlow, renderWithStore } from "@/lib/test-fixtures";
import { FlowCanvas } from "./flow-canvas";

describe("FlowCanvas", () => {
  it("renders one element per fixture node", () => {
    const store = createFlowStore(fixtureFlow());
    const { container } = renderWithStore(<FlowCanvas />, store);
    for (const node of store.getState().nodes) {
      expect(container.querySelector(`[data-id="${node.id}"]`)).toBeTruthy();
    }
  });

  it("renders the pane and a minimap in edit mode", () => {
    const store = createFlowStore(fixtureFlow());
    const { container } = renderWithStore(<FlowCanvas />, store);
    expect(container.querySelector(".react-flow__pane")).toBeTruthy();
    expect(container.querySelector(".react-flow__minimap")).toBeTruthy();
  });

  it("locks dragging and exposes the mode in view mode", () => {
    const store = createFlowStore(fixtureFlow(), { mode: "view" });
    const { container } = renderWithStore(<FlowCanvas />, store);
    const wrapper = container.querySelector('[data-slot="flow-canvas"]');
    expect(wrapper).toHaveAttribute("data-mode", "view");
    const node = container.querySelector(".react-flow__node");
    expect(node?.className).not.toContain("draggable");
  });

  it("leaves dragging enabled in edit mode", () => {
    const store = createFlowStore(fixtureFlow());
    const { container } = renderWithStore(<FlowCanvas />, store);
    const node = container.querySelector(".react-flow__node");
    expect(node?.className).toContain("draggable");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test -- components/builder/flow-canvas.test.tsx`
Expected: FAIL with `Cannot find module './flow-canvas'`.

- [ ] **Step 3: Finalize the connection validator and implement the canvas**

```ts
// components/builder/use-connection-validator.ts
"use client";

import { useCallback } from "react";
import type { Connection, Edge } from "@xyflow/react";
import { useFlowStoreApi } from "@/lib/flow-store";
import { isValidBuilderConnection } from "@/lib/connection-rules";

export function useConnectionValidator(): (c: Connection | Edge) => boolean {
  const storeApi = useFlowStoreApi();
  return useCallback(
    (c: Connection | Edge) => isValidBuilderConnection(c, storeApi.getState().nodes),
    [storeApi],
  );
}
```

```tsx
// components/builder/flow-canvas.tsx
"use client";

import * as React from "react";
import {
  Background,
  BackgroundVariant,
  MiniMap,
  ReactFlow,
  ReactFlowProvider,
  useReactFlow,
  type Connection,
  type EdgeChange,
  type NodeChange,
  type OnMoveEnd,
} from "@xyflow/react";
import "@xyflow/react/dist/style.css";
import { cn } from "@/lib/utils";
import { useFlowStore, useFlowStoreApi } from "@/lib/flow-store";
import type { BuilderEdge as BuilderEdgeType, BuilderNode } from "@/lib/flow-convert";
import { nodeTypes } from "./nodes/node-types";
import { BuilderEdge } from "./builder-edge";
import { useConnectionValidator } from "./use-connection-validator";

const edgeTypes = { typed: BuilderEdge };

function FlowCanvasInner({ className }: { className?: string }) {
  const nodes = useFlowStore((s) => s.nodes);
  const edges = useFlowStore((s) => s.edges);
  const mode = useFlowStore((s) => s.mode);
  const tool = useFlowStore((s) => s.tool);
  const storeApi = useFlowStoreApi();
  const validate = useConnectionValidator();
  useReactFlow(); // establishes the ReactFlow context consumers below this component rely on
  const locked = mode === "view";
  const initialViewport = React.useRef(storeApi.getState().viewport ?? { x: 0, y: 0, zoom: 1 });

  const onNodesChange = React.useCallback(
    (changes: NodeChange<BuilderNode>[]) => storeApi.getState().applyNodeChanges(changes),
    [storeApi],
  );
  const onEdgesChange = React.useCallback(
    (changes: EdgeChange<BuilderEdgeType>[]) => storeApi.getState().applyEdgeChanges(changes),
    [storeApi],
  );
  const onConnect = React.useCallback(
    (connection: Connection) => storeApi.getState().connect(connection),
    [storeApi],
  );
  const onMoveEnd: OnMoveEnd = React.useCallback(
    (_event, viewport) => storeApi.getState().setViewport(viewport),
    [storeApi],
  );

  return (
    <div className={cn("relative h-full min-h-0 w-full", className)} data-slot="flow-canvas" data-mode={mode}>
      <ReactFlow<BuilderNode, BuilderEdgeType>
        nodes={nodes}
        edges={edges}
        nodeTypes={nodeTypes}
        edgeTypes={edgeTypes}
        onNodesChange={onNodesChange}
        onEdgesChange={onEdgesChange}
        onConnect={onConnect}
        isValidConnection={validate}
        snapToGrid
        snapGrid={[16, 16]}
        panOnDrag={tool === "hand" ? true : [1, 2]}
        selectionOnDrag={tool === "select"}
        nodesDraggable={!locked}
        nodesConnectable={!locked}
        elementsSelectable={!locked}
        deleteKeyCode={locked ? null : undefined}
        fitView={mode === "view"}
        defaultViewport={initialViewport.current}
        onMoveEnd={onMoveEnd}
        proOptions={{ hideAttribution: true }}
      >
        <Background variant={BackgroundVariant.Dots} gap={16} />
        <MiniMap />
      </ReactFlow>
    </div>
  );
}

export function FlowCanvas({ className }: { className?: string }) {
  return (
    <ReactFlowProvider>
      <FlowCanvasInner className={className} />
    </ReactFlowProvider>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/flow-canvas.test.tsx`
Expected: PASS, 4 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/flow-canvas.tsx components/builder/flow-canvas.test.tsx \
  components/builder/use-connection-validator.ts
git commit -m "feat(builder): add flow-canvas core rendering and tool modes

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 8b: Flow canvas drag-and-drop and while-frame parenting

**Files:**
- Modify: `components/builder/flow-canvas.tsx`
- Modify: `components/builder/flow-canvas.test.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // lib/group-utils.ts
  export type Rect = { x: number; y: number; width: number; height: number };
  export function findWhileAt(point: XYPosition, nodes: BuilderNode[], excludeId?: string): BuilderNode | undefined;
  export function toRelativePosition(point: XYPosition, parent: BuilderNode, nodes: BuilderNode[]): XYPosition;

  // lib/flow-store.ts (FlowState, continued)
  addNode(kind: NodeKind, position: XYPosition, parentId?: string): string;
  setParent(id: string, parentId: string | undefined, position: XYPosition): void;
  ```
  and `useReactFlow().screenToFlowPosition` (from `@xyflow/react`, already in scope via Task 8a's `FlowCanvasInner`).
- Produces: `FlowCanvas` now handles `application/sat-node-kind` drops and re-parents top-level nodes dragged over a `while` frame. No new exported names.

- [ ] **Step 1: Extend the failing test**

```tsx
// components/builder/flow-canvas.test.tsx
// (append to the file written in Task 8a; the full file now reads:)
import { describe, expect, it } from "vitest";
import { fireEvent } from "@testing-library/react";
import { createFlowStore } from "@/lib/flow-store";
import { fixtureFlow, renderWithStore } from "@/lib/test-fixtures";
import { FlowCanvas } from "./flow-canvas";

describe("FlowCanvas", () => {
  it("renders one element per fixture node", () => {
    const store = createFlowStore(fixtureFlow());
    const { container } = renderWithStore(<FlowCanvas />, store);
    for (const node of store.getState().nodes) {
      expect(container.querySelector(`[data-id="${node.id}"]`)).toBeTruthy();
    }
  });

  it("renders the pane and a minimap in edit mode", () => {
    const store = createFlowStore(fixtureFlow());
    const { container } = renderWithStore(<FlowCanvas />, store);
    expect(container.querySelector(".react-flow__pane")).toBeTruthy();
    expect(container.querySelector(".react-flow__minimap")).toBeTruthy();
  });

  it("locks dragging and exposes the mode in view mode", () => {
    const store = createFlowStore(fixtureFlow(), { mode: "view" });
    const { container } = renderWithStore(<FlowCanvas />, store);
    const wrapper = container.querySelector('[data-slot="flow-canvas"]');
    expect(wrapper).toHaveAttribute("data-mode", "view");
    const node = container.querySelector(".react-flow__node");
    expect(node?.className).not.toContain("draggable");
  });

  it("leaves dragging enabled in edit mode", () => {
    const store = createFlowStore(fixtureFlow());
    const { container } = renderWithStore(<FlowCanvas />, store);
    const node = container.querySelector(".react-flow__node");
    expect(node?.className).toContain("draggable");
  });

  it("adds a node from a palette drop", () => {
    const store = createFlowStore(fixtureFlow());
    const { container } = renderWithStore(<FlowCanvas />, store);
    const pane = container.querySelector(".react-flow__pane") as Element;
    const before = store.getState().nodes.length;
    const dataTransfer = {
      getData: (mime: string) => (mime === "application/sat-node-kind" ? "agent" : ""),
      dropEffect: "",
    };
    fireEvent.dragOver(pane, { dataTransfer });
    fireEvent.drop(pane, { dataTransfer, clientX: 140, clientY: 90 });
    const added = store.getState().nodes.slice(before);
    expect(added.some((n) => n.type === "agent")).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify the new case fails**

Run: `pnpm test -- components/builder/flow-canvas.test.tsx`
Expected: FAIL with the drop test fails because no node of type `agent` is added (`onDrop` is not wired yet); the four Task 8a tests still pass.

- [ ] **Step 3: Wire drop and drag-stop handlers**

```tsx
// components/builder/flow-canvas.tsx
"use client";

import * as React from "react";
import {
  Background,
  BackgroundVariant,
  MiniMap,
  ReactFlow,
  ReactFlowProvider,
  useReactFlow,
  type Connection,
  type EdgeChange,
  type NodeChange,
  type OnMoveEnd,
  type OnNodeDrag,
} from "@xyflow/react";
import "@xyflow/react/dist/style.css";
import { cn } from "@/lib/utils";
import { useFlowStore, useFlowStoreApi } from "@/lib/flow-store";
import type { NodeKind } from "@/lib/flow-schema";
import type { BuilderEdge as BuilderEdgeType, BuilderNode } from "@/lib/flow-convert";
import { findWhileAt, toRelativePosition } from "@/lib/group-utils";
import { nodeTypes } from "./nodes/node-types";
import { BuilderEdge } from "./builder-edge";
import { useConnectionValidator } from "./use-connection-validator";

const edgeTypes = { typed: BuilderEdge };
const DRAG_MIME = "application/sat-node-kind";

function FlowCanvasInner({ className }: { className?: string }) {
  const nodes = useFlowStore((s) => s.nodes);
  const edges = useFlowStore((s) => s.edges);
  const mode = useFlowStore((s) => s.mode);
  const tool = useFlowStore((s) => s.tool);
  const storeApi = useFlowStoreApi();
  const validate = useConnectionValidator();
  const { screenToFlowPosition } = useReactFlow();
  const locked = mode === "view";
  const initialViewport = React.useRef(storeApi.getState().viewport ?? { x: 0, y: 0, zoom: 1 });

  const onNodesChange = React.useCallback(
    (changes: NodeChange<BuilderNode>[]) => storeApi.getState().applyNodeChanges(changes),
    [storeApi],
  );
  const onEdgesChange = React.useCallback(
    (changes: EdgeChange<BuilderEdgeType>[]) => storeApi.getState().applyEdgeChanges(changes),
    [storeApi],
  );
  const onConnect = React.useCallback(
    (connection: Connection) => storeApi.getState().connect(connection),
    [storeApi],
  );
  const onMoveEnd: OnMoveEnd = React.useCallback(
    (_event, viewport) => storeApi.getState().setViewport(viewport),
    [storeApi],
  );

  const onDragOver = React.useCallback((event: React.DragEvent) => {
    event.preventDefault();
    event.dataTransfer.dropEffect = "move";
  }, []);

  const onDrop = React.useCallback(
    (event: React.DragEvent) => {
      event.preventDefault();
      const kind = event.dataTransfer.getData(DRAG_MIME) as NodeKind | "";
      if (!kind) return;
      const position = screenToFlowPosition({ x: event.clientX, y: event.clientY });
      const state = storeApi.getState();
      const container = findWhileAt(position, state.nodes);
      if (container) {
        state.addNode(kind, toRelativePosition(position, container, state.nodes), container.id);
      } else {
        state.addNode(kind, position);
      }
    },
    [screenToFlowPosition, storeApi],
  );

  const onNodeDragStop: OnNodeDrag<BuilderNode> = React.useCallback(
    (_event, node) => {
      if (node.parentId) return;
      const state = storeApi.getState();
      const container = findWhileAt(node.position, state.nodes, node.id);
      if (!container) return;
      state.setParent(node.id, container.id, toRelativePosition(node.position, container, state.nodes));
    },
    [storeApi],
  );

  return (
    <div className={cn("relative h-full min-h-0 w-full", className)} data-slot="flow-canvas" data-mode={mode}>
      <ReactFlow<BuilderNode, BuilderEdgeType>
        nodes={nodes}
        edges={edges}
        nodeTypes={nodeTypes}
        edgeTypes={edgeTypes}
        onNodesChange={onNodesChange}
        onEdgesChange={onEdgesChange}
        onConnect={onConnect}
        isValidConnection={validate}
        snapToGrid
        snapGrid={[16, 16]}
        panOnDrag={tool === "hand" ? true : [1, 2]}
        selectionOnDrag={tool === "select"}
        nodesDraggable={!locked}
        nodesConnectable={!locked}
        elementsSelectable={!locked}
        deleteKeyCode={locked ? null : undefined}
        fitView={mode === "view"}
        defaultViewport={initialViewport.current}
        onMoveEnd={onMoveEnd}
        onDrop={onDrop}
        onDragOver={onDragOver}
        onNodeDragStop={onNodeDragStop}
        proOptions={{ hideAttribution: true }}
      >
        <Background variant={BackgroundVariant.Dots} gap={16} />
        <MiniMap />
      </ReactFlow>
    </div>
  );
}

export function FlowCanvas({ className }: { className?: string }) {
  return (
    <ReactFlowProvider>
      <FlowCanvasInner className={className} />
    </ReactFlowProvider>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/flow-canvas.test.tsx`
Expected: PASS, 5 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/flow-canvas.tsx components/builder/flow-canvas.test.tsx
git commit -m "feat(builder): wire flow-canvas drag-and-drop and while-frame parenting

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 9a: Node palette and FlowCanvas children slot

**Files:**
- Create: `components/builder/node-palette.tsx`
- Create: `components/builder/node-palette.test.tsx`
- Modify: `components/builder/flow-canvas.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // components/builder/node-catalog.ts
  export const PALETTE_GROUPS: { group: NodeGroup; label: string; kinds: NodeKind[] }[];
  export function catalogFor(kind: NodeKind): CatalogEntry;

  // lib/flow-store.ts
  export function useFlowStoreApi(): FlowStore;
  // FlowState.addNode(kind: NodeKind, position: XYPosition, parentId?: string): string;
  ```
  and `useReactFlow().screenToFlowPosition` from `@xyflow/react`.
- Produces:
  ```ts
  // components/builder/node-palette.tsx
  export function NodePalette(props: { className?: string }): JSX.Element;
  ```
  `FlowCanvas` gains `children?: React.ReactNode`, rendered inside its `ReactFlowProvider`/wrapper div so `NodePalette` (mounted as its child by `flow-builder.tsx` in Task 11) can call `useReactFlow()` itself.

The palette computes "the viewport center" from the DOM: it queries the nearest `.react-flow` root element's bounding rect (the class React Flow itself assigns to its wrapper) and falls back to the window's center when that rect is unavailable (unit tests under jsdom, where the rect is all-zero, still produce a deterministic position).

- [ ] **Step 1: Write the failing test**

```tsx
// components/builder/node-palette.test.tsx
import { describe, expect, it, vi } from "vitest";
import { fireEvent, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { ReactFlowProvider } from "@xyflow/react";
import { createFlowStore } from "@/lib/flow-store";
import { fixtureFlow, renderWithStore } from "@/lib/test-fixtures";
import { NodePalette } from "./node-palette";

function renderPalette(store = createFlowStore(fixtureFlow())) {
  return { store, ...renderWithStore(<ReactFlowProvider><NodePalette /></ReactFlowProvider>, store) };
}

describe("NodePalette", () => {
  it("adds a node at the viewport center on click", async () => {
    const user = userEvent.setup();
    const { store } = renderPalette();
    const before = store.getState().nodes.length;
    await user.click(screen.getByRole("button", { name: "Add Agent" }));
    expect(store.getState().nodes.length).toBe(before + 1);
    expect(store.getState().nodes.at(-1)?.type).toBe("agent");
  });

  it("sets the drag payload on dragstart", () => {
    renderPalette();
    const button = screen.getByRole("button", { name: "Add Agent" });
    const dataTransfer = { setData: vi.fn(), effectAllowed: "" };
    fireEvent.dragStart(button, { dataTransfer });
    expect(dataTransfer.setData).toHaveBeenCalledWith("application/sat-node-kind", "agent");
    expect(dataTransfer.effectAllowed).toBe("move");
  });

  it("groups items under their palette group label", () => {
    renderPalette();
    expect(screen.getByText("Core")).toBeInTheDocument();
    expect(screen.getByText("Tools")).toBeInTheDocument();
    expect(screen.getByText("Logic")).toBeInTheDocument();
    expect(screen.getByText("Data")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test -- components/builder/node-palette.test.tsx`
Expected: FAIL with `Cannot find module './node-palette'`.

- [ ] **Step 3: Implement the palette and add the children slot**

```tsx
// components/builder/node-palette.tsx
"use client";

import * as React from "react";
import { useReactFlow } from "@xyflow/react";
import { Button } from "@/components/ui/button";
import { cn } from "@/lib/utils";
import type { NodeKind } from "@/lib/flow-schema";
import { PALETTE_GROUPS, catalogFor } from "@/components/builder/node-catalog";
import { useFlowStoreApi } from "@/lib/flow-store";

export function NodePalette({ className }: { className?: string }) {
  const { screenToFlowPosition } = useReactFlow();
  const storeApi = useFlowStoreApi();

  const addAtViewportCenter = React.useCallback(
    (kind: NodeKind) => {
      const bounds = document.querySelector(".react-flow")?.getBoundingClientRect();
      const center = bounds
        ? { x: bounds.left + bounds.width / 2, y: bounds.top + bounds.height / 2 }
        : { x: window.innerWidth / 2, y: window.innerHeight / 2 };
      storeApi.getState().addNode(kind, screenToFlowPosition(center));
    },
    [screenToFlowPosition, storeApi],
  );

  const onDragStart = React.useCallback((event: React.DragEvent, kind: NodeKind) => {
    event.dataTransfer.setData("application/sat-node-kind", kind);
    event.dataTransfer.effectAllowed = "move";
  }, []);

  return (
    <div
      data-slot="node-palette"
      className={cn(
        "absolute top-4 left-4 z-10 flex max-h-[calc(100%-2rem)] w-52 flex-col gap-3 overflow-y-auto rounded-xl border border-border bg-card p-3 shadow-md",
        className,
      )}
    >
      {PALETTE_GROUPS.map((group) => (
        <div key={group.group} data-slot="palette-group" className="flex flex-col gap-1">
          <div className="px-1 text-xs font-medium text-muted-foreground">{group.label}</div>
          {group.kinds.map((kind) => {
            const entry = catalogFor(kind);
            const Icon = entry.icon;
            return (
              <Button
                key={kind}
                type="button"
                variant="ghost"
                size="sm"
                draggable
                onDragStart={(event) => onDragStart(event, kind)}
                onClick={() => addAtViewportCenter(kind)}
                className="justify-start gap-2"
                aria-label={`Add ${entry.label}`}
              >
                <Icon className="size-4" />
                {entry.label}
              </Button>
            );
          })}
        </div>
      ))}
    </div>
  );
}
```

`components/builder/flow-canvas.tsx` gains a `children` slot so the palette, toolbar and inspector mount inside its `ReactFlowProvider`. This is three surgical edits to the file Task 8b produced. Do not retype the rest of the file; its body, props and handlers are unchanged.

**Edit 1**, the inner component's signature. Find:

```tsx
function FlowCanvasInner({ className }: { className?: string }) {
```

Replace with:

```tsx
function FlowCanvasInner({ className, children }: { className?: string; children?: React.ReactNode }) {
```

**Edit 2**, render the children after `<ReactFlow>` but inside the wrapper div, so overlays position against the canvas. Find (the tail of `FlowCanvasInner`'s return):

```tsx
        <Background variant={BackgroundVariant.Dots} gap={16} />
        <MiniMap />
      </ReactFlow>
    </div>
  );
}
```

Replace with:

```tsx
        <Background variant={BackgroundVariant.Dots} gap={16} />
        <MiniMap />
      </ReactFlow>
      {children}
    </div>
  );
}
```

**Edit 3**, the exported wrapper. Find:

```tsx
export function FlowCanvas({ className }: { className?: string }) {
  return (
    <ReactFlowProvider>
      <FlowCanvasInner className={className} />
    </ReactFlowProvider>
  );
}
```

Replace with:

```tsx
export function FlowCanvas({ className, children }: { className?: string; children?: React.ReactNode }) {
  return (
    <ReactFlowProvider>
      <FlowCanvasInner className={className}>{children}</FlowCanvasInner>
    </ReactFlowProvider>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/node-palette.test.tsx components/builder/flow-canvas.test.tsx`
Expected: PASS, 3 tests in `node-palette.test.tsx`, 5 tests still passing in `flow-canvas.test.tsx`.

- [ ] **Step 5: Commit**

```bash
git add components/builder/node-palette.tsx components/builder/node-palette.test.tsx \
  components/builder/flow-canvas.tsx
git commit -m "feat(builder): add node-palette and a children slot on FlowCanvas

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 9b: Canvas toolbar

**Files:**
- Create: `components/builder/canvas-toolbar.tsx`
- Create: `components/builder/canvas-toolbar.test.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // lib/flow-store.ts
  export function useFlowStore<T>(selector: (s: FlowState) => T): T;
  export function useFlowStoreApi(): FlowStore;
  export function useFlowTemporal(): { undo: () => void; redo: () => void; canUndo: boolean; canRedo: boolean };
  // FlowState.setTool(tool: Tool): void;
  ```
- Produces:
  ```ts
  // components/builder/canvas-toolbar.tsx
  export function CanvasToolbar(props: { className?: string }): JSX.Element;
  ```
  Mounted by `flow-builder.tsx` in Task 11, as a sibling of `FlowCanvas` (not inside it, since it is not viewport-relative).

Assumes `components/ui/tooltip.tsx` (scaffolded in Task 1 via the shadcn CLI, base-nova style) exports `Tooltip`, `TooltipTrigger` (accepting `render`, per the Base UI convention already established by `DropdownMenuTrigger`/`DialogPrimitive.Close`) and `TooltipContent`, matching the `Dialog`/`DropdownMenu` Root/Trigger/Content naming convention.

- [ ] **Step 1: Write the failing test**

```tsx
// components/builder/canvas-toolbar.test.tsx
import { describe, expect, it } from "vitest";
import { act, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { createFlowStore } from "@/lib/flow-store";
import { fixtureFlow, renderWithStore } from "@/lib/test-fixtures";
import { CanvasToolbar } from "./canvas-toolbar";

describe("CanvasToolbar", () => {
  it("disables undo initially and enables it after an action", () => {
    const store = createFlowStore(fixtureFlow());
    renderWithStore(<CanvasToolbar />, store);
    expect(screen.getByRole("button", { name: "Undo" })).toBeDisabled();
    act(() => {
      store.getState().addNode("agent", { x: 0, y: 0 });
    });
    expect(screen.getByRole("button", { name: "Undo" })).not.toBeDisabled();
  });

  it("toggles aria-pressed when switching tools, defaulting to select", async () => {
    const user = userEvent.setup();
    const store = createFlowStore(fixtureFlow());
    renderWithStore(<CanvasToolbar />, store);
    const hand = screen.getByRole("button", { name: "Hand tool" });
    const select = screen.getByRole("button", { name: "Select tool" });
    expect(select).toHaveAttribute("aria-pressed", "true");
    expect(hand).toHaveAttribute("aria-pressed", "false");
    await user.click(hand);
    expect(hand).toHaveAttribute("aria-pressed", "true");
    expect(select).toHaveAttribute("aria-pressed", "false");
    expect(store.getState().tool).toBe("hand");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test -- components/builder/canvas-toolbar.test.tsx`
Expected: FAIL with `Cannot find module './canvas-toolbar'`.

- [ ] **Step 3: Implement the toolbar**

```tsx
// components/builder/canvas-toolbar.tsx
"use client";

import { Hand, MousePointer2, Redo2, Undo2 } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Tooltip, TooltipContent, TooltipTrigger } from "@/components/ui/tooltip";
import { cn } from "@/lib/utils";
import { useFlowStore, useFlowStoreApi, useFlowTemporal } from "@/lib/flow-store";

export function CanvasToolbar({ className }: { className?: string }) {
  const tool = useFlowStore((s) => s.tool);
  const storeApi = useFlowStoreApi();
  const { undo, redo, canUndo, canRedo } = useFlowTemporal();

  return (
    <div
      data-slot="canvas-toolbar"
      className={cn(
        "absolute bottom-4 left-1/2 z-10 flex -translate-x-1/2 items-center gap-1 rounded-full border border-border bg-card p-1 shadow-md",
        className,
      )}
    >
      <Tooltip>
        <TooltipTrigger
          render={
            <Button
              type="button"
              variant="ghost"
              size="icon-sm"
              aria-pressed={tool === "hand"}
              aria-label="Hand tool"
              onClick={() => storeApi.getState().setTool("hand")}
              className="aria-pressed:bg-muted"
            />
          }
        >
          <Hand className="size-4" />
        </TooltipTrigger>
        <TooltipContent>Hand (H)</TooltipContent>
      </Tooltip>
      <Tooltip>
        <TooltipTrigger
          render={
            <Button
              type="button"
              variant="ghost"
              size="icon-sm"
              aria-pressed={tool === "select"}
              aria-label="Select tool"
              onClick={() => storeApi.getState().setTool("select")}
              className="aria-pressed:bg-muted"
            />
          }
        >
          <MousePointer2 className="size-4" />
        </TooltipTrigger>
        <TooltipContent>Select (V)</TooltipContent>
      </Tooltip>
      <div className="mx-1 h-5 w-px bg-border" />
      <Tooltip>
        <TooltipTrigger
          render={
            <Button type="button" variant="ghost" size="icon-sm" aria-label="Undo" disabled={!canUndo} onClick={() => undo()} />
          }
        >
          <Undo2 className="size-4" />
        </TooltipTrigger>
        <TooltipContent>Undo (&#8984;Z)</TooltipContent>
      </Tooltip>
      <Tooltip>
        <TooltipTrigger
          render={
            <Button type="button" variant="ghost" size="icon-sm" aria-label="Redo" disabled={!canRedo} onClick={() => redo()} />
          }
        >
          <Redo2 className="size-4" />
        </TooltipTrigger>
        <TooltipContent>Redo (&#8679;&#8984;Z)</TooltipContent>
      </Tooltip>
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/canvas-toolbar.test.tsx`
Expected: PASS, 2 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/canvas-toolbar.tsx components/builder/canvas-toolbar.test.tsx
git commit -m "feat(builder): add canvas-toolbar with tool toggle and undo/redo

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 9c: Top bar core controls

**Files:**
- Create: `components/builder/top-bar.tsx`
- Create: `components/builder/top-bar.test.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // lib/flow-store.ts
  export type SaveStatus = "saved" | "draft" | "unsaved-draft" | "not-saving";
  export function saveStatus(s: FlowState): SaveStatus;
  export function useFlowStore<T>(selector: (s: FlowState) => T): T;
  export function useFlowStoreApi(): FlowStore;
  // FlowState.setMeta(patch: Partial<Pick<FlowMeta, "name" | "description">>): void;
  // FlowState.setMode(mode: Mode): void;

  // lib/flow-schema.ts
  export type Flow = { /* ...as in the skeleton... */ };
  ```
- Produces:
  ```ts
  // components/builder/top-bar.tsx
  export function TopBar(props: { slug: string; file: Flow | null }): JSX.Element;
  ```
  Extended with the overflow menu in Task 9d; mounted by `flow-builder.tsx` in Task 11.

`next/navigation`'s App Router hooks (`useRouter`, `usePathname`, `useSearchParams`) require Next's own runtime and are mocked in the Vitest environment; the plan mocks them with `vi.hoisted` so the mock functions stay inspectable from the test bodies.

- [ ] **Step 1: Write the failing test**

```tsx
// components/builder/top-bar.test.tsx
import { describe, expect, it, vi } from "vitest";
import { screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { createFlowStore } from "@/lib/flow-store";
import { fixtureFlow, renderWithStore } from "@/lib/test-fixtures";
import { TopBar } from "./top-bar";

const { mockReplace, mockPush } = vi.hoisted(() => ({ mockReplace: vi.fn(), mockPush: vi.fn() }));
vi.mock("next/navigation", () => ({
  useRouter: () => ({ replace: mockReplace, push: mockPush }),
  usePathname: () => "/flow/my-flow",
  useSearchParams: () => new URLSearchParams(),
}));

describe("TopBar", () => {
  it("shows Saved for a file-backed, untouched flow", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    renderWithStore(<TopBar slug="my-flow" file={flow} />, store);
    expect(screen.getByText("Saved")).toBeInTheDocument();
  });

  it("shows Draft after an edit to a file-backed flow", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    renderWithStore(<TopBar slug="my-flow" file={flow} />, store);
    const target = store.getState().nodes[0];
    store.getState().updateNodeData(target.id, { label: "Renamed" });
    expect(screen.getByText("Draft")).toBeInTheDocument();
  });

  it("commits the name on blur and updates the URL when switching to view mode", async () => {
    const user = userEvent.setup();
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    renderWithStore(<TopBar slug="my-flow" file={flow} />, store);
    const input = screen.getByLabelText("Flow name");
    await user.clear(input);
    await user.type(input, "Renamed flow");
    await user.tab();
    expect(store.getState().meta.name).toBe("Renamed flow");

    await user.click(screen.getByRole("button", { name: "View" }));
    expect(store.getState().mode).toBe("view");
    expect(mockReplace).toHaveBeenCalledWith("/flow/my-flow?mode=view");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test -- components/builder/top-bar.test.tsx`
Expected: FAIL with `Cannot find module './top-bar'`.

- [ ] **Step 3: Implement the core top bar**

```tsx
// components/builder/top-bar.tsx
"use client";

import * as React from "react";
import Link from "next/link";
import { usePathname, useRouter, useSearchParams } from "next/navigation";
import { ArrowLeft } from "lucide-react";
import { Badge } from "@/components/ui/badge";
import { Input } from "@/components/ui/input";
import { Tooltip, TooltipContent, TooltipTrigger } from "@/components/ui/tooltip";
import { Button } from "@/components/ui/button";
import type { Flow } from "@/lib/flow-schema";
import { saveStatus, useFlowStore, useFlowStoreApi, type SaveStatus } from "@/lib/flow-store";

const STATUS_LABEL: Record<SaveStatus, string> = {
  saved: "Saved",
  draft: "Draft",
  "unsaved-draft": "Unsaved draft",
  "not-saving": "Not saving",
};

export function TopBar({ slug, file }: { slug: string; file: Flow | null }) {
  const name = useFlowStore((s) => s.meta.name);
  const mode = useFlowStore((s) => s.mode);
  const status = useFlowStore(saveStatus);
  const storeApi = useFlowStoreApi();
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const [draftName, setDraftName] = React.useState(name);

  React.useEffect(() => {
    setDraftName(name);
  }, [name]);

  const commitName = () => {
    const trimmed = draftName.trim();
    if (trimmed && trimmed !== name) {
      storeApi.getState().setMeta({ name: trimmed });
    } else {
      setDraftName(name);
    }
  };

  const setModeAndUrl = (next: "edit" | "view") => {
    storeApi.getState().setMode(next);
    const params = new URLSearchParams(searchParams.toString());
    if (next === "view") {
      params.set("mode", "view");
    } else {
      params.delete("mode");
    }
    const query = params.toString();
    router.replace(query ? `${pathname}?${query}` : pathname);
  };

  const badge = (
    <Badge data-slot="status-badge" variant="secondary">
      {STATUS_LABEL[status]}
    </Badge>
  );

  return (
    <header data-slot="top-bar" className="flex h-12 items-center gap-3 border-b border-border bg-card px-3">
      <Button type="button" variant="ghost" size="icon-sm" render={<Link href="/" aria-label="Back to flows" />}>
        <ArrowLeft className="size-4" />
      </Button>
      <Input
        value={draftName}
        onChange={(event) => setDraftName(event.target.value)}
        onBlur={commitName}
        onKeyDown={(event) => {
          if (event.key === "Enter") event.currentTarget.blur();
        }}
        aria-label="Flow name"
        className="h-7 w-48"
      />
      {status === "not-saving" ? (
        <Tooltip>
          <TooltipTrigger render={<span />}>{badge}</TooltipTrigger>
          <TooltipContent>Browser storage is unavailable; export to keep your work</TooltipContent>
        </Tooltip>
      ) : (
        badge
      )}
      <div className="ml-auto flex items-center gap-1">
        <div className="flex rounded-lg border border-border p-0.5">
          <Button
            type="button"
            variant={mode === "edit" ? "secondary" : "ghost"}
            size="sm"
            aria-pressed={mode === "edit"}
            onClick={() => setModeAndUrl("edit")}
          >
            Edit
          </Button>
          <Button
            type="button"
            variant={mode === "view" ? "secondary" : "ghost"}
            size="sm"
            aria-pressed={mode === "view"}
            onClick={() => setModeAndUrl("view")}
          >
            View
          </Button>
        </div>
      </div>
    </header>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/top-bar.test.tsx`
Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/top-bar.tsx components/builder/top-bar.test.tsx
git commit -m "feat(builder): add top-bar name editing, status badge and mode toggle

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 9d: Top bar overflow menu

**Files:**
- Modify: `components/builder/top-bar.tsx`
- Modify: `components/builder/top-bar.test.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // lib/flow-schema.ts
  export type ParseResult = { ok: true; flow: Flow } | { ok: false; errors: string[] };
  export function parseFlow(json: unknown): ParseResult;

  // lib/flow-convert.ts
  export function canonicalFlowJson(flow: Flow): string;

  // lib/storage.ts
  export function deleteDraft(slug: string): void;

  // lib/flow-store.ts (FlowState, continued)
  exportFlow(): Flow;
  importFlow(flow: Flow): void;
  ```
- Produces: `TopBar` is now complete (Export JSON, Import JSON, Copy share link, Revert to file, Delete draft). No new exported names.

- [ ] **Step 1: Extend the failing test**

```tsx
// components/builder/top-bar.test.tsx
// (append to the file written in Task 9c; the full file now reads:)
import { describe, expect, it, vi } from "vitest";
import { screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { createFlowStore } from "@/lib/flow-store";
import { canonicalFlowJson } from "@/lib/flow-convert";
import { fixtureFlow, renderWithStore } from "@/lib/test-fixtures";
import type { Flow } from "@/lib/flow-schema";
import { TopBar } from "./top-bar";

const { mockReplace, mockPush } = vi.hoisted(() => ({ mockReplace: vi.fn(), mockPush: vi.fn() }));
vi.mock("next/navigation", () => ({
  useRouter: () => ({ replace: mockReplace, push: mockPush }),
  usePathname: () => "/flow/my-flow",
  useSearchParams: () => new URLSearchParams(),
}));

describe("TopBar", () => {
  it("shows Saved for a file-backed, untouched flow", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    renderWithStore(<TopBar slug="my-flow" file={flow} />, store);
    expect(screen.getByText("Saved")).toBeInTheDocument();
  });

  it("shows Draft after an edit to a file-backed flow", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    renderWithStore(<TopBar slug="my-flow" file={flow} />, store);
    const target = store.getState().nodes[0];
    store.getState().updateNodeData(target.id, { label: "Renamed" });
    expect(screen.getByText("Draft")).toBeInTheDocument();
  });

  it("commits the name on blur and updates the URL when switching to view mode", async () => {
    const user = userEvent.setup();
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    renderWithStore(<TopBar slug="my-flow" file={flow} />, store);
    const input = screen.getByLabelText("Flow name");
    await user.clear(input);
    await user.type(input, "Renamed flow");
    await user.tab();
    expect(store.getState().meta.name).toBe("Renamed flow");

    await user.click(screen.getByRole("button", { name: "View" }));
    expect(store.getState().mode).toBe("view");
    expect(mockReplace).toHaveBeenCalledWith("/flow/my-flow?mode=view");
  });

  it("exports a file named <slug>.json", async () => {
    const user = userEvent.setup();
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    const createObjectURL = vi.fn(() => "blob:mock");
    const revokeObjectURL = vi.fn();
    URL.createObjectURL = createObjectURL;
    URL.revokeObjectURL = revokeObjectURL;
    const realCreateElement = document.createElement.bind(document);
    const downloads: string[] = [];
    vi.spyOn(document, "createElement").mockImplementation((tag: string) => {
      const el = realCreateElement(tag);
      if (tag === "a") {
        vi.spyOn(el as HTMLAnchorElement, "click").mockImplementation(() => {
          downloads.push((el as HTMLAnchorElement).download);
        });
      }
      return el;
    });
    renderWithStore(<TopBar slug="my-flow" file={flow} />, store);
    await user.click(screen.getByRole("button", { name: "More options" }));
    await user.click(screen.getByRole("menuitem", { name: "Export JSON" }));
    expect(createObjectURL).toHaveBeenCalled();
    expect(downloads).toEqual(["my-flow.json"]);
    vi.restoreAllMocks();
  });

  it("shows the error list for invalid JSON and leaves the node count unchanged", async () => {
    const user = userEvent.setup();
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    const before = store.getState().nodes.length;
    renderWithStore(<TopBar slug="my-flow" file={flow} />, store);
    await user.click(screen.getByRole("button", { name: "More options" }));
    await user.click(screen.getByRole("menuitem", { name: "Import JSON" }));
    await user.type(screen.getByLabelText("Flow JSON text"), "{ not json");
    await user.click(screen.getByRole("button", { name: "Import" }));
    expect(await screen.findByText(/Invalid JSON/)).toBeInTheDocument();
    expect(store.getState().nodes.length).toBe(before);
  });

  it("replaces nodes when importing a valid flow", async () => {
    const user = userEvent.setup();
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    const replacement: Flow = {
      schemaVersion: 1,
      id: "replacement",
      name: "Replacement",
      source: "manual",
      createdAt: "2026-01-02T00:00:00.000Z",
      updatedAt: "2026-01-02T00:00:00.000Z",
      nodes: [{ id: "start-only", kind: "start", position: { x: 0, y: 0 }, data: { label: "Start", trigger: "manual" } }],
      edges: [],
    };
    renderWithStore(<TopBar slug="my-flow" file={flow} />, store);
    await user.click(screen.getByRole("button", { name: "More options" }));
    await user.click(screen.getByRole("menuitem", { name: "Import JSON" }));
    await user.type(screen.getByLabelText("Flow JSON text"), canonicalFlowJson(replacement));
    await user.click(screen.getByRole("button", { name: "Import" }));
    expect(store.getState().nodes).toHaveLength(1);
    expect(store.getState().nodes[0].id).toBe("start-only");
  });

  it("copies the view URL to the clipboard", async () => {
    const user = userEvent.setup();
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    const writeText = vi.fn().mockResolvedValue(undefined);
    Object.assign(navigator, { clipboard: { writeText } });
    renderWithStore(<TopBar slug="my-flow" file={flow} />, store);
    await user.click(screen.getByRole("button", { name: "More options" }));
    await user.click(screen.getByRole("menuitem", { name: "Copy share link" }));
    expect(writeText).toHaveBeenCalledWith(expect.stringContaining("/flow/my-flow?mode=view"));
  });
});
```

- [ ] **Step 2: Run test to verify the new cases fail**

Run: `pnpm test -- components/builder/top-bar.test.tsx`
Expected: FAIL with no "More options" button exists yet; the three Task 9c cases still pass.

- [ ] **Step 3: Add the overflow menu**

```tsx
// components/builder/top-bar.tsx
"use client";

import * as React from "react";
import Link from "next/link";
import { usePathname, useRouter, useSearchParams } from "next/navigation";
import { ArrowLeft, MoreVertical } from "lucide-react";
import { Badge } from "@/components/ui/badge";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Tooltip, TooltipContent, TooltipTrigger } from "@/components/ui/tooltip";
import { Button } from "@/components/ui/button";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { canonicalFlowJson } from "@/lib/flow-convert";
import { parseFlow, type Flow } from "@/lib/flow-schema";
import { deleteDraft } from "@/lib/storage";
import { saveStatus, useFlowStore, useFlowStoreApi, type SaveStatus } from "@/lib/flow-store";

const STATUS_LABEL: Record<SaveStatus, string> = {
  saved: "Saved",
  draft: "Draft",
  "unsaved-draft": "Unsaved draft",
  "not-saving": "Not saving",
};

function ImportDialog({
  open,
  onOpenChange,
  onImport,
}: {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  onImport: (flow: Flow) => void;
}) {
  const [text, setText] = React.useState("");
  const [errors, setErrors] = React.useState<string[] | null>(null);

  const handleFile = async (fileList: FileList | null) => {
    const file = fileList?.[0];
    if (!file) return;
    setText(await file.text());
  };

  const submit = () => {
    let parsed: unknown;
    try {
      parsed = JSON.parse(text);
    } catch {
      setErrors(["Invalid JSON: could not parse the input."]);
      return;
    }
    const result = parseFlow(parsed);
    if (!result.ok) {
      setErrors(result.errors);
      return;
    }
    setErrors(null);
    onImport(result.flow);
    onOpenChange(false);
    setText("");
  };

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Import flow JSON</DialogTitle>
          <DialogDescription>Choose a file or paste flow JSON below.</DialogDescription>
        </DialogHeader>
        <input
          type="file"
          accept="application/json"
          aria-label="Flow JSON file"
          onChange={(event) => void handleFile(event.target.files)}
        />
        <Textarea
          value={text}
          onChange={(event) => setText(event.target.value)}
          rows={10}
          aria-label="Flow JSON text"
          className="font-mono text-xs"
        />
        {errors && (
          <ul
            data-slot="import-errors"
            className="list-disc space-y-1 rounded-lg bg-destructive/10 p-2 pl-6 text-xs text-destructive"
          >
            {errors.map((error) => (
              <li key={error}>{error}</li>
            ))}
          </ul>
        )}
        <DialogFooter>
          <Button type="button" variant="outline" onClick={() => onOpenChange(false)}>
            Cancel
          </Button>
          <Button type="button" onClick={submit} disabled={!text.trim()}>
            Import
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}

export function TopBar({ slug, file }: { slug: string; file: Flow | null }) {
  const name = useFlowStore((s) => s.meta.name);
  const mode = useFlowStore((s) => s.mode);
  const status = useFlowStore(saveStatus);
  const storeApi = useFlowStoreApi();
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const [draftName, setDraftName] = React.useState(name);
  const [importOpen, setImportOpen] = React.useState(false);
  const [copied, setCopied] = React.useState(false);

  React.useEffect(() => {
    setDraftName(name);
  }, [name]);

  const commitName = () => {
    const trimmed = draftName.trim();
    if (trimmed && trimmed !== name) {
      storeApi.getState().setMeta({ name: trimmed });
    } else {
      setDraftName(name);
    }
  };

  const setModeAndUrl = (next: "edit" | "view") => {
    storeApi.getState().setMode(next);
    const params = new URLSearchParams(searchParams.toString());
    if (next === "view") {
      params.set("mode", "view");
    } else {
      params.delete("mode");
    }
    const query = params.toString();
    router.replace(query ? `${pathname}?${query}` : pathname);
  };

  const exportJson = () => {
    const json = canonicalFlowJson(storeApi.getState().exportFlow());
    const blob = new Blob([json], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const anchor = document.createElement("a");
    anchor.href = url;
    anchor.download = `${slug}.json`;
    anchor.click();
    URL.revokeObjectURL(url);
  };

  const copyShareLink = async () => {
    const url = `${window.location.origin}/flow/${slug}?mode=view`;
    await navigator.clipboard.writeText(url);
    setCopied(true);
    window.setTimeout(() => setCopied(false), 1500);
  };

  const revertToFile = () => {
    if (!file) return;
    storeApi.getState().importFlow(file);
    deleteDraft(slug);
  };

  const deleteDraftAndReset = () => {
    deleteDraft(slug);
    if (file) {
      storeApi.getState().importFlow(file);
    } else {
      router.push("/");
    }
  };

  const badge = (
    <Badge data-slot="status-badge" variant="secondary">
      {STATUS_LABEL[status]}
    </Badge>
  );

  return (
    <header data-slot="top-bar" className="flex h-12 items-center gap-3 border-b border-border bg-card px-3">
      <Button type="button" variant="ghost" size="icon-sm" render={<Link href="/" aria-label="Back to flows" />}>
        <ArrowLeft className="size-4" />
      </Button>
      <Input
        value={draftName}
        onChange={(event) => setDraftName(event.target.value)}
        onBlur={commitName}
        onKeyDown={(event) => {
          if (event.key === "Enter") event.currentTarget.blur();
        }}
        aria-label="Flow name"
        className="h-7 w-48"
      />
      {status === "not-saving" ? (
        <Tooltip>
          <TooltipTrigger render={<span />}>{badge}</TooltipTrigger>
          <TooltipContent>Browser storage is unavailable; export to keep your work</TooltipContent>
        </Tooltip>
      ) : (
        badge
      )}
      <div className="ml-auto flex items-center gap-1">
        <div className="flex rounded-lg border border-border p-0.5">
          <Button
            type="button"
            variant={mode === "edit" ? "secondary" : "ghost"}
            size="sm"
            aria-pressed={mode === "edit"}
            onClick={() => setModeAndUrl("edit")}
          >
            Edit
          </Button>
          <Button
            type="button"
            variant={mode === "view" ? "secondary" : "ghost"}
            size="sm"
            aria-pressed={mode === "view"}
            onClick={() => setModeAndUrl("view")}
          >
            View
          </Button>
        </div>
        <DropdownMenu>
          <DropdownMenuTrigger render={<Button type="button" variant="ghost" size="icon-sm" aria-label="More options" />}>
            <MoreVertical className="size-4" />
          </DropdownMenuTrigger>
          <DropdownMenuContent align="end">
            <DropdownMenuItem onClick={exportJson}>Export JSON</DropdownMenuItem>
            <DropdownMenuItem onClick={() => setImportOpen(true)}>Import JSON</DropdownMenuItem>
            <DropdownMenuItem onClick={() => void copyShareLink()}>
              {copied ? "Copied" : "Copy share link"}
            </DropdownMenuItem>
            <DropdownMenuSeparator />
            <DropdownMenuItem disabled={!file} onClick={revertToFile}>
              Revert to file
            </DropdownMenuItem>
            <DropdownMenuItem variant="destructive" onClick={deleteDraftAndReset}>
              Delete draft
            </DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>
      </div>
      <ImportDialog
        open={importOpen}
        onOpenChange={setImportOpen}
        onImport={(flow) => storeApi.getState().importFlow(flow)}
      />
    </header>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/top-bar.test.tsx`
Expected: PASS, 8 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/top-bar.tsx components/builder/top-bar.test.tsx
git commit -m "feat(builder): add top-bar overflow menu for export, import, share and revert

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 10a: Inspector shell, single/multi select and simple field editors

**Files:**
- Create: `components/builder/node-inspector.tsx`
- Create: `components/builder/node-inspector.test.tsx`

**Interfaces:**
- Consumes:
  ```ts
  // components/builder/node-catalog.ts
  export type FieldDescriptor =
    | { type: "text"; key: string; label: string; placeholder?: string }
    | { type: "textarea"; key: string; label: string; placeholder?: string }
    | { type: "number"; key: string; label: string; min?: number }
    | { type: "select"; key: string; label: string; options: { value: string; label: string }[] }
    | { type: "chips"; key: string; label: string }
    | { type: "string-list"; key: string; label: string; itemLabel: string; min?: number }
    | { type: "condition-list"; key: "conditions"; label: string }
    | { type: "kv-list"; key: "assignments"; label: string };
  export function catalogFor(kind: NodeKind): CatalogEntry;

  // lib/flow-store.ts
  export function useFlowStore<T>(selector: (s: FlowState) => T): T;
  export function useFlowStoreApi(): FlowStore;
  // FlowState.updateNodeData(id: string, patch: Partial<AnyKindData>): void;
  // FlowState.removeSelection(): void;
  // FlowState.duplicateSelection(): void;
  ```
- Produces:
  ```ts
  // components/builder/node-inspector.tsx
  export function NodeInspector(props: { className?: string }): JSX.Element;
  ```
  Extended with the list field editors in Task 10b; mounted by `flow-builder.tsx` in Task 11. `FieldEditor` (internal) is extended with `chips`/`string-list`/`condition-list`/`kv-list` cases in Task 10b, its `text`/`textarea`/`number`/`select` cases are final as of this task.

A universal "Label" field is rendered by the inspector shell itself, before `catalogFor(kind).fields`, since `label` lives on every kind's `Base` data and is not one of the kind-specific `FieldDescriptor` entries.

- [ ] **Step 1: Write the failing test**

```tsx
// components/builder/node-inspector.test.tsx
import { describe, expect, it } from "vitest";
import { act, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { createFlowStore } from "@/lib/flow-store";
import { fixtureFlow, renderWithStore } from "@/lib/test-fixtures";
import { NodeInspector } from "./node-inspector";

function selectOnly(store: ReturnType<typeof createFlowStore>, id: string) {
  act(() => {
    store.setState((s) => ({ nodes: s.nodes.map((n) => ({ ...n, selected: n.id === id })) }));
  });
}

describe("NodeInspector", () => {
  it("renders nothing when no node is selected", () => {
    const store = createFlowStore(fixtureFlow());
    const { container } = renderWithStore(<NodeInspector />, store);
    expect(container.querySelector('[data-slot="inspector"]')).toBeNull();
  });

  it("shows the count and deletes all selected nodes on multi-select Delete", async () => {
    const user = userEvent.setup();
    const store = createFlowStore(fixtureFlow());
    act(() => {
      store.setState((s) => ({ nodes: s.nodes.map((n) => ({ ...n, selected: true })) }));
    });
    const count = store.getState().nodes.length;
    renderWithStore(<NodeInspector />, store);
    expect(screen.getByText(`${count} nodes selected`)).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: "Delete" }));
    expect(store.getState().nodes).toHaveLength(0);
  });

  it("updates the store when the label input changes", async () => {
    const user = userEvent.setup();
    const store = createFlowStore(fixtureFlow());
    const target = store.getState().nodes.find((n) => n.type === "agent")!;
    selectOnly(store, target.id);
    renderWithStore(<NodeInspector />, store);
    const input = screen.getByLabelText("Label");
    await user.clear(input);
    await user.type(input, "Renamed agent");
    expect(store.getState().nodes.find((n) => n.id === target.id)?.data.label).toBe("Renamed agent");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm test -- components/builder/node-inspector.test.tsx`
Expected: FAIL with `Cannot find module './node-inspector'`.

- [ ] **Step 3: Implement the shell and the simple field editors**

```tsx
// components/builder/node-inspector.tsx
"use client";

import * as React from "react";
import { Copy, Trash2 } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { cn } from "@/lib/utils";
import { catalogFor, type FieldDescriptor } from "@/components/builder/node-catalog";
import type { AnyKindData } from "@/lib/flow-schema";
import type { BuilderNode } from "@/lib/flow-convert";
import { useFlowStore, useFlowStoreApi } from "@/lib/flow-store";

function LabeledField({ label, children }: { label: string; children: React.ReactNode }) {
  return (
    <label className="flex flex-col gap-1 text-xs font-medium text-muted-foreground">
      {label}
      {children}
    </label>
  );
}

function fieldValue(data: AnyKindData, key: string): unknown {
  return (data as Record<string, unknown>)[key];
}

function FieldEditor({
  field,
  data,
  onChange,
}: {
  field: FieldDescriptor;
  data: AnyKindData;
  onChange: (patch: Partial<AnyKindData>) => void;
}) {
  switch (field.type) {
    case "text":
      return (
        <LabeledField label={field.label}>
          <Input
            value={String(fieldValue(data, field.key) ?? "")}
            placeholder={field.placeholder}
            onChange={(event) => onChange({ [field.key]: event.target.value } as Partial<AnyKindData>)}
          />
        </LabeledField>
      );
    case "textarea":
      return (
        <LabeledField label={field.label}>
          <Textarea
            value={String(fieldValue(data, field.key) ?? "")}
            placeholder={field.placeholder}
            rows={4}
            onChange={(event) => onChange({ [field.key]: event.target.value } as Partial<AnyKindData>)}
          />
        </LabeledField>
      );
    case "number":
      return (
        <LabeledField label={field.label}>
          <Input
            type="number"
            min={field.min}
            value={String(fieldValue(data, field.key) ?? "")}
            onChange={(event) =>
              onChange({
                [field.key]: event.target.value === "" ? undefined : Number(event.target.value),
              } as Partial<AnyKindData>)
            }
          />
        </LabeledField>
      );
    case "select":
      return (
        <div className="flex flex-col gap-1 text-xs font-medium text-muted-foreground">
          <span>{field.label}</span>
          <Select
            value={String(fieldValue(data, field.key) ?? "")}
            onValueChange={(value) => onChange({ [field.key]: value } as Partial<AnyKindData>)}
          >
            <SelectTrigger aria-label={field.label}>
              <SelectValue />
            </SelectTrigger>
            <SelectContent>
              {field.options.map((option) => (
                <SelectItem key={option.value} value={option.value}>
                  {option.label}
                </SelectItem>
              ))}
            </SelectContent>
          </Select>
        </div>
      );
    default:
      return null; // chips, string-list, condition-list, kv-list: Task 10b
  }
}

function SingleSelectPanel({ node }: { node: BuilderNode }) {
  const storeApi = useFlowStoreApi();
  const entry = catalogFor(node.type);
  const Icon = entry.icon;

  const onFieldChange = (patch: Partial<AnyKindData>) => {
    storeApi.getState().updateNodeData(node.id, patch);
  };

  return (
    <div className="flex flex-col gap-4">
      <div className="flex items-center gap-2">
        <div
          className="flex size-8 items-center justify-center rounded-lg text-white"
          style={{ background: `var(--flow-${entry.group})` }}
        >
          <Icon className="size-4" />
        </div>
        <div className="min-w-0 flex-1">
          <div className="truncate text-sm font-medium">{node.data.label}</div>
          <div className="text-xs text-muted-foreground">{entry.label}</div>
        </div>
      </div>
      <LabeledField label="Label">
        <Input
          value={node.data.label}
          onChange={(event) => onFieldChange({ label: event.target.value } as Partial<AnyKindData>)}
        />
      </LabeledField>
      {entry.fields.map((field) => (
        <FieldEditor key={field.key} field={field} data={node.data} onChange={onFieldChange} />
      ))}
      <div className="flex gap-2 border-t border-border pt-3">
        <Button
          type="button"
          variant="outline"
          size="sm"
          className="flex-1"
          onClick={() => storeApi.getState().duplicateSelection()}
        >
          <Copy className="size-3.5" /> Duplicate
        </Button>
        <Button
          type="button"
          variant="destructive"
          size="sm"
          className="flex-1"
          onClick={() => storeApi.getState().removeSelection()}
        >
          <Trash2 className="size-3.5" /> Delete
        </Button>
      </div>
    </div>
  );
}

function MultiSelectPanel({ count, onDelete }: { count: number; onDelete: () => void }) {
  return (
    <div className="flex flex-col gap-3">
      <div className="text-sm font-medium">{count} nodes selected</div>
      <Button type="button" variant="destructive" size="sm" onClick={onDelete}>
        Delete
      </Button>
    </div>
  );
}

export function NodeInspector({ className }: { className?: string }) {
  const selection = useFlowStore((s) => s.nodes.filter((n) => n.selected));
  const storeApi = useFlowStoreApi();

  if (selection.length === 0) return null;

  return (
    <div
      data-slot="inspector"
      className={cn(
        "absolute top-4 right-4 max-h-[calc(100%-2rem)] w-80 overflow-y-auto rounded-xl border border-border bg-card p-3 shadow-md",
        className,
      )}
    >
      {selection.length > 1 ? (
        <MultiSelectPanel count={selection.length} onDelete={() => storeApi.getState().removeSelection()} />
      ) : (
        <SingleSelectPanel node={selection[0]} />
      )}
    </div>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/node-inspector.test.tsx`
Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/node-inspector.tsx components/builder/node-inspector.test.tsx
git commit -m "feat(builder): add node-inspector shell with simple field editors

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 10b: Inspector list field editors

**Files:**
- Modify: `components/builder/node-inspector.tsx`
- Modify: `components/builder/node-inspector.test.tsx`

**Interfaces:**
- Consumes: `FieldDescriptor` (unchanged, Task 10a) plus:
  ```ts
  // lib/flow-schema.ts
  export type AgentData = Base & { instructions: string; model?: string; tools?: string[] };
  export type ClassifyData = Base & { categories: string[] };
  export type IfElseData = Base & { conditions: { label: string; expression: string }[] };
  export type SetStateData = Base & { assignments: { key: string; value: string }[] };

  // components/builder/node-catalog.ts
  export function portsFor(node: BuilderNode): PortSpec[];
  ```
- Produces: `NodeInspector` and its internal `FieldEditor` are now complete for all eight `FieldDescriptor` variants. No new exported names.

- [ ] **Step 1: Extend the failing test**

```tsx
// components/builder/node-inspector.test.tsx
// (append to the file written in Task 10a; the full file now reads:)
import { describe, expect, it } from "vitest";
import { act, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { createFlowStore } from "@/lib/flow-store";
import { fixtureFlow, renderWithStore } from "@/lib/test-fixtures";
import { portsFor } from "@/components/builder/node-catalog";
import type { AgentData } from "@/lib/flow-schema";
import type { Flow } from "@/lib/flow-schema";
import { NodeInspector } from "./node-inspector";

function selectOnly(store: ReturnType<typeof createFlowStore>, id: string) {
  act(() => {
    store.setState((s) => ({ nodes: s.nodes.map((n) => ({ ...n, selected: n.id === id })) }));
  });
}

const ifElseFlow: Flow = {
  schemaVersion: 1,
  id: "if-else-fixture",
  name: "If-else fixture",
  source: "manual",
  createdAt: "2026-01-01T00:00:00.000Z",
  updatedAt: "2026-01-01T00:00:00.000Z",
  nodes: [
    {
      id: "if-1",
      kind: "if-else",
      position: { x: 0, y: 0 },
      data: { label: "Check", conditions: [{ label: "Ready", expression: "state.ready == true" }] },
    },
  ],
  edges: [],
};

const classifyFlow: Flow = {
  schemaVersion: 1,
  id: "classify-fixture",
  name: "Classify fixture",
  source: "manual",
  createdAt: "2026-01-01T00:00:00.000Z",
  updatedAt: "2026-01-01T00:00:00.000Z",
  nodes: [{ id: "classify-1", kind: "classify", position: { x: 0, y: 0 }, data: { label: "Route", categories: ["personal"] } }],
  edges: [],
};

describe("NodeInspector", () => {
  it("renders nothing when no node is selected", () => {
    const store = createFlowStore(fixtureFlow());
    const { container } = renderWithStore(<NodeInspector />, store);
    expect(container.querySelector('[data-slot="inspector"]')).toBeNull();
  });

  it("shows the count and deletes all selected nodes on multi-select Delete", async () => {
    const user = userEvent.setup();
    const store = createFlowStore(fixtureFlow());
    act(() => {
      store.setState((s) => ({ nodes: s.nodes.map((n) => ({ ...n, selected: true })) }));
    });
    const count = store.getState().nodes.length;
    renderWithStore(<NodeInspector />, store);
    expect(screen.getByText(`${count} nodes selected`)).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: "Delete" }));
    expect(store.getState().nodes).toHaveLength(0);
  });

  it("updates the store when the label input changes", async () => {
    const user = userEvent.setup();
    const store = createFlowStore(fixtureFlow());
    const target = store.getState().nodes.find((n) => n.type === "agent")!;
    selectOnly(store, target.id);
    renderWithStore(<NodeInspector />, store);
    const input = screen.getByLabelText("Label");
    await user.clear(input);
    await user.type(input, "Renamed agent");
    expect(store.getState().nodes.find((n) => n.id === target.id)?.data.label).toBe("Renamed agent");
  });

  it("adds a condition on an if-else node, increasing its out ports by one", async () => {
    const user = userEvent.setup();
    const store = createFlowStore(ifElseFlow);
    const target = store.getState().nodes[0];
    const before = portsFor(target).filter((p) => p.dir === "out").length;
    selectOnly(store, target.id);
    renderWithStore(<NodeInspector />, store);
    await user.click(screen.getByRole("button", { name: "Add condition" }));
    const updated = store.getState().nodes.find((n) => n.id === target.id)!;
    const after = portsFor(updated).filter((p) => p.dir === "out").length;
    expect(after).toBe(before + 1);
  });

  it("disables removing the last classify category", () => {
    const store = createFlowStore(classifyFlow);
    selectOnly(store, store.getState().nodes[0].id);
    renderWithStore(<NodeInspector />, store);
    expect(screen.getByRole("button", { name: "Remove Category 1" })).toBeDisabled();
  });

  it("adds a tool chip on Enter", async () => {
    const user = userEvent.setup();
    const store = createFlowStore(fixtureFlow());
    const target = store.getState().nodes.find((n) => n.type === "agent")!;
    selectOnly(store, target.id);
    renderWithStore(<NodeInspector />, store);
    const input = screen.getByPlaceholderText("Add and press Enter");
    await user.type(input, "bash{Enter}");
    const updated = store.getState().nodes.find((n) => n.id === target.id)!;
    expect((updated.data as AgentData).tools).toContain("bash");
  });

  it("duplicates the selected node", async () => {
    const user = userEvent.setup();
    const store = createFlowStore(fixtureFlow());
    const target = store.getState().nodes.find((n) => n.type === "agent")!;
    selectOnly(store, target.id);
    renderWithStore(<NodeInspector />, store);
    const before = store.getState().nodes.length;
    await user.click(screen.getByRole("button", { name: "Duplicate" }));
    expect(store.getState().nodes.length).toBe(before + 1);
  });
});
```

- [ ] **Step 2: Run test to verify the new cases fail**

Run: `pnpm test -- components/builder/node-inspector.test.tsx`
Expected: FAIL with chips/condition-list/string-list cases render nothing (`FieldEditor`'s default branch), so "Add condition", "Remove Category 1" and the chip input are not found; the three Task 10a cases still pass.

- [ ] **Step 3: Implement the list field editors**

```tsx
// components/builder/node-inspector.tsx
// Add these imports alongside the Task 10a imports:
import type { AgentData, ClassifyData, GuardrailsData, IfElseData, SetStateData } from "@/lib/flow-schema";
import { Badge } from "@/components/ui/badge";

// Add these components above FieldEditor, then extend FieldEditor's switch:

function ChipsField({
  field,
  data,
  onChange,
}: {
  field: Extract<FieldDescriptor, { type: "chips" }>;
  data: AnyKindData;
  onChange: (patch: Partial<AnyKindData>) => void;
}) {
  const values = (fieldValue(data, field.key) as string[] | undefined) ?? [];
  const [draft, setDraft] = React.useState("");

  const commit = () => {
    const value = draft.trim();
    if (!value) return;
    onChange({ [field.key]: [...values, value] } as Partial<AnyKindData>);
    setDraft("");
  };

  const remove = (index: number) => {
    onChange({ [field.key]: values.filter((_, i) => i !== index) } as Partial<AnyKindData>);
  };

  return (
    <LabeledField label={field.label}>
      <div className="flex flex-wrap gap-1.5">
        {values.map((value, index) => (
          <Badge key={`${value}-${index}`} variant="secondary" className="gap-1">
            {value}
            <button type="button" aria-label={`Remove ${value}`} onClick={() => remove(index)} className="ml-0.5">
              &times;
            </button>
          </Badge>
        ))}
      </div>
      <Input
        value={draft}
        placeholder="Add and press Enter"
        onChange={(event) => setDraft(event.target.value)}
        onKeyDown={(event) => {
          if (event.key === "Enter") {
            event.preventDefault();
            commit();
          }
        }}
      />
    </LabeledField>
  );
}

function StringListField({
  field,
  data,
  onChange,
}: {
  field: Extract<FieldDescriptor, { type: "string-list" }>;
  data: AnyKindData;
  onChange: (patch: Partial<AnyKindData>) => void;
}) {
  const values = (fieldValue(data, field.key) as string[] | undefined) ?? [];
  const min = field.min ?? 0;

  const update = (index: number, value: string) => {
    onChange({ [field.key]: values.map((v, i) => (i === index ? value : v)) } as Partial<AnyKindData>);
  };
  const add = () => {
    onChange({ [field.key]: [...values, ""] } as Partial<AnyKindData>);
  };
  const remove = (index: number) => {
    onChange({ [field.key]: values.filter((_, i) => i !== index) } as Partial<AnyKindData>);
  };

  return (
    <LabeledField label={field.label}>
      <div className="flex flex-col gap-1.5">
        {values.map((value, index) => (
          <div key={index} className="flex items-center gap-1.5">
            <Input aria-label={`${field.itemLabel} ${index + 1}`} value={value} onChange={(event) => update(index, event.target.value)} />
            <Button
              type="button"
              variant="ghost"
              size="icon-sm"
              aria-label={`Remove ${field.itemLabel} ${index + 1}`}
              disabled={values.length <= min}
              onClick={() => remove(index)}
            >
              <Trash2 className="size-3.5" />
            </Button>
          </div>
        ))}
        <Button type="button" variant="outline" size="sm" onClick={add}>
          Add {field.itemLabel}
        </Button>
      </div>
    </LabeledField>
  );
}

function ConditionListField({
  data,
  onChange,
}: {
  data: AnyKindData;
  onChange: (patch: Partial<AnyKindData>) => void;
}) {
  const conditions = (data as IfElseData).conditions;
  const min = 1;

  const update = (index: number, patch: Partial<{ label: string; expression: string }>) => {
    onChange({ conditions: conditions.map((c, i) => (i === index ? { ...c, ...patch } : c)) });
  };
  const add = () => {
    onChange({ conditions: [...conditions, { label: `Condition ${conditions.length + 1}`, expression: "" }] });
  };
  const remove = (index: number) => {
    onChange({ conditions: conditions.filter((_, i) => i !== index) });
  };

  return (
    <LabeledField label="Conditions">
      <div className="flex flex-col gap-2">
        {conditions.map((condition, index) => (
          <div key={index} className="flex flex-col gap-1 rounded-lg border border-border p-2">
            <div className="flex items-center gap-1.5">
              <Input aria-label={`Condition ${index + 1} label`} value={condition.label} onChange={(event) => update(index, { label: event.target.value })} />
              <Button
                type="button"
                variant="ghost"
                size="icon-sm"
                aria-label={`Remove condition ${index + 1}`}
                disabled={conditions.length <= min}
                onClick={() => remove(index)}
              >
                <Trash2 className="size-3.5" />
              </Button>
            </div>
            <Input
              aria-label={`Condition ${index + 1} expression`}
              value={condition.expression}
              placeholder="state.ready == true"
              className="font-mono text-xs"
              onChange={(event) => update(index, { expression: event.target.value })}
            />
          </div>
        ))}
        <Button type="button" variant="outline" size="sm" onClick={add}>
          Add condition
        </Button>
      </div>
    </LabeledField>
  );
}

function KvListField({ data, onChange }: { data: AnyKindData; onChange: (patch: Partial<AnyKindData>) => void }) {
  const assignments = (data as SetStateData).assignments;

  const update = (index: number, patch: Partial<{ key: string; value: string }>) => {
    onChange({ assignments: assignments.map((a, i) => (i === index ? { ...a, ...patch } : a)) });
  };
  const add = () => {
    onChange({ assignments: [...assignments, { key: "", value: "" }] });
  };
  const remove = (index: number) => {
    onChange({ assignments: assignments.filter((_, i) => i !== index) });
  };

  return (
    <LabeledField label="Assignments">
      <div className="flex flex-col gap-1.5">
        {assignments.map((assignment, index) => (
          <div key={index} className="flex items-center gap-1.5">
            <Input aria-label={`Assignment ${index + 1} key`} value={assignment.key} placeholder="key" onChange={(event) => update(index, { key: event.target.value })} />
            <Input aria-label={`Assignment ${index + 1} value`} value={assignment.value} placeholder="value" onChange={(event) => update(index, { value: event.target.value })} />
            <Button type="button" variant="ghost" size="icon-sm" aria-label={`Remove assignment ${index + 1}`} onClick={() => remove(index)}>
              <Trash2 className="size-3.5" />
            </Button>
          </div>
        ))}
        <Button type="button" variant="outline" size="sm" onClick={add}>
          Add assignment
        </Button>
      </div>
    </LabeledField>
  );
}

// Extend FieldEditor's switch (replacing the Task 10a `default` branch):
//   case "chips": return <ChipsField field={field} data={data} onChange={onChange} />;
//   case "string-list": return <StringListField field={field} data={data} onChange={onChange} />;
//   case "condition-list": return <ConditionListField data={data} onChange={onChange} />;
//   case "kv-list": return <KvListField data={data} onChange={onChange} />;
```

Apply the switch extension directly in `FieldEditor` (Task 10a), replacing its `default: return null;` branch:

```tsx
    case "chips":
      return <ChipsField field={field} data={data} onChange={onChange} />;
    case "string-list":
      return <StringListField field={field} data={data} onChange={onChange} />;
    case "condition-list":
      return <ConditionListField data={data} onChange={onChange} />;
    case "kv-list":
      return <KvListField data={data} onChange={onChange} />;
    default:
      return null;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm test -- components/builder/node-inspector.test.tsx`
Expected: PASS, 7 tests.

- [ ] **Step 5: Commit**

```bash
git add components/builder/node-inspector.tsx components/builder/node-inspector.test.tsx
git commit -m "feat(builder): add chips, string-list, condition-list and kv-list field editors

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```
