### Task 4: Node catalog

**Files:**
- Create: `components/builder/node-catalog.ts`
- Test: `components/builder/node-catalog.test.ts`

**Interfaces:**
- Consumes: `NODE_KINDS`, `NodeKind`, `NodeGroup`, `KindData`, `AnyKindData`, `parseFlow`, `Flow` from `lib/flow-schema.ts` (Task 3) · `HandleDir` from `components/flow-kit/flow-types.ts` (Task 2) · `BuilderNode` from `lib/flow-convert.ts` (Task 3).
- Produces (relied on by Task 5, Task 6, and Part C tasks 7-10):
  ```ts
  import type { LucideIcon } from "lucide-react";
  export type PortSpec = { dir: HandleDir; port?: string; label?: string };
  export type FieldDescriptor =
    | { type: "text"; key: string; label: string; placeholder?: string }
    | { type: "textarea"; key: string; label: string; placeholder?: string }
    | { type: "number"; key: string; label: string; min?: number }
    | { type: "select"; key: string; label: string; options: { value: string; label: string }[] }
    | { type: "chips"; key: string; label: string }             // string[]
    | { type: "string-list"; key: string; label: string; itemLabel: string; min?: number }   // string[] rows
    | { type: "condition-list"; key: "conditions"; label: string }                           // { label, expression }[]
    | { type: "kv-list"; key: "assignments"; label: string };                                // { key, value }[]
  export type Renderer = "kind" | "terminal" | "branch" | "while" | "note";
  export interface CatalogEntry<K extends NodeKind = NodeKind> {
    kind: K; label: string; group: NodeGroup; icon: LucideIcon; renderer: Renderer;
    subtitle?: (data: KindData[K]) => string | undefined;
    defaultData: () => KindData[K];
    ports: (data: KindData[K]) => PortSpec[];
    fields: FieldDescriptor[];
    defaultSize?: { width: number; height: number };   // note {220,140}, while {480,320}; others undefined
  }
  export const NODE_CATALOG: { [K in NodeKind]: CatalogEntry<K> };
  export const PALETTE_GROUPS: { group: NodeGroup; label: string; kinds: NodeKind[] }[];   // Core, Tools, Logic, Data in spec order
  export function catalogFor(kind: NodeKind): CatalogEntry;
  export function portsFor(node: BuilderNode): PortSpec[];
  export function portLabel(node: BuilderNode, port: string | undefined): string | undefined;
  ```

This task's tests rely on Vitest's `globals: true` (Task 1), no `describe`/`test`/`expect` import needed.

- [ ] **Step 1: Write the failing test file**

Create `components/builder/node-catalog.test.ts`:

```ts
import { NODE_KINDS, parseFlow, type Flow, type NodeKind } from "@/lib/flow-schema";
import type { BuilderNode } from "@/lib/flow-convert";
import { NODE_CATALOG, PALETTE_GROUPS, catalogFor, portsFor, portLabel } from "./node-catalog";

function nodeFor(kind: NodeKind): BuilderNode {
  return {
    id: `${kind}-1`,
    type: kind,
    position: { x: 0, y: 0 },
    data: catalogFor(kind).defaultData(),
  };
}

describe("NODE_CATALOG", () => {
  test("every kind in NODE_KINDS has a catalog entry with the same kind", () => {
    for (const kind of NODE_KINDS) {
      expect(NODE_CATALOG[kind].kind).toBe(kind);
    }
  });

  test("parseFlow accepts a flow containing one node of each kind built from defaultData()", () => {
    const flow: Flow = {
      schemaVersion: 1,
      id: "catalog-check",
      name: "Catalog check",
      source: "manual",
      createdAt: "2026-09-12T00:00:00.000Z",
      updatedAt: "2026-09-12T00:00:00.000Z",
      nodes: NODE_KINDS.map((kind) => {
        const n = nodeFor(kind);
        return { id: n.id, kind, position: n.position, data: n.data };
      }),
      edges: [],
    };
    const result = parseFlow(flow);
    expect(result.ok).toBe(true);
  });

  test("portsFor if-else with two conditions returns in, c0, c1, else with the condition labels", () => {
    const node = nodeFor("if-else");
    node.data = {
      label: "If/else",
      conditions: [
        { label: "Has details", expression: "input.hasDetails" },
        { label: "Is urgent", expression: "input.isUrgent" },
      ],
    };
    expect(portsFor(node)).toEqual([
      { dir: "in" },
      { dir: "out", port: "c0", label: "Has details" },
      { dir: "out", port: "c1", label: "Is urgent" },
      { dir: "out", port: "else", label: "Else" },
    ]);
  });

  test("portsFor classify with three categories returns in plus cat-0..cat-2", () => {
    const node = nodeFor("classify");
    node.data = { label: "Classify", categories: ["Personal", "Client", "Promotion"] };
    expect(portsFor(node)).toEqual([
      { dir: "in" },
      { dir: "out", port: "cat-0", label: "Personal" },
      { dir: "out", port: "cat-1", label: "Client" },
      { dir: "out", port: "cat-2", label: "Promotion" },
    ]);
  });

  test("portsFor guardrails returns pass/fail", () => {
    expect(portsFor(nodeFor("guardrails"))).toEqual([
      { dir: "in" },
      { dir: "out", port: "pass", label: "Pass" },
      { dir: "out", port: "fail", label: "Fail" },
    ]);
  });

  test("portsFor user-approval returns approve/reject", () => {
    expect(portsFor(nodeFor("user-approval"))).toEqual([
      { dir: "in" },
      { dir: "out", port: "approve", label: "Approve" },
      { dir: "out", port: "reject", label: "Reject" },
    ]);
  });

  test("portsFor note returns no ports", () => {
    expect(portsFor(nodeFor("note"))).toEqual([]);
  });

  test("portLabel returns Else for else and the condition label for c0", () => {
    const node = nodeFor("if-else");
    node.data = {
      label: "If/else",
      conditions: [{ label: "Has details", expression: "input.hasDetails" }],
    };
    expect(portLabel(node, "else")).toBe("Else");
    expect(portLabel(node, "c0")).toBe("Has details");
  });

  test("subtitle: agent falls back to Agent when model is unset", () => {
    expect(NODE_CATALOG.agent.subtitle?.({ label: "Agent", instructions: "" })).toBe("Agent");
    expect(
      NODE_CATALOG.agent.subtitle?.({ label: "Agent", instructions: "", model: "claude-sonnet-5" }),
    ).toBe("claude-sonnet-5");
  });

  test("subtitle: mcp shows the server", () => {
    expect(NODE_CATALOG.mcp.subtitle?.({ label: "MCP", server: "gh" })).toBe("gh");
  });

  test("subtitle: file-search shows the store", () => {
    expect(NODE_CATALOG["file-search"].subtitle?.({ label: "File search", store: "docs" })).toBe(
      "docs",
    );
  });

  test("subtitle: start shows the trigger label", () => {
    expect(NODE_CATALOG.start.subtitle?.({ label: "Start", trigger: "manual" })).toBe("Manual");
    expect(NODE_CATALOG.start.subtitle?.({ label: "Start", trigger: "schedule" })).toBe("Schedule");
  });

  test("subtitle: while shows the condition", () => {
    expect(NODE_CATALOG.while.subtitle?.({ label: "While", condition: "tasks remain" })).toBe(
      "tasks remain",
    );
  });

  test("subtitle: user-approval truncates the prompt to 40 chars", () => {
    const short = "Approve the plan?";
    expect(
      NODE_CATALOG["user-approval"].subtitle?.({ label: "User approval", prompt: short }),
    ).toBe(short);
    const long = "Approve the kickoff scope, budget, and timeline commitments before we proceed";
    const truncated = NODE_CATALOG["user-approval"].subtitle?.({
      label: "User approval",
      prompt: long,
    });
    expect(truncated).toHaveLength(40);
    expect(truncated).toBe(`${long.slice(0, 37)}...`);
  });

  test("subtitle: kinds without a subtitle function are undefined", () => {
    const noSubtitle = ["classify", "end", "note", "guardrails", "if-else", "transform", "set-state"] as const;
    for (const kind of noSubtitle) {
      expect(NODE_CATALOG[kind].subtitle).toBeUndefined();
    }
  });

  test("defaultSize is set only for note and while", () => {
    expect(NODE_CATALOG.note.defaultSize).toEqual({ width: 220, height: 140 });
    expect(NODE_CATALOG.while.defaultSize).toEqual({ width: 480, height: 320 });
    for (const kind of NODE_KINDS) {
      if (kind === "note" || kind === "while") continue;
      expect(NODE_CATALOG[kind].defaultSize).toBeUndefined();
    }
  });

  test("renderer assignment matches the file structure", () => {
    const renderers: Record<NodeKind, string> = {
      start: "terminal",
      end: "terminal",
      agent: "kind",
      mcp: "kind",
      "file-search": "kind",
      transform: "kind",
      "set-state": "kind",
      "user-approval": "kind",
      "if-else": "branch",
      classify: "branch",
      guardrails: "branch",
      while: "while",
      note: "note",
    };
    for (const kind of NODE_KINDS) {
      expect(NODE_CATALOG[kind].renderer).toBe(renderers[kind]);
    }
  });

  test("every field list starts with a text label field and ends with a textarea description field", () => {
    for (const kind of NODE_KINDS) {
      const fields = NODE_CATALOG[kind].fields;
      expect(fields[0]).toEqual({ type: "text", key: "label", label: "Label" });
      expect(fields[fields.length - 1]).toEqual({
        type: "textarea",
        key: "description",
        label: "Description",
      });
    }
  });

  test("classify's categories field requires at least one item", () => {
    const field = NODE_CATALOG.classify.fields.find((f) => f.key === "categories");
    expect(field).toMatchObject({ type: "string-list", min: 1 });
  });

  test("catalogFor returns the entry for a given kind", () => {
    expect(catalogFor("agent").kind).toBe("agent");
  });
});

describe("PALETTE_GROUPS", () => {
  test("lists the 13 kinds exactly once in the spec's order", () => {
    expect(PALETTE_GROUPS.flatMap((g) => g.kinds)).toEqual([...NODE_KINDS]);
  });

  test("groups the kinds as Core, Tools, Logic, Data in that order", () => {
    expect(PALETTE_GROUPS.map((g) => g.group)).toEqual(["core", "tools", "logic", "data"]);
    expect(PALETTE_GROUPS.map((g) => g.label)).toEqual(["Core", "Tools", "Logic", "Data"]);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm test -- components/builder/node-catalog.test.ts`
Expected: FAIL with `Cannot find module './node-catalog'` (the file does not exist yet).

- [ ] **Step 3: Implement the node catalog**

Create `components/builder/node-catalog.ts`:

```ts
import {
  Play,
  Bot,
  ListChecks,
  Square,
  StickyNote,
  FileSearch,
  ShieldCheck,
  Plug,
  GitBranch,
  Repeat,
  UserCheck,
  Shuffle,
  Database,
  type LucideIcon,
} from "lucide-react";
import type { HandleDir } from "@/components/flow-kit/flow-types";
import { NODE_KINDS, type NodeKind, type NodeGroup, type KindData } from "@/lib/flow-schema";
import type { BuilderNode } from "@/lib/flow-convert";

export type PortSpec = { dir: HandleDir; port?: string; label?: string };

export type FieldDescriptor =
  | { type: "text"; key: string; label: string; placeholder?: string }
  | { type: "textarea"; key: string; label: string; placeholder?: string }
  | { type: "number"; key: string; label: string; min?: number }
  | { type: "select"; key: string; label: string; options: { value: string; label: string }[] }
  | { type: "chips"; key: string; label: string }
  | { type: "string-list"; key: string; label: string; itemLabel: string; min?: number }
  | { type: "condition-list"; key: "conditions"; label: string }
  | { type: "kv-list"; key: "assignments"; label: string };

export type Renderer = "kind" | "terminal" | "branch" | "while" | "note";

export interface CatalogEntry<K extends NodeKind = NodeKind> {
  kind: K;
  label: string;
  group: NodeGroup;
  icon: LucideIcon;
  renderer: Renderer;
  subtitle?: (data: KindData[K]) => string | undefined;
  defaultData: () => KindData[K];
  ports: (data: KindData[K]) => PortSpec[];
  fields: FieldDescriptor[];
  defaultSize?: { width: number; height: number };
}

// Every kind gets the same leading label field and trailing description field; kind-specific
// fields are sandwiched between the two. Implemented once here rather than repeated 13 times.
function withCommonFields(fields: FieldDescriptor[]): FieldDescriptor[] {
  return [
    { type: "text", key: "label", label: "Label" },
    ...fields,
    { type: "textarea", key: "description", label: "Description" },
  ];
}

// Inspector shows a human label for the trigger enum; also doubles as the start node's subtitle.
const TRIGGER_LABELS: Record<KindData["start"]["trigger"], string> = {
  manual: "Manual",
  schedule: "Schedule",
  webhook: "Webhook",
  event: "Event",
};

function truncate(s: string, max: number): string {
  return s.length > max ? `${s.slice(0, max - 3)}...` : s;
}

export const NODE_CATALOG: { [K in NodeKind]: CatalogEntry<K> } = {
  start: {
    kind: "start",
    label: "Start",
    group: "core",
    icon: Play,
    renderer: "terminal",
    subtitle: (data) => TRIGGER_LABELS[data.trigger],
    defaultData: () => ({ label: "Start", trigger: "manual" }),
    ports: () => [{ dir: "out" }],
    fields: withCommonFields([
      {
        type: "select",
        key: "trigger",
        label: "Trigger",
        options: [
          { value: "manual", label: "Manual" },
          { value: "schedule", label: "Schedule" },
          { value: "webhook", label: "Webhook" },
          { value: "event", label: "Event" },
        ],
      },
      { type: "text", key: "triggerDetail", label: "Trigger detail" },
    ]),
  },

  agent: {
    kind: "agent",
    label: "Agent",
    group: "core",
    icon: Bot,
    renderer: "kind",
    subtitle: (data) => data.model ?? "Agent",
    defaultData: () => ({ label: "Agent", instructions: "" }),
    ports: () => [{ dir: "in" }, { dir: "out" }],
    fields: withCommonFields([
      { type: "textarea", key: "instructions", label: "Instructions" },
      { type: "text", key: "model", label: "Model" },
      { type: "chips", key: "tools", label: "Tools" },
    ]),
  },

  classify: {
    kind: "classify",
    label: "Classify",
    group: "core",
    icon: ListChecks,
    renderer: "branch",
    defaultData: () => ({ label: "Classify", categories: ["Category 1"] }),
    ports: (data) => [
      { dir: "in" },
      ...data.categories.map((label, i) => ({ dir: "out" as const, port: `cat-${i}`, label })),
    ],
    fields: withCommonFields([
      { type: "string-list", key: "categories", label: "Categories", itemLabel: "Category", min: 1 },
    ]),
  },

  end: {
    kind: "end",
    label: "End",
    group: "core",
    icon: Square,
    renderer: "terminal",
    defaultData: () => ({ label: "End" }),
    ports: () => [{ dir: "in" }],
    fields: withCommonFields([{ type: "text", key: "outputLabel", label: "Output label" }]),
  },

  note: {
    kind: "note",
    label: "Note",
    group: "core",
    icon: StickyNote,
    renderer: "note",
    defaultData: () => ({ label: "Note", text: "", color: "yellow" }),
    ports: () => [],
    fields: withCommonFields([
      { type: "textarea", key: "text", label: "Text" },
      {
        type: "select",
        key: "color",
        label: "Color",
        options: [
          { value: "yellow", label: "Yellow" },
          { value: "blue", label: "Blue" },
          { value: "green", label: "Green" },
          { value: "pink", label: "Pink" },
        ],
      },
    ]),
    defaultSize: { width: 220, height: 140 },
  },

  "file-search": {
    kind: "file-search",
    label: "File search",
    group: "tools",
    icon: FileSearch,
    renderer: "kind",
    subtitle: (data) => data.store,
    defaultData: () => ({ label: "File search", store: "" }),
    ports: () => [{ dir: "in" }, { dir: "out" }],
    fields: withCommonFields([
      { type: "text", key: "store", label: "Store" },
      { type: "number", key: "topK", label: "Top K", min: 1 },
    ]),
  },

  guardrails: {
    kind: "guardrails",
    label: "Guardrails",
    group: "tools",
    icon: ShieldCheck,
    renderer: "branch",
    defaultData: () => ({ label: "Guardrails", checks: [] }),
    ports: () => [
      { dir: "in" },
      { dir: "out", port: "pass", label: "Pass" },
      { dir: "out", port: "fail", label: "Fail" },
    ],
    fields: withCommonFields([
      { type: "string-list", key: "checks", label: "Checks", itemLabel: "Check" },
    ]),
  },

  mcp: {
    kind: "mcp",
    label: "MCP",
    group: "tools",
    icon: Plug,
    renderer: "kind",
    subtitle: (data) => data.server,
    defaultData: () => ({ label: "MCP", server: "" }),
    ports: () => [{ dir: "in" }, { dir: "out" }],
    fields: withCommonFields([
      { type: "text", key: "server", label: "Server" },
      { type: "chips", key: "tools", label: "Tools" },
    ]),
  },

  "if-else": {
    kind: "if-else",
    label: "If/else",
    group: "logic",
    icon: GitBranch,
    renderer: "branch",
    defaultData: () => ({
      label: "If/else",
      conditions: [{ label: "Condition 1", expression: "" }],
    }),
    ports: (data) => [
      { dir: "in" },
      ...data.conditions.map((c, i) => ({ dir: "out" as const, port: `c${i}`, label: c.label })),
      { dir: "out", port: "else", label: "Else" },
    ],
    fields: withCommonFields([{ type: "condition-list", key: "conditions", label: "Conditions" }]),
  },

  while: {
    kind: "while",
    label: "While",
    group: "logic",
    icon: Repeat,
    renderer: "while",
    subtitle: (data) => data.condition,
    defaultData: () => ({ label: "While", condition: "" }),
    ports: () => [{ dir: "in" }, { dir: "out" }],
    fields: withCommonFields([
      { type: "text", key: "condition", label: "Condition" },
      { type: "number", key: "maxIterations", label: "Max iterations", min: 1 },
    ]),
    defaultSize: { width: 480, height: 320 },
  },

  "user-approval": {
    kind: "user-approval",
    label: "User approval",
    group: "logic",
    icon: UserCheck,
    renderer: "kind",
    subtitle: (data) => truncate(data.prompt, 40),
    defaultData: () => ({ label: "User approval", prompt: "" }),
    ports: () => [
      { dir: "in" },
      { dir: "out", port: "approve", label: "Approve" },
      { dir: "out", port: "reject", label: "Reject" },
    ],
    fields: withCommonFields([{ type: "textarea", key: "prompt", label: "Prompt" }]),
  },

  transform: {
    kind: "transform",
    label: "Transform",
    group: "data",
    icon: Shuffle,
    renderer: "kind",
    defaultData: () => ({ label: "Transform", expression: "" }),
    ports: () => [{ dir: "in" }, { dir: "out" }],
    fields: withCommonFields([{ type: "textarea", key: "expression", label: "Expression" }]),
  },

  "set-state": {
    kind: "set-state",
    label: "Set state",
    group: "data",
    icon: Database,
    renderer: "kind",
    defaultData: () => ({ label: "Set state", assignments: [] }),
    ports: () => [{ dir: "in" }, { dir: "out" }],
    fields: withCommonFields([{ type: "kv-list", key: "assignments", label: "Assignments" }]),
  },
};

export const PALETTE_GROUPS: { group: NodeGroup; label: string; kinds: NodeKind[] }[] = [
  { group: "core", label: "Core", kinds: ["start", "agent", "classify", "end", "note"] },
  { group: "tools", label: "Tools", kinds: ["file-search", "guardrails", "mcp"] },
  { group: "logic", label: "Logic", kinds: ["if-else", "while", "user-approval"] },
  { group: "data", label: "Data", kinds: ["transform", "set-state"] },
];

// Sanity check that PALETTE_GROUPS and NODE_KINDS never drift apart silently.
if (process.env.NODE_ENV !== "production") {
  const flattened = PALETTE_GROUPS.flatMap((g) => g.kinds);
  if (flattened.length !== NODE_KINDS.length || !NODE_KINDS.every((k) => flattened.includes(k))) {
    console.warn("PALETTE_GROUPS is missing or duplicating a kind from NODE_KINDS");
  }
}

export function catalogFor(kind: NodeKind): CatalogEntry {
  // NODE_CATALOG[kind] is a union of CatalogEntry<K> across every K; narrowing it back to the
  // widened CatalogEntry (K defaulted to NodeKind) is the same widening portsFor/portLabel below
  // rely on, since kind is a runtime string here, not a type-level literal.
  return NODE_CATALOG[kind] as CatalogEntry;
}

export function portsFor(node: BuilderNode): PortSpec[] {
  return catalogFor(node.type).ports(node.data);
}

export function portLabel(node: BuilderNode, port: string | undefined): string | undefined {
  return portsFor(node).find((p) => p.port === port)?.label;
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm test -- components/builder/node-catalog.test.ts`
Expected: PASS, all tests green.

- [ ] **Step 5: Commit**

```bash
git add components/builder/node-catalog.ts components/builder/node-catalog.test.ts
git commit -m "$(cat <<'COMMIT_EOF'
feat(catalog): add node catalog with 13 kinds

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
COMMIT_EOF
)"
```

---

### Task 5: Store, storage, draft persistence, connection rules

Split into four sub-tasks (5a-5d) because each produces an independently reviewable and
committable file: a reviewer could accept the connection-rule matrix while still wanting
changes to the undo-coalescing logic, or accept the store while asking for changes to the
debounce timing. Build them in order. 5c depends on 5a and on Task 4's `catalogFor`; 5d
depends on 5b and 5c.

#### Task 5a: Connection rules

**Files:**
- Create: `lib/connection-rules.ts`
- Test: `lib/connection-rules.test.ts`

**Interfaces:**
- Consumes: `isValidFlowConnection`, `parseHandleId` from `components/flow-kit/flow-types.ts`
  (Task 2) · `portHandleId` from `lib/flow-handles.ts` (Task 3, test only) · `BuilderNode` from
  `lib/flow-convert.ts` (Task 3).
- Produces (relied on by Task 5c's `connect` and `duplicateSelection`, and by
  `components/builder/use-connection-validator.ts`, created in Task 6):
  ```ts
  import type { Connection, Edge } from "@xyflow/react";
  /** isValidFlowConnection(c) && source !== target && parentId(source) === parentId(target). Unknown ids -> false. */
  export function isValidBuilderConnection(c: Connection | Edge, nodes: BuilderNode[]): boolean;
  export function edgeIdFor(c: Connection): string;   // `${source}${port ? "@" + port : ""}->${target}`
  ```

- [ ] **Step 1: Write the failing test**

Create `lib/connection-rules.test.ts`:

```ts
import { isValidBuilderConnection, edgeIdFor } from "./connection-rules";
import { portHandleId } from "./flow-handles";
import type { BuilderNode } from "./flow-convert";

function node(id: string, parentId?: string): BuilderNode {
  return {
    id,
    type: "agent",
    position: { x: 0, y: 0 },
    parentId,
    data: { label: "Agent", instructions: "" },
  };
}

const nodes: BuilderNode[] = [
  node("a"),
  node("b"),
  node("while-1"),
  node("c1", "while-1"),
  node("c2", "while-1"),
  node("while-2"),
  node("d1", "while-2"),
];

describe("isValidBuilderConnection", () => {
  test("valid top-level pair", () => {
    const c = {
      source: "a",
      target: "b",
      sourceHandle: portHandleId("a", "out"),
      targetHandle: portHandleId("b", "in"),
    };
    expect(isValidBuilderConnection(c, nodes)).toBe(true);
  });

  test("self-connection rejected", () => {
    const c = {
      source: "a",
      target: "a",
      sourceHandle: portHandleId("a", "out"),
      targetHandle: portHandleId("a", "in"),
    };
    expect(isValidBuilderConnection(c, nodes)).toBe(false);
  });

  test("child-to-top-level rejected", () => {
    const c = {
      source: "c1",
      target: "a",
      sourceHandle: portHandleId("c1", "out"),
      targetHandle: portHandleId("a", "in"),
    };
    expect(isValidBuilderConnection(c, nodes)).toBe(false);
  });

  test("two children of the same while accepted", () => {
    const c = {
      source: "c1",
      target: "c2",
      sourceHandle: portHandleId("c1", "out"),
      targetHandle: portHandleId("c2", "in"),
    };
    expect(isValidBuilderConnection(c, nodes)).toBe(true);
  });

  test("children of different whiles rejected", () => {
    const c = {
      source: "c1",
      target: "d1",
      sourceHandle: portHandleId("c1", "out"),
      targetHandle: portHandleId("d1", "in"),
    };
    expect(isValidBuilderConnection(c, nodes)).toBe(false);
  });

  test("in to in rejected", () => {
    const c = {
      source: "a",
      target: "b",
      sourceHandle: portHandleId("a", "in"),
      targetHandle: portHandleId("b", "in"),
    };
    expect(isValidBuilderConnection(c, nodes)).toBe(false);
  });

  test("unknown node id rejected", () => {
    const c = {
      source: "a",
      target: "ghost",
      sourceHandle: portHandleId("a", "out"),
      targetHandle: portHandleId("ghost", "in"),
    };
    expect(isValidBuilderConnection(c, nodes)).toBe(false);
  });
});

describe("edgeIdFor", () => {
  test("without a port", () => {
    const c = {
      source: "a",
      target: "b",
      sourceHandle: portHandleId("a", "out"),
      targetHandle: portHandleId("b", "in"),
    };
    expect(edgeIdFor(c)).toBe("a->b");
  });

  test("with a port", () => {
    const c = {
      source: "a",
      target: "b",
      sourceHandle: portHandleId("a", "out", "c0"),
      targetHandle: portHandleId("b", "in"),
    };
    expect(edgeIdFor(c)).toBe("a@c0->b");
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm test -- lib/connection-rules.test.ts`
Expected: FAIL with `Cannot find module './connection-rules'`.

- [ ] **Step 3: Implement connection rules**

Create `lib/connection-rules.ts`:

```ts
import type { Connection, Edge } from "@xyflow/react";
import { isValidFlowConnection, parseHandleId } from "@/components/flow-kit/flow-types";
import type { BuilderNode } from "@/lib/flow-convert";

export function isValidBuilderConnection(c: Connection | Edge, nodes: BuilderNode[]): boolean {
  if (!isValidFlowConnection(c)) return false;
  if (c.source === c.target) return false;
  const source = nodes.find((n) => n.id === c.source);
  const target = nodes.find((n) => n.id === c.target);
  if (!source || !target) return false;
  return source.parentId === target.parentId;
}

export function edgeIdFor(c: Connection): string {
  const port = parseHandleId(c.sourceHandle)?.port;
  return `${c.source}${port ? `@${port}` : ""}->${c.target}`;
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm test -- lib/connection-rules.test.ts`
Expected: PASS, all tests green.

- [ ] **Step 5: Commit**

```bash
git add lib/connection-rules.ts lib/connection-rules.test.ts
git commit -m "$(cat <<'COMMIT_EOF'
feat(store): add builder connection validation rules

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
COMMIT_EOF
)"
```

#### Task 5b: localStorage draft adapter

**Files:**
- Create: `lib/storage.ts`
- Test: `lib/storage.test.ts`

**Interfaces:**
- Consumes: `parseFlow`, `Flow` from `lib/flow-schema.ts` (Task 3) · `canonicalFlowJson` from
  `lib/flow-convert.ts` (Task 3, test only) · `fixtureFlow` from `lib/test-fixtures.ts` (Task 3,
  test only).
- Produces (relied on by Task 5d's `bindDraftPersistence` and, later, Part D's index page and
  overflow menu):
  ```ts
  export const DRAFT_PREFIX = "sat:flow:";
  export class StorageWriteError extends Error {}
  export function draftKey(slug: string): string;
  export function isStorageAvailable(): boolean;      // try/catch around a probe write
  export function loadDraft(slug: string): Flow | null; // parseFlow; invalid or missing -> null
  export function saveDraft(flow: Flow): void;         // throws StorageWriteError
  export function deleteDraft(slug: string): void;
  export function listDrafts(): Flow[];                // by updatedAt desc; skips unparsable entries
  ```

`vitest.setup.ts` (Task 1) resets `localStorage` in a global `beforeEach`, so these tests do not
reset it themselves.

- [ ] **Step 1: Write the failing test**

Create `lib/storage.test.ts`:

```ts
import {
  DRAFT_PREFIX,
  StorageWriteError,
  deleteDraft,
  draftKey,
  isStorageAvailable,
  listDrafts,
  loadDraft,
  saveDraft,
} from "./storage";
import { canonicalFlowJson } from "./flow-convert";
import { fixtureFlow } from "./test-fixtures";

describe("storage", () => {
  test("save then load equals input via canonicalFlowJson", () => {
    const flow = fixtureFlow();
    saveDraft(flow);
    const loaded = loadDraft(flow.id);
    expect(loaded).not.toBeNull();
    expect(canonicalFlowJson(loaded!)).toBe(canonicalFlowJson(flow));
  });

  test("delete removes the draft", () => {
    const flow = fixtureFlow();
    saveDraft(flow);
    deleteDraft(flow.id);
    expect(loadDraft(flow.id)).toBeNull();
  });

  test("listDrafts is sorted by updatedAt desc", () => {
    const older = { ...fixtureFlow(), id: "draft-aaaaaa", updatedAt: "2026-01-01T00:00:00.000Z" };
    const newer = { ...fixtureFlow(), id: "draft-bbbbbb", updatedAt: "2026-06-01T00:00:00.000Z" };
    saveDraft(older);
    saveDraft(newer);
    expect(listDrafts().map((f) => f.id)).toEqual(["draft-bbbbbb", "draft-aaaaaa"]);
  });

  test("a corrupted entry is skipped", () => {
    window.localStorage.setItem(`${DRAFT_PREFIX}broken`, "{not json");
    saveDraft(fixtureFlow());
    expect(() => listDrafts()).not.toThrow();
    expect(listDrafts().some((f) => f.id === "broken")).toBe(false);
  });

  test("listDrafts skips keys without the prefix", () => {
    window.localStorage.setItem("unrelated-key", JSON.stringify({ any: "thing" }));
    saveDraft(fixtureFlow());
    expect(listDrafts().length).toBe(1);
  });

  test("draftKey adds the prefix", () => {
    expect(draftKey("ship")).toBe(`${DRAFT_PREFIX}ship`);
  });

  test("isStorageAvailable returns true under jsdom", () => {
    expect(isStorageAvailable()).toBe(true);
  });

  test("saveDraft throws StorageWriteError when localStorage.setItem throws", () => {
    const spy = vi.spyOn(Storage.prototype, "setItem").mockImplementation(() => {
      throw new Error("quota");
    });
    expect(() => saveDraft(fixtureFlow())).toThrow(StorageWriteError);
    spy.mockRestore();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm test -- lib/storage.test.ts`
Expected: FAIL with `Cannot find module './storage'`.

- [ ] **Step 3: Implement the storage adapter**

Create `lib/storage.ts`:

```ts
import { parseFlow, type Flow } from "@/lib/flow-schema";

export const DRAFT_PREFIX = "sat:flow:";

export class StorageWriteError extends Error {}

export function draftKey(slug: string): string {
  return `${DRAFT_PREFIX}${slug}`;
}

export function isStorageAvailable(): boolean {
  try {
    const probeKey = "__sat_storage_probe__";
    window.localStorage.setItem(probeKey, "1");
    window.localStorage.removeItem(probeKey);
    return true;
  } catch {
    return false;
  }
}

export function loadDraft(slug: string): Flow | null {
  try {
    const raw = window.localStorage.getItem(draftKey(slug));
    if (raw === null) return null;
    const result = parseFlow(JSON.parse(raw));
    return result.ok ? result.flow : null;
  } catch {
    return null;
  }
}

export function saveDraft(flow: Flow): void {
  try {
    window.localStorage.setItem(draftKey(flow.id), JSON.stringify(flow));
  } catch (err) {
    throw new StorageWriteError(err instanceof Error ? err.message : "failed to write draft");
  }
}

export function deleteDraft(slug: string): void {
  try {
    window.localStorage.removeItem(draftKey(slug));
  } catch {
    // Best-effort: a failing removeItem leaves the draft in place; the caller can retry.
  }
}

export function listDrafts(): Flow[] {
  const drafts: Flow[] = [];
  try {
    for (let i = 0; i < window.localStorage.length; i++) {
      const key = window.localStorage.key(i);
      if (!key || !key.startsWith(DRAFT_PREFIX)) continue;
      const raw = window.localStorage.getItem(key);
      if (raw === null) continue;
      try {
        const result = parseFlow(JSON.parse(raw));
        if (result.ok) drafts.push(result.flow);
      } catch {
        continue;
      }
    }
  } catch {
    return [];
  }
  return drafts.sort((a, b) => b.updatedAt.localeCompare(a.updatedAt));
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm test -- lib/storage.test.ts`
Expected: PASS, all tests green.

- [ ] **Step 5: Commit**

```bash
git add lib/storage.ts lib/storage.test.ts
git commit -m "$(cat <<'COMMIT_EOF'
feat(store): add localStorage draft adapter

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
COMMIT_EOF
)"
```

#### Task 5c: Flow store

**Files:**
- Create: `lib/flow-store.ts`
- Test: `lib/flow-store.test.ts`
- Modify: `lib/test-fixtures.ts` (Task 3 already exports `fixtureFlow()`; this sub-task appends
  `renderWithStore`)

**Interfaces:**
- Consumes: `Flow`, `FlowMeta`, `NodeKind`, `AnyKindData` from `lib/flow-schema.ts` (Task 3) ·
  `BuilderNode`, `BuilderEdge`, `sortParentsFirst`, `flowToGraph`, `graphToFlow`,
  `canonicalFlowJson` from `lib/flow-convert.ts` (Task 3) · `portHandleId` from
  `lib/flow-handles.ts` (Task 3) · `parseHandleId` from `components/flow-kit/flow-types.ts`
  (Task 2) · `catalogFor` from `components/builder/node-catalog.ts` (Task 4) ·
  `isValidBuilderConnection`, `edgeIdFor` from `lib/connection-rules.ts` (Task 5a) ·
  `fixtureFlow` from `lib/test-fixtures.ts` (Task 3, test only).
- Produces (relied on by Task 5d, Task 6's `use-connection-validator.ts`, and every Part C and
  Part D task that reads or mutates the graph):
  ```ts
  import type { StoreApi } from "zustand";
  import type { TemporalState } from "zundo";
  import type { Connection, EdgeChange, NodeChange, XYPosition } from "@xyflow/react";
  export type Tool = "select" | "hand";
  export type Mode = "edit" | "view";
  export type SaveStatus = "saved" | "draft" | "unsaved-draft" | "not-saving";
  export interface FlowState {
    meta: FlowMeta; nodes: BuilderNode[]; edges: BuilderEdge[]; viewport?: FlowViewport;
    mode: Mode; tool: Tool; storageStatus: "ok" | "failing";
    fileSnapshot: string | null;               // canonicalFlowJson of the committed file, null for browser-only drafts
    setMeta(patch: Partial<Pick<FlowMeta, "name" | "description">>): void;
    setMode(mode: Mode): void; setTool(tool: Tool): void; setStorageStatus(s: "ok" | "failing"): void;
    addNode(kind: NodeKind, position: XYPosition, parentId?: string): string;   // returns id `${kind}-${nanoid-like 6}`; nodes stay parent-first; new node selected, others deselected
    updateNodeData(id: string, patch: Partial<AnyKindData>): void;
    applyNodeChanges(changes: NodeChange<BuilderNode>[]): void;   // pauses temporal while any position change has dragging=true, resumes on the first with dragging=false
    applyEdgeChanges(changes: EdgeChange<BuilderEdge>[]): void;
    connect(c: Connection): void;              // no-op unless isValidBuilderConnection; edge {id: edgeIdFor(c), type:"typed", data:{port}}
    removeSelection(): void;                   // removes selected nodes, their descendants, and touching edges
    duplicateSelection(): void;                // clones selected nodes (+ descendants and internal edges) offset {x:40,y:40}, new ids, selects the clones
    setParent(id: string, parentId: string | undefined, position: XYPosition): void;  // position is relative to the new parent (or absolute when undefined)
    setViewport(v: FlowViewport): void;
    importFlow(flow: Flow): void;              // replaces meta, nodes, edges, viewport; clears temporal history
    exportFlow(): Flow;                        // graphToFlow(...) with updatedAt = new Date().toISOString()
    markFile(flow: Flow | null): void;         // sets fileSnapshot
  }
  export type FlowStore = StoreApi<FlowState> & { temporal: StoreApi<TemporalState<Partial<FlowState>>> };
  export function createFlowStore(initial: Flow, opts?: { file?: Flow | null; mode?: Mode }): FlowStore;
  export function saveStatus(s: FlowState): SaveStatus;   // not-saving if storageStatus failing; saved if fileSnapshot equals canonicalFlowJson(exportFlow() with updatedAt normalized); draft if fileSnapshot set; unsaved-draft otherwise
  export const FlowStoreContext: React.Context<FlowStore | null>;
  export function FlowStoreProvider(props: { store: FlowStore; children: React.ReactNode }): JSX.Element;
  export function useFlowStore<T>(selector: (s: FlowState) => T): T;   // throws if used outside the provider
  export function useFlowStoreApi(): FlowStore;
  export function useFlowTemporal(): { undo: () => void; redo: () => void; canUndo: boolean; canRedo: boolean };
  ```
  (`FlowViewport` is `{ x: number; y: number; zoom: number }`, exported from `lib/flow-schema.ts`,
  Task 3.)

`createFlowStore` defaults `opts.file` to `null` (an unsaved browser-only draft) when `opts` or
`opts.file` is omitted. Callers that care about save status always pass `file` explicitly; the
default only matters for tests that construct a store without caring about save status.

`applyNodeChanges` coalesces a drag gesture into exactly one undo step: a change list containing
a `position` change with `dragging: true` pauses `temporal` **before** applying (so every
intermediate move during the drag is untracked); a change list containing a `position` change
with `dragging: false` resumes `temporal` **before** applying that final change, so that one
final `set()` call is the only one diffed, capturing the whole gesture, from the position
frozen when `pause()` was first called, straight to the released position, as a single past
state.

- [ ] **Step 1: Write the failing tests**

Create `lib/flow-store.test.ts`:

```ts
import { renderHook, act } from "@testing-library/react";
import * as React from "react";
import { createFlowStore, saveStatus, useFlowStore, useFlowTemporal, FlowStoreProvider } from "./flow-store";
import { portHandleId } from "./flow-handles";
import { fixtureFlow } from "./test-fixtures";

describe("createFlowStore", () => {
  test("add then undo removes the node then redo restores it", () => {
    const store = createFlowStore(fixtureFlow());
    const before = store.getState().nodes.length;
    const id = store.getState().addNode("agent", { x: 100, y: 100 });
    expect(store.getState().nodes.some((n) => n.id === id)).toBe(true);

    store.temporal.getState().undo();
    expect(store.getState().nodes.some((n) => n.id === id)).toBe(false);
    expect(store.getState().nodes.length).toBe(before);

    store.temporal.getState().redo();
    expect(store.getState().nodes.some((n) => n.id === id)).toBe(true);
  });

  test("a drag simulated as three position changes with dragging true then one with false produces exactly one past state", () => {
    const store = createFlowStore(fixtureFlow());
    const nodeId = store.getState().nodes[0].id;
    const before = store.temporal.getState().pastStates.length;

    store.getState().applyNodeChanges([{ id: nodeId, type: "position", position: { x: 10, y: 10 }, dragging: true }]);
    store.getState().applyNodeChanges([{ id: nodeId, type: "position", position: { x: 20, y: 10 }, dragging: true }]);
    store.getState().applyNodeChanges([{ id: nodeId, type: "position", position: { x: 30, y: 10 }, dragging: true }]);
    store.getState().applyNodeChanges([{ id: nodeId, type: "position", position: { x: 30, y: 10 }, dragging: false }]);

    expect(store.temporal.getState().pastStates.length).toBe(before + 1);
  });

  test("select changes produce no past state", () => {
    const store = createFlowStore(fixtureFlow());
    const nodeId = store.getState().nodes[0].id;
    const before = store.temporal.getState().pastStates.length;

    store.getState().applyNodeChanges([{ id: nodeId, type: "select", selected: true }]);

    expect(store.temporal.getState().pastStates.length).toBe(before);
  });

  test("connect rejects a cross-boundary connection", () => {
    const store = createFlowStore(fixtureFlow());
    const whileId = store.getState().addNode("while", { x: 0, y: 0 });
    const childId = store.getState().addNode("agent", { x: 10, y: 10 }, whileId);
    const before = store.getState().edges.length;

    store.getState().connect({
      source: childId,
      target: "end-1",
      sourceHandle: portHandleId(childId, "out"),
      targetHandle: portHandleId("end-1", "in"),
    });

    expect(store.getState().edges.length).toBe(before);
  });

  test("removeSelection of a while removes its children and their edges", () => {
    const store = createFlowStore(fixtureFlow());
    const whileId = store.getState().addNode("while", { x: 0, y: 0 });
    const child1 = store.getState().addNode("agent", { x: 10, y: 10 }, whileId);
    const child2 = store.getState().addNode("transform", { x: 10, y: 120 }, whileId);
    store.getState().connect({
      source: child1,
      target: child2,
      sourceHandle: portHandleId(child1, "out"),
      targetHandle: portHandleId(child2, "in"),
    });
    expect(store.getState().edges.some((e) => e.source === child1 && e.target === child2)).toBe(true);

    store.setState((s) => ({ nodes: s.nodes.map((n) => ({ ...n, selected: n.id === whileId })) }));
    store.getState().removeSelection();

    const ids = store.getState().nodes.map((n) => n.id);
    expect(ids).not.toContain(whileId);
    expect(ids).not.toContain(child1);
    expect(ids).not.toContain(child2);
    expect(store.getState().edges.some((e) => e.source === child1 || e.target === child2)).toBe(false);
  });

  test("duplicateSelection of a while with one child produces two new nodes with the child parented to the new while and no id collisions", () => {
    const store = createFlowStore(fixtureFlow());
    const whileId = store.getState().addNode("while", { x: 0, y: 0 });
    const childId = store.getState().addNode("agent", { x: 10, y: 10 }, whileId);
    const before = store.getState().nodes.length;

    store.setState((s) => ({ nodes: s.nodes.map((n) => ({ ...n, selected: n.id === whileId })) }));
    store.getState().duplicateSelection();

    const after = store.getState().nodes;
    expect(after.length).toBe(before + 2);
    const ids = after.map((n) => n.id);
    expect(new Set(ids).size).toBe(ids.length);

    const clonedWhile = after.find((n) => n.type === "while" && n.id !== whileId)!;
    expect(clonedWhile).toBeDefined();
    const clonedChild = after.find(
      (n) => n.type === "agent" && n.id !== childId && n.parentId === clonedWhile.id,
    );
    expect(clonedChild).toBeDefined();
  });

  test("importFlow clears history", () => {
    const store = createFlowStore(fixtureFlow());
    store.getState().addNode("agent", { x: 0, y: 0 });
    expect(store.temporal.getState().pastStates.length).toBeGreaterThan(0);

    store.getState().importFlow(fixtureFlow());

    expect(store.temporal.getState().pastStates.length).toBe(0);
  });
});

describe("saveStatus", () => {
  test("returns saved right after createFlowStore(flow, { file: flow })", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    expect(saveStatus(store.getState())).toBe("saved");
  });

  test("returns draft after addNode", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    store.getState().addNode("agent", { x: 0, y: 0 });
    expect(saveStatus(store.getState())).toBe("draft");
  });

  test("returns unsaved-draft when file is null", () => {
    const store = createFlowStore(fixtureFlow(), { file: null });
    expect(saveStatus(store.getState())).toBe("unsaved-draft");
  });

  test("returns not-saving when storageStatus is failing", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow, { file: flow });
    store.getState().setStorageStatus("failing");
    expect(saveStatus(store.getState())).toBe("not-saving");
  });
});

describe("React bindings", () => {
  test("useFlowStore throws outside the provider", () => {
    expect(() => renderHook(() => useFlowStore((s) => s.mode))).toThrow(
      "useFlowStore must be used within a FlowStoreProvider",
    );
  });

  test("useFlowTemporal().canUndo flips true after an action", () => {
    const store = createFlowStore(fixtureFlow());
    const wrapper = ({ children }: { children: React.ReactNode }) =>
      React.createElement(FlowStoreProvider, { store }, children);
    const { result } = renderHook(() => useFlowTemporal(), { wrapper });

    expect(result.current.canUndo).toBe(false);

    act(() => {
      store.getState().addNode("agent", { x: 0, y: 0 });
    });

    expect(result.current.canUndo).toBe(true);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm test -- lib/flow-store.test.ts`
Expected: FAIL with `Cannot find module './flow-store'`.

- [ ] **Step 3: Implement the flow store**

Create `lib/flow-store.ts`:

```ts
"use client";
import * as React from "react";
import { createStore, type StoreApi } from "zustand/vanilla";
import { useStore } from "zustand";
import { temporal, type TemporalState } from "zundo";
import {
  applyNodeChanges as applyXyNodeChanges,
  applyEdgeChanges as applyXyEdgeChanges,
  type Connection,
  type EdgeChange,
  type NodeChange,
  type XYPosition,
} from "@xyflow/react";
import type { Flow, FlowMeta, FlowViewport, NodeKind, AnyKindData } from "@/lib/flow-schema";
import {
  type BuilderNode,
  type BuilderEdge,
  sortParentsFirst,
  flowToGraph,
  graphToFlow,
  canonicalFlowJson,
} from "@/lib/flow-convert";
import { parseHandleId } from "@/components/flow-kit/flow-types";
import { portHandleId } from "@/lib/flow-handles";
import { catalogFor } from "@/components/builder/node-catalog";
import { isValidBuilderConnection, edgeIdFor } from "@/lib/connection-rules";

export type Tool = "select" | "hand";
export type Mode = "edit" | "view";
export type SaveStatus = "saved" | "draft" | "unsaved-draft" | "not-saving";

export interface FlowState {
  meta: FlowMeta;
  nodes: BuilderNode[];
  edges: BuilderEdge[];
  viewport?: FlowViewport;
  mode: Mode;
  tool: Tool;
  storageStatus: "ok" | "failing";
  fileSnapshot: string | null;
  setMeta(patch: Partial<Pick<FlowMeta, "name" | "description">>): void;
  setMode(mode: Mode): void;
  setTool(tool: Tool): void;
  setStorageStatus(s: "ok" | "failing"): void;
  addNode(kind: NodeKind, position: XYPosition, parentId?: string): string;
  updateNodeData(id: string, patch: Partial<AnyKindData>): void;
  applyNodeChanges(changes: NodeChange<BuilderNode>[]): void;
  applyEdgeChanges(changes: EdgeChange<BuilderEdge>[]): void;
  connect(c: Connection): void;
  removeSelection(): void;
  duplicateSelection(): void;
  setParent(id: string, parentId: string | undefined, position: XYPosition): void;
  setViewport(v: FlowViewport): void;
  importFlow(flow: Flow): void;
  exportFlow(): Flow;
  markFile(flow: Flow | null): void;
}

export type FlowStore = StoreApi<FlowState> & {
  temporal: StoreApi<TemporalState<Partial<FlowState>>>;
};

function randomSuffix(): string {
  const chars = "abcdefghijklmnopqrstuvwxyz0123456789";
  let s = "";
  for (let i = 0; i < 6; i++) s += chars[Math.floor(Math.random() * chars.length)];
  return s;
}

function metaFrom(flow: Flow): FlowMeta {
  return {
    id: flow.id,
    name: flow.name,
    description: flow.description,
    source: flow.source,
    sourceRef: flow.sourceRef,
    createdAt: flow.createdAt,
    updatedAt: flow.updatedAt,
  };
}

// Selected node ids plus every descendant, computed by repeated passes until no id is added.
// Fine for wave 1's graph sizes; not worth a parent-indexed map yet.
function withDescendants(ids: Set<string>, nodes: BuilderNode[]): Set<string> {
  const result = new Set(ids);
  let changed = true;
  while (changed) {
    changed = false;
    for (const n of nodes) {
      if (n.parentId && result.has(n.parentId) && !result.has(n.id)) {
        result.add(n.id);
        changed = true;
      }
    }
  }
  return result;
}

function remapHandle(id: string | null | undefined, newNodeId: string): string | null {
  const parsed = parseHandleId(id);
  return parsed ? portHandleId(newNodeId, parsed.dir, parsed.port) : (id ?? null);
}

export function createFlowStore(
  initial: Flow,
  opts?: { file?: Flow | null; mode?: Mode },
): FlowStore {
  const graph = flowToGraph(initial);

  const store = createStore<FlowState>()(
    temporal(
      (set, get, api) => ({
        meta: metaFrom(initial),
        nodes: sortParentsFirst(graph.nodes),
        edges: graph.edges,
        viewport: initial.viewport,
        mode: opts?.mode ?? "edit",
        tool: "select",
        storageStatus: "ok",
        fileSnapshot: opts?.file ? canonicalFlowJson(opts.file) : null,

        setMeta(patch) {
          set((s) => ({ meta: { ...s.meta, ...patch } }));
        },
        setMode(mode) {
          set({ mode });
        },
        setTool(tool) {
          set({ tool });
        },
        setStorageStatus(s) {
          set({ storageStatus: s });
        },

        addNode(kind, position, parentId) {
          const id = `${kind}-${randomSuffix()}`;
          const entry = catalogFor(kind);
          const node: BuilderNode = {
            id,
            type: kind,
            position,
            data: entry.defaultData(),
            selected: true,
            ...(parentId ? { parentId, extent: "parent" as const } : {}),
            ...(entry.defaultSize
              ? { width: entry.defaultSize.width, height: entry.defaultSize.height }
              : {}),
          };
          set((s) => ({
            nodes: sortParentsFirst([...s.nodes.map((n) => ({ ...n, selected: false })), node]),
          }));
          return id;
        },

        updateNodeData(id, patch) {
          set((s) => ({
            nodes: s.nodes.map((n) => (n.id === id ? { ...n, data: { ...n.data, ...patch } } : n)),
          }));
        },

        applyNodeChanges(changes) {
          const dragStarting = changes.some((c) => c.type === "position" && c.dragging === true);
          const dragEnding = changes.some((c) => c.type === "position" && c.dragging === false);
          // pause before the mid-drag updates so they never enter history; resume before the
          // drag-end update so exactly that one set() call is diffed, from the position frozen
          // at pause() straight to the released position.
          if (dragStarting) api.temporal.getState().pause();
          if (dragEnding) api.temporal.getState().resume();
          set((s) => ({ nodes: sortParentsFirst(applyXyNodeChanges(changes, s.nodes)) }));
        },

        applyEdgeChanges(changes) {
          set((s) => ({ edges: applyXyEdgeChanges(changes, s.edges) }));
        },

        connect(c) {
          const { nodes, edges } = get();
          if (!isValidBuilderConnection(c, nodes)) return;
          const port = parseHandleId(c.sourceHandle)?.port;
          const edge: BuilderEdge = {
            id: edgeIdFor(c),
            source: c.source,
            target: c.target,
            sourceHandle: c.sourceHandle,
            targetHandle: c.targetHandle,
            type: "typed",
            data: { port },
          };
          set({ edges: [...edges, edge] });
        },

        removeSelection() {
          const { nodes, edges } = get();
          const selectedIds = withDescendants(
            new Set(nodes.filter((n) => n.selected).map((n) => n.id)),
            nodes,
          );
          if (selectedIds.size === 0) return;
          set({
            nodes: nodes.filter((n) => !selectedIds.has(n.id)),
            edges: edges.filter((e) => !selectedIds.has(e.source) && !selectedIds.has(e.target)),
          });
        },

        duplicateSelection() {
          const { nodes, edges } = get();
          const selectedIds = withDescendants(
            new Set(nodes.filter((n) => n.selected).map((n) => n.id)),
            nodes,
          );
          if (selectedIds.size === 0) return;

          const idMap = new Map<string, string>();
          for (const n of nodes) {
            if (selectedIds.has(n.id)) idMap.set(n.id, `${n.type}-${randomSuffix()}`);
          }

          const clones = sortParentsFirst(nodes.filter((n) => selectedIds.has(n.id))).map((n) => ({
            ...n,
            id: idMap.get(n.id)!,
            parentId: n.parentId ? idMap.get(n.parentId) : undefined,
            position: { x: n.position.x + 40, y: n.position.y + 40 },
            selected: true,
          }));

          const internalEdges = edges
            .filter((e) => selectedIds.has(e.source) && selectedIds.has(e.target))
            .map((e) => {
              const newSource = idMap.get(e.source)!;
              const newTarget = idMap.get(e.target)!;
              const sourceHandle = remapHandle(e.sourceHandle, newSource);
              const targetHandle = remapHandle(e.targetHandle, newTarget);
              return {
                ...e,
                id: edgeIdFor({ source: newSource, target: newTarget, sourceHandle, targetHandle }),
                source: newSource,
                target: newTarget,
                sourceHandle,
                targetHandle,
              };
            });

          set((s) => ({
            nodes: sortParentsFirst([...s.nodes.map((n) => ({ ...n, selected: false })), ...clones]),
            edges: [...s.edges, ...internalEdges],
          }));
        },

        setParent(id, parentId, position) {
          set((s) => ({
            nodes: sortParentsFirst(
              s.nodes.map((n) =>
                n.id === id
                  ? { ...n, parentId, extent: parentId ? ("parent" as const) : undefined, position }
                  : n,
              ),
            ),
          }));
        },

        setViewport(v) {
          set({ viewport: v });
        },

        importFlow(flow) {
          const g = flowToGraph(flow);
          set({
            meta: metaFrom(flow),
            nodes: sortParentsFirst(g.nodes),
            edges: g.edges,
            viewport: flow.viewport,
          });
          api.temporal.getState().clear();
        },

        exportFlow() {
          const { meta, nodes, edges, viewport } = get();
          return {
            ...graphToFlow(meta, nodes, edges, viewport),
            updatedAt: new Date().toISOString(),
          };
        },

        markFile(flow) {
          set({ fileSnapshot: flow ? canonicalFlowJson(flow) : null });
        },
      }),
      {
        limit: 100,
        partialize: (s) => ({ meta: s.meta, nodes: s.nodes, edges: s.edges }),
        equality: (a, b) =>
          canonicalFlowJson(graphToFlow(a.meta!, a.nodes!, a.edges!)) ===
          canonicalFlowJson(graphToFlow(b.meta!, b.nodes!, b.edges!)),
      },
    ),
  );

  return store as FlowStore;
}

export function saveStatus(s: FlowState): SaveStatus {
  if (s.storageStatus === "failing") return "not-saving";
  if (s.fileSnapshot === null) return "unsaved-draft";
  const current = canonicalFlowJson(graphToFlow(s.meta, s.nodes, s.edges, s.viewport));
  return current === s.fileSnapshot ? "saved" : "draft";
}

export const FlowStoreContext = React.createContext<FlowStore | null>(null);

export function FlowStoreProvider(props: {
  store: FlowStore;
  children: React.ReactNode;
}): JSX.Element {
  return React.createElement(FlowStoreContext.Provider, { value: props.store }, props.children);
}

export function useFlowStore<T>(selector: (s: FlowState) => T): T {
  const store = React.useContext(FlowStoreContext);
  if (!store) throw new Error("useFlowStore must be used within a FlowStoreProvider");
  return useStore(store, selector);
}

export function useFlowStoreApi(): FlowStore {
  const store = React.useContext(FlowStoreContext);
  if (!store) throw new Error("useFlowStoreApi must be used within a FlowStoreProvider");
  return store;
}

export function useFlowTemporal(): {
  undo: () => void;
  redo: () => void;
  canUndo: boolean;
  canRedo: boolean;
} {
  const store = useFlowStoreApi();
  const undo = useStore(store.temporal, (t) => t.undo);
  const redo = useStore(store.temporal, (t) => t.redo);
  const canUndo = useStore(store.temporal, (t) => t.pastStates.length > 0);
  const canRedo = useStore(store.temporal, (t) => t.futureStates.length > 0);
  return { undo, redo, canUndo, canRedo };
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm test -- lib/flow-store.test.ts`
Expected: PASS, all tests green.

- [ ] **Step 5: Add `renderWithStore` to the shared test fixtures**

`lib/test-fixtures.ts` already exists from Task 3 and exports `fixtureFlow()`. Add the imports
below to its existing import list, and append the `renderWithStore` function to the end of the
file, do not change `fixtureFlow` itself.

Add to the top of `lib/test-fixtures.ts`:

```ts
import { render, type RenderResult } from "@testing-library/react";
import * as React from "react";
import { ReactFlowProvider } from "@xyflow/react";
import { createFlowStore, FlowStoreProvider, type FlowStore } from "@/lib/flow-store";
```

Append to the end of `lib/test-fixtures.ts`:

```ts
export function renderWithStore(
  ui: React.ReactNode,
  store: FlowStore = createFlowStore(fixtureFlow()),
): RenderResult & { store: FlowStore } {
  const result = render(
    React.createElement(ReactFlowProvider, null, React.createElement(FlowStoreProvider, { store }, ui)),
  );
  return { ...result, store };
}
```

This does not need its own test, it is exercised by every test in Task 6 and Part C that calls
it.

- [ ] **Step 6: Run the full test suite and commit**

Run: `pnpm test -- lib/flow-store.test.ts lib/storage.test.ts lib/connection-rules.test.ts`
Expected: PASS, all tests green (confirms the `test-fixtures.ts` edit did not break the earlier
sub-tasks' tests, since they also import from it indirectly through `fixtureFlow`).

```bash
git add lib/flow-store.ts lib/flow-store.test.ts lib/test-fixtures.ts
git commit -m "$(cat <<'COMMIT_EOF'
feat(store): add flow store with undo history and save status

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
COMMIT_EOF
)"
```

#### Task 5d: Draft persistence

**Files:**
- Create: `lib/draft-persistence.ts`
- Test: `lib/draft-persistence.test.ts`

**Interfaces:**
- Consumes: `FlowStore` from `lib/flow-store.ts` (Task 5c) · `saveDraft`, `loadDraft`,
  `StorageWriteError` from `lib/storage.ts` (Task 5b) · `canonicalFlowJson` from
  `lib/flow-convert.ts` (Task 3) · `createFlowStore` and `fixtureFlow` (test only).
- Produces (relied on by Part D's `flow-builder.tsx`, which binds this in an effect for edit
  mode):
  ```ts
  export function bindDraftPersistence(store: FlowStore, slug: string, opts?: { debounceMs?: number }): () => void;
  // subscribes to the store; 500 ms after the last graph or meta change writes saveDraft(exportFlow()) under `slug`; on StorageWriteError sets storageStatus "failing"; a later success sets "ok"; returns unsubscribe
  ```

- [ ] **Step 1: Write the failing tests**

Create `lib/draft-persistence.test.ts`:

```ts
import { bindDraftPersistence } from "./draft-persistence";
import { createFlowStore } from "./flow-store";
import { loadDraft } from "./storage";
import { fixtureFlow } from "./test-fixtures";

describe("bindDraftPersistence", () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  test("writes the draft 500ms after a change", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow);
    const unsubscribe = bindDraftPersistence(store, flow.id);

    store.getState().addNode("agent", { x: 0, y: 0 });
    vi.advanceTimersByTime(500);

    const draft = loadDraft(flow.id);
    expect(draft).not.toBeNull();
    expect(draft!.nodes.length).toBe(flow.nodes.length + 1);
    unsubscribe();
  });

  test("two rapid changes write once", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow);
    const setItemSpy = vi.spyOn(Storage.prototype, "setItem");
    const unsubscribe = bindDraftPersistence(store, flow.id);

    store.getState().addNode("agent", { x: 0, y: 0 });
    vi.advanceTimersByTime(100);
    store.getState().addNode("transform", { x: 0, y: 0 });
    vi.advanceTimersByTime(500);

    expect(setItemSpy).toHaveBeenCalledTimes(1);
    unsubscribe();
    setItemSpy.mockRestore();
  });

  test("a throwing setItem flips storageStatus to failing and a later success flips it back to ok", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow);
    const unsubscribe = bindDraftPersistence(store, flow.id);
    const spy = vi.spyOn(Storage.prototype, "setItem").mockImplementation(() => {
      throw new Error("quota");
    });

    store.getState().addNode("agent", { x: 0, y: 0 });
    vi.advanceTimersByTime(500);
    expect(store.getState().storageStatus).toBe("failing");

    spy.mockRestore();
    store.getState().addNode("transform", { x: 0, y: 0 });
    vi.advanceTimersByTime(500);
    expect(store.getState().storageStatus).toBe("ok");

    unsubscribe();
  });

  test("unsubscribe stops writes", () => {
    const flow = fixtureFlow();
    const store = createFlowStore(flow);
    const unsubscribe = bindDraftPersistence(store, flow.id);
    unsubscribe();

    const setItemSpy = vi.spyOn(Storage.prototype, "setItem");
    store.getState().addNode("agent", { x: 0, y: 0 });
    vi.advanceTimersByTime(500);

    expect(setItemSpy).not.toHaveBeenCalled();
    setItemSpy.mockRestore();
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm test -- lib/draft-persistence.test.ts`
Expected: FAIL with `Cannot find module './draft-persistence'`.

- [ ] **Step 3: Implement draft persistence**

Create `lib/draft-persistence.ts`:

```ts
import { canonicalFlowJson } from "@/lib/flow-convert";
import { saveDraft, StorageWriteError } from "@/lib/storage";
import type { FlowStore } from "@/lib/flow-store";

export function bindDraftPersistence(
  store: FlowStore,
  slug: string,
  opts?: { debounceMs?: number },
): () => void {
  const debounceMs = opts?.debounceMs ?? 500;
  let timer: ReturnType<typeof setTimeout> | undefined;
  let lastWritten: string | undefined;

  const flush = () => {
    timer = undefined;
    const flow = { ...store.getState().exportFlow(), id: slug };
    const json = canonicalFlowJson(flow);
    if (json === lastWritten) return;
    try {
      saveDraft(flow);
      lastWritten = json;
      store.getState().setStorageStatus("ok");
    } catch (err) {
      if (err instanceof StorageWriteError) {
        store.getState().setStorageStatus("failing");
      } else {
        throw err;
      }
    }
  };

  const unsubscribe = store.subscribe((state, prevState) => {
    if (
      state.meta === prevState.meta &&
      state.nodes === prevState.nodes &&
      state.edges === prevState.edges
    ) {
      return;
    }
    if (timer !== undefined) clearTimeout(timer);
    timer = setTimeout(flush, debounceMs);
  });

  return () => {
    if (timer !== undefined) clearTimeout(timer);
    unsubscribe();
  };
}
```

A pure selection change still produces a new `nodes` array reference (React Flow's
`applyNodeChanges` always returns a new array), so it still reschedules the timer, but
`graphToFlow`/`fromBuilderNode` (Task 3) strip `selected` before serializing, so the recomputed
`canonicalFlowJson` equals `lastWritten` and `flush()` no-ops without touching storage. This is
why the "two rapid changes write once" test above only needs the debounce guard, not an
additional content check, to hold in general.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm test -- lib/draft-persistence.test.ts`
Expected: PASS, all tests green.

- [ ] **Step 5: Commit**

```bash
git add lib/draft-persistence.ts lib/draft-persistence.test.ts
git commit -m "$(cat <<'COMMIT_EOF'
feat(store): add debounced draft persistence

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
COMMIT_EOF
)"
```

---

### Task 6: Group utils and while-node (the drop-into-frame spike)

**Files:**
- Create: `lib/group-utils.ts`
- Test: `lib/group-utils.test.ts`
- Create: `components/builder/use-connection-validator.ts` (no dedicated test in this task, see
  the note after Step 5)
- Create: `components/builder/nodes/while-node.tsx`
- Test: `components/builder/nodes/while-node.test.tsx`

**Interfaces:**
- Consumes: `XYPosition`, `Connection`, `Edge`, `NodeProps`, `NodeResizer`, `Position` from
  `@xyflow/react` (package) · `BuilderNode` from `lib/flow-convert.ts` (Task 3) · `catalogFor`
  from `components/builder/node-catalog.ts` (Task 4) · `CONTROL` from `lib/flow-handles.ts`
  (Task 3) · `TypedHandle` from `components/flow-kit/typed-handle.tsx` (Task 2) ·
  `isValidBuilderConnection` from `lib/connection-rules.ts` (Task 5a) · `useFlowStore` from
  `lib/flow-store.ts` (Task 5c) · `WhileData` from `lib/flow-schema.ts` (Task 3) · `cn` from
  `lib/utils.ts` (Task 1) · `renderWithStore`, `fixtureFlow` from `lib/test-fixtures.ts`
  (Task 3/5c, test only).
- Produces (relied on by Task 7's other node renderers, Task 8's `flow-canvas.tsx`, and
  `components/builder/nodes/node-types.ts`):
  ```ts
  import type { XYPosition } from "@xyflow/react";
  export type Rect = { x: number; y: number; width: number; height: number };
  export function absolutePosition(node: BuilderNode, nodes: BuilderNode[]): XYPosition;   // sums parent chain
  export function nodeBounds(node: BuilderNode, nodes: BuilderNode[]): Rect;  // width/height ?? measured ?? catalog defaultSize ?? {260, 80}
  export function findWhileAt(point: XYPosition, nodes: BuilderNode[], excludeId?: string): BuilderNode | undefined;  // innermost while containing point
  export function toRelativePosition(point: XYPosition, parent: BuilderNode, nodes: BuilderNode[]): XYPosition;
  ```
  and
  ```ts
  export function useConnectionValidator(): (c: Connection | Edge) => boolean;
  ```
  and `export function WhileNode(props: NodeProps<BuilderNode>): JSX.Element;`, registered later
  under the `"while"` key of Task 7's `nodeTypes` map.

`use-connection-validator.ts` is defined here, one task earlier than its natural home among the
canvas-adjacent files, purely so `WhileNode` has something to pass to `TypedHandle`'s `validate`
prop and this task compiles standalone. **Note for Part C:** the file already exists after this
task, Task 7's other renderers and Task 8's canvas import it as-is; do not recreate it.

#### Part 1: `lib/group-utils.ts`

- [ ] **Step 1: Write the failing test**

Create `lib/group-utils.test.ts`:

```ts
import { absolutePosition, findWhileAt, nodeBounds, toRelativePosition } from "./group-utils";
import type { BuilderNode } from "./flow-convert";

function whileNode(
  id: string,
  position: { x: number; y: number },
  parentId?: string,
  size = { width: 480, height: 320 },
): BuilderNode {
  return {
    id,
    type: "while",
    position,
    parentId,
    extent: parentId ? "parent" : undefined,
    width: size.width,
    height: size.height,
    data: { label: "While", condition: "" },
  };
}

function agentNode(id: string, position: { x: number; y: number }, parentId?: string): BuilderNode {
  return { id, type: "agent", position, parentId, data: { label: "Agent", instructions: "" } };
}

describe("group-utils", () => {
  test("nested while inside while resolves the inner one", () => {
    const outer = whileNode("outer", { x: 0, y: 0 });
    const inner = whileNode("inner", { x: 40, y: 40 }, "outer", { width: 200, height: 150 });
    const nodes = [outer, inner];

    expect(findWhileAt({ x: 100, y: 100 }, nodes)?.id).toBe("inner");
  });

  test("point outside every frame returns undefined", () => {
    const outer = whileNode("outer", { x: 0, y: 0 });

    expect(findWhileAt({ x: 5000, y: 5000 }, [outer])).toBeUndefined();
  });

  test("a while cannot be its own drop target", () => {
    const outer = whileNode("outer", { x: 0, y: 0 });

    expect(findWhileAt({ x: 100, y: 100 }, [outer], "outer")).toBeUndefined();
  });

  test("a while cannot drop into its own descendant", () => {
    const outer = whileNode("outer", { x: 0, y: 0 });
    const inner = whileNode("inner", { x: 40, y: 40 }, "outer", { width: 200, height: 150 });
    const nodes = [outer, inner];

    expect(findWhileAt({ x: 100, y: 100 }, nodes, "outer")).toBeUndefined();
  });

  test("relative conversion round-trips with absolutePosition", () => {
    const outer = whileNode("outer", { x: 50, y: 60 });
    const child = agentNode("child", { x: 10, y: 20 }, "outer");
    const nodes = [outer, child];

    const abs = absolutePosition(child, nodes);
    expect(abs).toEqual({ x: 60, y: 80 });

    const rel = toRelativePosition(abs, outer, nodes);
    expect(rel).toEqual(child.position);
  });

  test("nodeBounds falls back to catalog defaultSize when width/height are unset", () => {
    const bare: BuilderNode = {
      id: "bare",
      type: "while",
      position: { x: 0, y: 0 },
      data: { label: "While", condition: "" },
    };

    expect(nodeBounds(bare, [bare])).toEqual({ x: 0, y: 0, width: 480, height: 320 });
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm test -- lib/group-utils.test.ts`
Expected: FAIL with `Cannot find module './group-utils'`.

- [ ] **Step 3: Implement group utils**

Create `lib/group-utils.ts`:

```ts
import type { XYPosition } from "@xyflow/react";
import { catalogFor } from "@/components/builder/node-catalog";
import type { BuilderNode } from "@/lib/flow-convert";

export type Rect = { x: number; y: number; width: number; height: number };

export function absolutePosition(node: BuilderNode, nodes: BuilderNode[]): XYPosition {
  let x = node.position.x;
  let y = node.position.y;
  let current = node;
  const seen = new Set<string>([node.id]);
  while (current.parentId) {
    const parent = nodes.find((n) => n.id === current.parentId);
    if (!parent || seen.has(parent.id)) break;
    seen.add(parent.id);
    x += parent.position.x;
    y += parent.position.y;
    current = parent;
  }
  return { x, y };
}

export function nodeBounds(node: BuilderNode, nodes: BuilderNode[]): Rect {
  const { x, y } = absolutePosition(node, nodes);
  const fallback = catalogFor(node.type).defaultSize ?? { width: 260, height: 80 };
  const width = node.width ?? node.measured?.width ?? fallback.width;
  const height = node.height ?? node.measured?.height ?? fallback.height;
  return { x, y, width, height };
}

export function findWhileAt(
  point: XYPosition,
  nodes: BuilderNode[],
  excludeId?: string,
): BuilderNode | undefined {
  const excluded = new Set<string>();
  if (excludeId) {
    excluded.add(excludeId);
    let changed = true;
    while (changed) {
      changed = false;
      for (const n of nodes) {
        if (n.parentId && excluded.has(n.parentId) && !excluded.has(n.id)) {
          excluded.add(n.id);
          changed = true;
        }
      }
    }
  }

  let best: BuilderNode | undefined;
  let bestArea = Infinity;
  for (const candidate of nodes) {
    if (candidate.type !== "while" || excluded.has(candidate.id)) continue;
    const bounds = nodeBounds(candidate, nodes);
    const inside =
      point.x >= bounds.x &&
      point.x <= bounds.x + bounds.width &&
      point.y >= bounds.y &&
      point.y <= bounds.y + bounds.height;
    if (!inside) continue;
    // Nested frames are strictly contained within their ancestors, so the smallest matching
    // area is always the innermost one.
    const area = bounds.width * bounds.height;
    if (area < bestArea) {
      best = candidate;
      bestArea = area;
    }
  }
  return best;
}

export function toRelativePosition(
  point: XYPosition,
  parent: BuilderNode,
  nodes: BuilderNode[],
): XYPosition {
  const parentAbs = absolutePosition(parent, nodes);
  return { x: point.x - parentAbs.x, y: point.y - parentAbs.y };
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm test -- lib/group-utils.test.ts`
Expected: PASS, all tests green.

#### Part 2: `use-connection-validator.ts` and `while-node.tsx`

- [ ] **Step 5: Create the connection-validator hook**

Create `components/builder/use-connection-validator.ts`:

```ts
"use client";
import { useCallback } from "react";
import type { Connection, Edge } from "@xyflow/react";
import { isValidBuilderConnection } from "@/lib/connection-rules";
import { useFlowStore } from "@/lib/flow-store";

export function useConnectionValidator(): (c: Connection | Edge) => boolean {
  const nodes = useFlowStore((s) => s.nodes);
  return useCallback((c: Connection | Edge) => isValidBuilderConnection(c, nodes), [nodes]);
}
```

No dedicated test file for this hook in this task, it has no branching logic of its own (it is
a thin, memoized wrapper around `isValidBuilderConnection`, already covered by Task 5a's tests),
and `while-node.test.tsx` below exercises it for real through `renderWithStore`'s live
`FlowStoreProvider` rather than a mock.

- [ ] **Step 6: Write the failing tests for `while-node.tsx`**

Create `components/builder/nodes/while-node.test.tsx`:

```tsx
import { screen } from "@testing-library/react";
import type { NodeProps } from "@xyflow/react";
import { createFlowStore } from "@/lib/flow-store";
import { fixtureFlow, renderWithStore } from "@/lib/test-fixtures";
import type { BuilderNode } from "@/lib/flow-convert";
import { WhileNode } from "./while-node";

function props(overrides: Partial<NodeProps<BuilderNode>> = {}): NodeProps<BuilderNode> {
  return {
    id: "w1",
    type: "while",
    data: { label: "While", condition: "has more tasks" },
    selected: false,
    dragging: false,
    zIndex: 0,
    selectable: true,
    deletable: true,
    draggable: true,
    isConnectable: true,
    positionAbsoluteX: 0,
    positionAbsoluteY: 0,
    ...overrides,
  };
}

describe("WhileNode", () => {
  test("renders the label and condition", () => {
    renderWithStore(<WhileNode {...props()} />);

    expect(screen.getByText("While")).toBeInTheDocument();
    expect(screen.getByText("has more tasks")).toBeInTheDocument();
  });

  test("shows Drop nodes here when empty", () => {
    renderWithStore(<WhileNode {...props()} />);

    expect(screen.getByText("Drop nodes here")).toBeInTheDocument();
  });

  test("hides the empty state when a child exists in the store", () => {
    const store = createFlowStore(fixtureFlow());
    store.getState().addNode("agent", { x: 10, y: 10 }, "w1");

    renderWithStore(<WhileNode {...props()} />, store);

    expect(screen.queryByText("Drop nodes here")).not.toBeInTheDocument();
  });

  test("renders exactly two handles with the expected ids", () => {
    const { container } = renderWithStore(<WhileNode {...props()} />);

    expect(container.querySelectorAll("[data-handleid]")).toHaveLength(2);
    expect(container.querySelector('[data-handleid="w1:control:in"]')).not.toBeNull();
    expect(container.querySelector('[data-handleid="w1:control:out"]')).not.toBeNull();
  });

  test("shows the resizer when selected", () => {
    const { container } = renderWithStore(<WhileNode {...props({ selected: true })} />);

    expect(container.querySelectorAll(".react-flow__resize-control").length).toBeGreaterThan(0);
  });

  test("hides the resizer when not selected", () => {
    const { container } = renderWithStore(<WhileNode {...props({ selected: false })} />);

    expect(container.querySelectorAll(".react-flow__resize-control")).toHaveLength(0);
  });
});
```

- [ ] **Step 7: Run the tests to verify they fail**

Run: `pnpm test -- components/builder/nodes/while-node.test.tsx`
Expected: FAIL with `Cannot find module './while-node'`.

- [ ] **Step 8: Implement the while-node renderer**

Create `components/builder/nodes/while-node.tsx`:

```tsx
"use client";
import { Repeat } from "lucide-react";
import { NodeResizer, Position, type NodeProps } from "@xyflow/react";
import { TypedHandle } from "@/components/flow-kit/typed-handle";
import { CONTROL } from "@/lib/flow-handles";
import { useFlowStore } from "@/lib/flow-store";
import { useConnectionValidator } from "@/components/builder/use-connection-validator";
import type { BuilderNode } from "@/lib/flow-convert";
import type { WhileData } from "@/lib/flow-schema";
import { cn } from "@/lib/utils";

export function WhileNode({ id, data, selected }: NodeProps<BuilderNode>) {
  const hasChildren = useFlowStore((s) => s.nodes.some((n) => n.parentId === id));
  const validate = useConnectionValidator();
  const whileData = data as WhileData;

  return (
    <div
      data-slot="node"
      data-kind="while"
      className={cn(
        "flex h-full w-full flex-col overflow-hidden rounded-2xl border-2 border-dashed border-border",
        selected && "ring-2 ring-ring",
      )}
    >
      <NodeResizer nodeId={id} minWidth={320} minHeight={200} isVisible={selected} />

      <TypedHandle
        nodeId={id}
        type="target"
        dataType={CONTROL}
        position={Position.Left}
        validate={validate}
        aria-label="Control input port"
      />

      <div
        data-slot="node-header"
        className="flex items-center gap-2 bg-card px-3 py-2"
      >
        <Repeat className="size-4 text-muted-foreground" aria-hidden="true" />
        <span data-slot="node-title" className="text-sm font-medium">
          {whileData.label}
        </span>
        <span className="truncate text-xs text-muted-foreground">{whileData.condition}</span>
      </div>

      <div
        data-slot="while-body"
        className="flex flex-1 items-center justify-center bg-muted/30 p-4 text-xs text-muted-foreground"
      >
        {!hasChildren && "Drop nodes here"}
      </div>

      <TypedHandle
        nodeId={id}
        type="source"
        dataType={CONTROL}
        position={Position.Right}
        validate={validate}
        aria-label="Control output port"
      />
    </div>
  );
}
```

- [ ] **Step 9: Run the tests to verify they pass**

Run: `pnpm test -- components/builder/nodes/while-node.test.tsx`
Expected: PASS, all tests green.

- [ ] **Step 10: Record the spike findings**

This task cannot run a dev server yet, there is no scaffolded canvas until Task 8. Instead it
leaves this checklist for Task 8's implementer to physically confirm the first time
`FlowCanvas` renders a `while` node with children, by running `pnpm dev`, opening
`/flow/subagent-build-loop` (its seed flow, added in Task 12, has a populated `while`), and
performing each step below.

**Findings: React Flow 12 group-node spike (confirm in Task 8)**

1. **Children cannot leave the frame.**
   Steps: drag a node whose `parentId` is a `while` id past the frame's dashed border and
   release.
   Expected: the node's position stops at the frame's inner edge; it never renders outside
   the dashed border. This follows from `extent: "parent"` on every child node (set by
   `toBuilderNode` in `lib/flow-convert.ts`, Task 3, and by `setParent` in `lib/flow-store.ts`,
   Task 5c), React Flow clamps a dragged node's position to its parent's measured bounds
   whenever `extent` is `"parent"`.

2. **Moving the parent moves its children.**
   Steps: drag the `while` frame itself by its header strip (not a resize handle) 200px right.
   Expected: every child node moves with it, keeping the same position relative to the frame.
   This needs no code in `lib/flow-store.ts`, it is built into React Flow for any node whose
   `parentId` points at the dragged node, and the child's own `position` (relative to the
   parent) never changes.

3. **Resize persists as a dimensions change with `setAttributes: true`.**
   Steps: select the `while` frame, drag a corner resize handle to grow it, release, then open
   the overflow menu and choose Export JSON.
   Expected: `FlowCanvas`'s `onNodesChange` receives a `NodeChange` shaped
   `{ type: "dimensions", dimensions: { width, height }, setAttributes: true }`, confirm with
   a temporary `console.log` in that handler, removed before committing Task 8. The exported
   JSON's node for this frame carries the new `width`/`height`, not the catalog default,
   because `setAttributes: true` is React Flow's own signal to also update the plain
   `node.width`/`node.height` fields, not just internal measured state. `applyNodeChanges`
   (Task 5c) applies this change unmodified, and `fromBuilderNode` (Task 3) keeps `width`/
   `height` on the exported `FlowNode` whenever they are defined.

4. **Parents must precede children in the nodes array.**
   Steps: add a node, drop it into a `while` frame, then inspect array order, e.g.
   `console.log(useFlowStoreApi().getState().nodes.map(n => n.id))` from React DevTools.
   Expected: the frame's id appears before every one of its children's ids at all times. If it
   does not, React Flow throws `Parent node [id] not found` or renders the child at the wrong
   absolute position on the next re-render. This is why every store action that can change
   `nodes` (`addNode`, `applyNodeChanges`, `setParent`, `duplicateSelection`, `importFlow`,
   all in Task 5c) runs its result through `sortParentsFirst` (Task 3) before storing it.

5. **Nested groups render with correct z-order.**
   Steps: build a `while` inside a `while` (drop a `while` node from the palette onto an
   existing frame, or call `setParent` twice), then drag the inner frame so it visually
   overlaps the outer frame's header.
   Expected: the inner frame and its children always render above the outer frame's own body
   fill, never hidden behind it. React Flow derives each node's stacking z-index from its
   depth in the parent chain automatically; nothing in the catalog or in `toBuilderNode` sets
   an explicit `zIndex` for `while` nodes to make this hold.

If any observation does not match, stop and revisit this task rather than working around it in
Task 8, a mismatch means an assumption in `lib/group-utils.ts` or `lib/flow-convert.ts` about
how React Flow 12 handles group nodes is wrong and needs fixing at the source.

- [ ] **Step 11: Commit**

```bash
git add lib/group-utils.ts lib/group-utils.test.ts \
  components/builder/use-connection-validator.ts \
  components/builder/nodes/while-node.tsx components/builder/nodes/while-node.test.tsx
git commit -m "$(cat <<'COMMIT_EOF'
feat(builder): add group utils and while-node spike

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
COMMIT_EOF
)"
```
