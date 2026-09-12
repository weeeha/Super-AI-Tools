### Task 12: Seed flows

Three committed flows under `flows/`, one per source skill named in the spec's Seed flows table. Node `data` uses the field names Task 3's `KindData` defines; port names are Task 4's `NODE_CATALOG` ports for each kind. Every JSON object's keys are alphabetical and every file is 2-space indented with a trailing newline, matching Task 3's `canonicalFlowJson`, because Task 12d adds a test that checks each file against it byte for byte. Node ids are `${kind}-${n}`, counting from 1 per kind in the order nodes appear in the file; edge ids follow `edgeIdFor` (`${source}${port ? "@" + port : ""}->${target}`); handles are `${nodeId}:control:in` and `${nodeId}:control:out` or `:out@${port}`. Edges never carry a stored `label` - the skeleton's `BuilderEdge` derives the port label at render time from the source node's data, so a committed edge only ever has `id`, `source`, `sourceHandle`, `target`, `targetHandle`. Global Constraints and the skeleton's Shared interfaces apply as written and are not repeated here.

#### Task 12a: `ship` flow

**Files:**
- Create: `flows/ship.json`

**Interfaces:**
- Consumes: `NodeKind`, `KindData`, `Flow`, `parseFlow` from `lib/flow-schema.ts` (Task 3) · the `control` handle codec from `lib/flow-handles.ts` (Task 3) · the `start`, `agent`, `if-else`, `mcp`, `end` port names and data fields from Task 4's `NODE_CATALOG`.
- Produces: `flows/ship.json`, one of the three files Task 12d's coverage test reads.

Source skill: `~/.claude/me-skills/skills/ship/SKILL.md`.

- [ ] **Step 1: Write `flows/ship.json`**

Ten nodes: `start` (manual trigger), an `agent` for the sanity check (skill step 1), an `if-else` on the working tree being clean versus dirty, a second `if-else` on the current branch being main (the never-push-to-main guardrail lives in this node's `description`, skill step 2), an `agent` for the project-brain refresh (step 3), an `agent` for the commit (step 4), three `mcp` nodes for the `git` push, the `gh` PR, and the `vercel` preview deploy (steps 5 to 7), and `end` returning the deploy link (step 8). The clean branch and the on-main branch both short-circuit straight to `end`.

```json
{
  "createdAt": "2026-09-12T00:00:00.000Z",
  "description": "Commit, push, open or update the PR, deploy a preview, and return the link.",
  "edges": [
    {
      "id": "start-1->agent-1",
      "source": "start-1",
      "sourceHandle": "start-1:control:out",
      "target": "agent-1",
      "targetHandle": "agent-1:control:in"
    },
    {
      "id": "agent-1->if-else-1",
      "source": "agent-1",
      "sourceHandle": "agent-1:control:out",
      "target": "if-else-1",
      "targetHandle": "if-else-1:control:in"
    },
    {
      "id": "if-else-1@c0->end-1",
      "source": "if-else-1",
      "sourceHandle": "if-else-1:control:out@c0",
      "target": "end-1",
      "targetHandle": "end-1:control:in"
    },
    {
      "id": "if-else-1@else->if-else-2",
      "source": "if-else-1",
      "sourceHandle": "if-else-1:control:out@else",
      "target": "if-else-2",
      "targetHandle": "if-else-2:control:in"
    },
    {
      "id": "if-else-2@c0->end-1",
      "source": "if-else-2",
      "sourceHandle": "if-else-2:control:out@c0",
      "target": "end-1",
      "targetHandle": "end-1:control:in"
    },
    {
      "id": "if-else-2@else->agent-2",
      "source": "if-else-2",
      "sourceHandle": "if-else-2:control:out@else",
      "target": "agent-2",
      "targetHandle": "agent-2:control:in"
    },
    {
      "id": "agent-2->agent-3",
      "source": "agent-2",
      "sourceHandle": "agent-2:control:out",
      "target": "agent-3",
      "targetHandle": "agent-3:control:in"
    },
    {
      "id": "agent-3->mcp-1",
      "source": "agent-3",
      "sourceHandle": "agent-3:control:out",
      "target": "mcp-1",
      "targetHandle": "mcp-1:control:in"
    },
    {
      "id": "mcp-1->mcp-2",
      "source": "mcp-1",
      "sourceHandle": "mcp-1:control:out",
      "target": "mcp-2",
      "targetHandle": "mcp-2:control:in"
    },
    {
      "id": "mcp-2->mcp-3",
      "source": "mcp-2",
      "sourceHandle": "mcp-2:control:out",
      "target": "mcp-3",
      "targetHandle": "mcp-3:control:in"
    },
    {
      "id": "mcp-3->end-1",
      "source": "mcp-3",
      "sourceHandle": "mcp-3:control:out",
      "target": "end-1",
      "targetHandle": "end-1:control:in"
    }
  ],
  "id": "ship",
  "name": "Ship",
  "nodes": [
    {
      "data": {
        "label": "Start",
        "trigger": "manual"
      },
      "id": "start-1",
      "kind": "start",
      "position": {
        "x": 0,
        "y": 0
      }
    },
    {
      "data": {
        "instructions": "Run git status and git diff --stat. If the working tree is clean and nothing is deployable, say so; don't fabricate a PR or a deploy.",
        "label": "Sanity check"
      },
      "id": "agent-1",
      "kind": "agent",
      "position": {
        "x": 320,
        "y": 0
      }
    },
    {
      "data": {
        "conditions": [
          {
            "expression": "git status and git diff --stat show no changes",
            "label": "Clean"
          }
        ],
        "label": "Working tree state"
      },
      "id": "if-else-1",
      "kind": "if-else",
      "position": {
        "x": 640,
        "y": 0
      }
    },
    {
      "data": {
        "conditions": [
          {
            "expression": "current branch is main",
            "label": "On main"
          }
        ],
        "description": "Never push straight to main; branch first (repo rule).",
        "label": "Branch check"
      },
      "id": "if-else-2",
      "kind": "if-else",
      "position": {
        "x": 960,
        "y": 0
      }
    },
    {
      "data": {
        "instructions": "If content/prj_*.md exists, before committing: tick finished checklist items, log real decisions, rewrite Next - the one thing, and bump updated_at and stage/health. Stage these files in the same commit.",
        "label": "Refresh project brain"
      },
      "id": "agent-2",
      "kind": "agent",
      "position": {
        "x": 1280,
        "y": 0
      }
    },
    {
      "data": {
        "instructions": "Stage and commit with a concrete message describing what actually changed. End the message with: Co-Authored-By: Claude <noreply@anthropic.com>.",
        "label": "Commit"
      },
      "id": "agent-3",
      "kind": "agent",
      "position": {
        "x": 1600,
        "y": 0
      }
    },
    {
      "data": {
        "label": "Push branch",
        "server": "git",
        "tools": [
          "push"
        ]
      },
      "id": "mcp-1",
      "kind": "mcp",
      "position": {
        "x": 1920,
        "y": 0
      }
    },
    {
      "data": {
        "description": "If a PR for this branch already exists, push to it; don't open a duplicate. Otherwise gh pr create with a real title and body.",
        "label": "Open or update PR",
        "server": "gh",
        "tools": [
          "pr create",
          "pr view"
        ]
      },
      "id": "mcp-2",
      "kind": "mcp",
      "position": {
        "x": 2240,
        "y": 0
      }
    },
    {
      "data": {
        "description": "Default to preview, not prod, unless told to ship prod.",
        "label": "Deploy preview",
        "server": "vercel",
        "tools": [
          "deploy"
        ]
      },
      "id": "mcp-3",
      "kind": "mcp",
      "position": {
        "x": 2560,
        "y": 0
      }
    },
    {
      "data": {
        "label": "End",
        "outputLabel": "Deploy link"
      },
      "id": "end-1",
      "kind": "end",
      "position": {
        "x": 2880,
        "y": 0
      }
    }
  ],
  "schemaVersion": 1,
  "source": "manual",
  "sourceRef": "~/.claude/me-skills/skills/ship/SKILL.md",
  "updatedAt": "2026-09-12T00:00:00.000Z"
}
```

- [ ] **Step 2: Verify the file is valid and complete**

Run:

```bash
node -e "const f=JSON.parse(require('node:fs').readFileSync('flows/ship.json','utf8')); console.log(f.nodes.length, f.edges.length, [...new Set(f.nodes.map(n=>n.kind))].sort().join(','))"
```

Expected: `10 11 agent,end,if-else,mcp,start`

- [ ] **Step 3: Commit**

```bash
git add flows/ship.json
git commit -m "$(cat <<'EOF'
feat(flows): add the ship seed flow

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
)"
```

---

#### Task 12b: `new-project` flow

**Files:**
- Create: `flows/new-project.json`

**Interfaces:**
- Consumes: `NodeKind`, `KindData`, `Flow`, `parseFlow` from `lib/flow-schema.ts` (Task 3) · the `classify`, `file-search`, `agent`, `set-state`, `user-approval`, `note`, `start`, `end` port names and data fields from Task 4's `NODE_CATALOG`.
- Produces: `flows/new-project.json`, one of the three files Task 12d's coverage test reads.

Source skill: `~/.claude/me-skills/skills/new-project/SKILL.md`.

- [ ] **Step 1: Write `flows/new-project.json`**

Thirteen nodes: `start`, a `classify` on the skill's three modes (personal exploration, client engagement, demo to engagement promotion), an `agent` for the light path (personal exploration exits straight to `end`, skill step 2), an `agent` for the promotion checklist that both the client-engagement and demo-to-engagement categories share (step 3), an `agent` for the identity map (step 4) feeding a `set-state` that records its three fields, a `note` carrying the guardrails list, two `file-search` nodes for the two canonical sources (the design-practice-kit template, then the local PLAYBOOK), an `agent` for seeding the HQ (step 5), an `agent` for declaring the pipeline (step 6), a `user-approval` for the kickoff scope commitments (step 7), and `end`. Both branches of the `user-approval` lead to `end`.

```json
{
  "createdAt": "2026-09-12T00:00:00.000Z",
  "description": "Classify a new project, seed its identity map and HQ, and get kickoff scope approved.",
  "edges": [
    {
      "id": "start-1->classify-1",
      "source": "start-1",
      "sourceHandle": "start-1:control:out",
      "target": "classify-1",
      "targetHandle": "classify-1:control:in"
    },
    {
      "id": "classify-1@cat-0->agent-2",
      "source": "classify-1",
      "sourceHandle": "classify-1:control:out@cat-0",
      "target": "agent-2",
      "targetHandle": "agent-2:control:in"
    },
    {
      "id": "classify-1@cat-1->agent-1",
      "source": "classify-1",
      "sourceHandle": "classify-1:control:out@cat-1",
      "target": "agent-1",
      "targetHandle": "agent-1:control:in"
    },
    {
      "id": "classify-1@cat-2->agent-1",
      "source": "classify-1",
      "sourceHandle": "classify-1:control:out@cat-2",
      "target": "agent-1",
      "targetHandle": "agent-1:control:in"
    },
    {
      "id": "agent-2->end-1",
      "source": "agent-2",
      "sourceHandle": "agent-2:control:out",
      "target": "end-1",
      "targetHandle": "end-1:control:in"
    },
    {
      "id": "agent-1->agent-3",
      "source": "agent-1",
      "sourceHandle": "agent-1:control:out",
      "target": "agent-3",
      "targetHandle": "agent-3:control:in"
    },
    {
      "id": "agent-3->set-state-1",
      "source": "agent-3",
      "sourceHandle": "agent-3:control:out",
      "target": "set-state-1",
      "targetHandle": "set-state-1:control:in"
    },
    {
      "id": "set-state-1->file-search-1",
      "source": "set-state-1",
      "sourceHandle": "set-state-1:control:out",
      "target": "file-search-1",
      "targetHandle": "file-search-1:control:in"
    },
    {
      "id": "file-search-1->file-search-2",
      "source": "file-search-1",
      "sourceHandle": "file-search-1:control:out",
      "target": "file-search-2",
      "targetHandle": "file-search-2:control:in"
    },
    {
      "id": "file-search-2->agent-4",
      "source": "file-search-2",
      "sourceHandle": "file-search-2:control:out",
      "target": "agent-4",
      "targetHandle": "agent-4:control:in"
    },
    {
      "id": "agent-4->agent-5",
      "source": "agent-4",
      "sourceHandle": "agent-4:control:out",
      "target": "agent-5",
      "targetHandle": "agent-5:control:in"
    },
    {
      "id": "agent-5->user-approval-1",
      "source": "agent-5",
      "sourceHandle": "agent-5:control:out",
      "target": "user-approval-1",
      "targetHandle": "user-approval-1:control:in"
    },
    {
      "id": "user-approval-1@approve->end-1",
      "source": "user-approval-1",
      "sourceHandle": "user-approval-1:control:out@approve",
      "target": "end-1",
      "targetHandle": "end-1:control:in"
    },
    {
      "id": "user-approval-1@reject->end-1",
      "source": "user-approval-1",
      "sourceHandle": "user-approval-1:control:out@reject",
      "target": "end-1",
      "targetHandle": "end-1:control:in"
    }
  ],
  "id": "new-project",
  "name": "New Project",
  "nodes": [
    {
      "data": {
        "label": "Start",
        "trigger": "manual"
      },
      "id": "start-1",
      "kind": "start",
      "position": {
        "x": 0,
        "y": 0
      }
    },
    {
      "data": {
        "categories": [
          "Personal exploration",
          "Client engagement",
          "Demo to engagement promotion"
        ],
        "label": "Classify project"
      },
      "id": "classify-1",
      "kind": "classify",
      "position": {
        "x": 320,
        "y": 0
      }
    },
    {
      "data": {
        "instructions": "Work through 01-promotion-checklist.md: name settled (renames go to the decision log), client org chosen, demo repo archived with a tombstone pointer, background-IP register drafted before client work starts.",
        "label": "Promotion checklist"
      },
      "id": "agent-1",
      "kind": "agent",
      "position": {
        "x": 640,
        "y": 0
      }
    },
    {
      "data": {
        "instructions": "Name it once; one repo; seed a minimal HQ README with a status header: what it is, status: exploration, live URL when it exists. Add it to whatever index tracks his projects. Skip the rest of the ritual.",
        "label": "Seed light HQ"
      },
      "id": "agent-2",
      "kind": "agent",
      "position": {
        "x": 640,
        "y": 160
      }
    },
    {
      "data": {
        "instructions": "Fill 02-identity-map.md: GitHub org, author email, and Vercel team per repo. Preflight every deploy: git config user.email and .vercel/project.json team must match the map.",
        "label": "Identity map"
      },
      "id": "agent-3",
      "kind": "agent",
      "position": {
        "x": 960,
        "y": 0
      }
    },
    {
      "data": {
        "assignments": [
          {
            "key": "githubOrg",
            "value": "set in 02-identity-map.md"
          },
          {
            "key": "authorEmail",
            "value": "set in 02-identity-map.md"
          },
          {
            "key": "vercelTeam",
            "value": "set in 02-identity-map.md"
          }
        ],
        "label": "Identity map values"
      },
      "id": "set-state-1",
      "kind": "set-state",
      "position": {
        "x": 1280,
        "y": 0
      }
    },
    {
      "data": {
        "color": "yellow",
        "label": "Guardrails",
        "text": "Never push to main; branch per task, PR per change. Everything published starts unlisted or preview; label every artifact exploration or deliverable. One canonical repo set per engagement, no clone-per-mood or folder-copy archives. Specs before builds: nothing gets built without 05-spec-template.md's acceptance criteria and states checklist."
      },
      "height": 176,
      "id": "note-1",
      "kind": "note",
      "position": {
        "x": 1280,
        "y": 320
      },
      "width": 256
    },
    {
      "data": {
        "label": "Design practice kit template",
        "store": "github.com/weeeha/design-practice-kit (template repo)"
      },
      "id": "file-search-1",
      "kind": "file-search",
      "position": {
        "x": 1600,
        "y": 0
      }
    },
    {
      "data": {
        "label": "Local PLAYBOOK",
        "store": "~/ClaudeCode Projects/DesignTeam/practice/PLAYBOOK.md"
      },
      "id": "file-search-2",
      "kind": "file-search",
      "position": {
        "x": 1920,
        "y": 0
      }
    },
    {
      "data": {
        "instructions": "gh repo create <org>/<name>-hq --template weeeha/design-practice-kit --private. Fill 06-project-hq-readme.md with the repo map and live index, and 07-decision-log.md. Delete unused templates from the instance.",
        "label": "Seed HQ"
      },
      "id": "agent-4",
      "kind": "agent",
      "position": {
        "x": 2240,
        "y": 0
      }
    },
    {
      "data": {
        "instructions": "Write 08-design-pipeline.md: source of truth (default code-first, Figma mirrors one-way), rhythm, and gates. Date it.",
        "label": "Declare pipeline"
      },
      "id": "agent-5",
      "kind": "agent",
      "position": {
        "x": 2560,
        "y": 0
      }
    },
    {
      "data": {
        "label": "Kickoff scope commitments",
        "prompt": "Walk into the kickoff meeting with a POV and leave with the scope commitments written. Approve before proceeding."
      },
      "id": "user-approval-1",
      "kind": "user-approval",
      "position": {
        "x": 2880,
        "y": 0
      }
    },
    {
      "data": {
        "label": "End",
        "outputLabel": "Kickoff ready"
      },
      "id": "end-1",
      "kind": "end",
      "position": {
        "x": 3200,
        "y": 0
      }
    }
  ],
  "schemaVersion": 1,
  "source": "manual",
  "sourceRef": "~/.claude/me-skills/skills/new-project/SKILL.md",
  "updatedAt": "2026-09-12T00:00:00.000Z"
}
```

- [ ] **Step 2: Verify the file is valid and complete**

Run:

```bash
node -e "const f=JSON.parse(require('node:fs').readFileSync('flows/new-project.json','utf8')); console.log(f.nodes.length, f.edges.length, [...new Set(f.nodes.map(n=>n.kind))].sort().join(','))"
```

Expected: `13 14 agent,classify,end,file-search,note,set-state,start,user-approval`

- [ ] **Step 3: Commit**

```bash
git add flows/new-project.json
git commit -m "$(cat <<'EOF'
feat(flows): add the new-project seed flow

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
)"
```

---

#### Task 12c: `subagent-build-loop` flow

**Files:**
- Create: `flows/subagent-build-loop.json`

**Interfaces:**
- Consumes: `NodeKind`, `KindData`, `Flow`, `parseFlow` from `lib/flow-schema.ts` (Task 3) · the `while`, `agent`, `if-else`, `transform`, `guardrails`, `user-approval`, `start`, `end` port names and data fields from Task 4's `NODE_CATALOG`, including the rule that a `while` node's children carry `parentId` and positions relative to the frame, and that an edge may only connect two nodes sharing the same `parentId`.
- Produces: `flows/subagent-build-loop.json`, the last of the three files Task 12d's coverage test reads.

Source skill: `~/.claude/plugins/cache/claude-plugins-official/superpowers/6.3.0/skills/subagent-driven-development/SKILL.md`.

- [ ] **Step 1: Write `flows/subagent-build-loop.json`**

Ten nodes: `start`, then a `while` frame ("more tasks remain in the plan") sized to hold its five children in a row - an `agent` for the implementer (with the no-subagents contract in its instructions), an `agent` for the spec reviewer, an `agent` for the quality reviewer, an `if-else` on whether the review passed, and a `transform` that carries the ledger context to the next implementer's dispatch, fed by both the approved and the findings-open branch. After the frame, top level again: a `guardrails` node for the four stop conditions from the skill's Setup section, a `user-approval` before finishing the branch, and `end`. The frame's own `width`/`height` and its children's positions are explicit per the spec's `while` rule; the children's `x` steps by 320 starting at 32, `y` is 64 for all five.

```json
{
  "createdAt": "2026-09-12T00:00:00.000Z",
  "description": "Loop an implementer and reviewers over the remaining tasks until the branch is ready to finish.",
  "edges": [
    {
      "id": "start-1->while-1",
      "source": "start-1",
      "sourceHandle": "start-1:control:out",
      "target": "while-1",
      "targetHandle": "while-1:control:in"
    },
    {
      "id": "while-1->guardrails-1",
      "source": "while-1",
      "sourceHandle": "while-1:control:out",
      "target": "guardrails-1",
      "targetHandle": "guardrails-1:control:in"
    },
    {
      "id": "guardrails-1@pass->user-approval-1",
      "source": "guardrails-1",
      "sourceHandle": "guardrails-1:control:out@pass",
      "target": "user-approval-1",
      "targetHandle": "user-approval-1:control:in"
    },
    {
      "id": "guardrails-1@fail->end-1",
      "source": "guardrails-1",
      "sourceHandle": "guardrails-1:control:out@fail",
      "target": "end-1",
      "targetHandle": "end-1:control:in"
    },
    {
      "id": "user-approval-1@approve->end-1",
      "source": "user-approval-1",
      "sourceHandle": "user-approval-1:control:out@approve",
      "target": "end-1",
      "targetHandle": "end-1:control:in"
    },
    {
      "id": "user-approval-1@reject->end-1",
      "source": "user-approval-1",
      "sourceHandle": "user-approval-1:control:out@reject",
      "target": "end-1",
      "targetHandle": "end-1:control:in"
    },
    {
      "id": "agent-1->agent-2",
      "source": "agent-1",
      "sourceHandle": "agent-1:control:out",
      "target": "agent-2",
      "targetHandle": "agent-2:control:in"
    },
    {
      "id": "agent-2->agent-3",
      "source": "agent-2",
      "sourceHandle": "agent-2:control:out",
      "target": "agent-3",
      "targetHandle": "agent-3:control:in"
    },
    {
      "id": "agent-3->if-else-1",
      "source": "agent-3",
      "sourceHandle": "agent-3:control:out",
      "target": "if-else-1",
      "targetHandle": "if-else-1:control:in"
    },
    {
      "id": "if-else-1@c0->transform-1",
      "source": "if-else-1",
      "sourceHandle": "if-else-1:control:out@c0",
      "target": "transform-1",
      "targetHandle": "transform-1:control:in"
    },
    {
      "id": "if-else-1@else->transform-1",
      "source": "if-else-1",
      "sourceHandle": "if-else-1:control:out@else",
      "target": "transform-1",
      "targetHandle": "transform-1:control:in"
    }
  ],
  "id": "subagent-build-loop",
  "name": "Subagent Build Loop",
  "nodes": [
    {
      "data": {
        "label": "Start",
        "trigger": "manual"
      },
      "id": "start-1",
      "kind": "start",
      "position": {
        "x": 0,
        "y": 0
      }
    },
    {
      "data": {
        "condition": "more tasks remain in the plan",
        "label": "Tasks remain"
      },
      "height": 240,
      "id": "while-1",
      "kind": "while",
      "position": {
        "x": 320,
        "y": 0
      },
      "width": 1680
    },
    {
      "data": {
        "instructions": "Implement the task: write the code, run the tests, self-review, then commit. Never dispatch subagents of your own; no helpers and no reviewers.",
        "label": "Implementer"
      },
      "id": "agent-1",
      "kind": "agent",
      "parentId": "while-1",
      "position": {
        "x": 32,
        "y": 64
      }
    },
    {
      "data": {
        "instructions": "Read the task brief and the diff. Verdict: spec compliant or not, against every requirement in the brief.",
        "label": "Spec reviewer"
      },
      "id": "agent-2",
      "kind": "agent",
      "parentId": "while-1",
      "position": {
        "x": 352,
        "y": 64
      }
    },
    {
      "data": {
        "instructions": "Read the same diff for code quality: clarity, tests, and consistency with the codebase. Verdict: approved, or findings rated Critical, Important, or Minor.",
        "label": "Quality reviewer"
      },
      "id": "agent-3",
      "kind": "agent",
      "parentId": "while-1",
      "position": {
        "x": 672,
        "y": 64
      }
    },
    {
      "data": {
        "conditions": [
          {
            "expression": "spec compliance and code quality are both approved",
            "label": "Approved"
          }
        ],
        "label": "Review passed?"
      },
      "id": "if-else-1",
      "kind": "if-else",
      "parentId": "while-1",
      "position": {
        "x": 992,
        "y": 64
      }
    },
    {
      "data": {
        "expression": "Carry forward interfaces, decisions, and any parked findings from this task's ledger entry into the next task's dispatch.",
        "label": "Context for next implementer"
      },
      "id": "transform-1",
      "kind": "transform",
      "parentId": "while-1",
      "position": {
        "x": 1312,
        "y": 64
      }
    },
    {
      "data": {
        "checks": [
          "Irreversible or destructive operation",
          "Security-sensitive action",
          "Side effect outside the worktree that needs a yes first",
          "Plan so broken every path forward is a guess"
        ],
        "label": "Stop conditions"
      },
      "id": "guardrails-1",
      "kind": "guardrails",
      "position": {
        "x": 2048,
        "y": 0
      }
    },
    {
      "data": {
        "label": "Finish branch",
        "prompt": "Confirm how to integrate the branch before finishing. A merge or a push to a shared branch needs a yes first."
      },
      "id": "user-approval-1",
      "kind": "user-approval",
      "position": {
        "x": 2368,
        "y": 0
      }
    },
    {
      "data": {
        "label": "End",
        "outputLabel": "Branch finished"
      },
      "id": "end-1",
      "kind": "end",
      "position": {
        "x": 2688,
        "y": 0
      }
    }
  ],
  "schemaVersion": 1,
  "source": "manual",
  "sourceRef": "~/.claude/plugins/cache/claude-plugins-official/superpowers/6.3.0/skills/subagent-driven-development/SKILL.md",
  "updatedAt": "2026-09-12T00:00:00.000Z"
}
```

- [ ] **Step 2: Verify the file is valid and complete**

Run:

```bash
node -e "const f=JSON.parse(require('node:fs').readFileSync('flows/subagent-build-loop.json','utf8')); console.log(f.nodes.length, f.edges.length, [...new Set(f.nodes.map(n=>n.kind))].sort().join(','))"
```

Expected: `10 11 agent,end,guardrails,if-else,start,transform,user-approval,while`

- [ ] **Step 3: Commit**

```bash
git add flows/subagent-build-loop.json
git commit -m "$(cat <<'EOF'
feat(flows): add the subagent-build-loop seed flow

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
)"
```

---

#### Task 12d: Coverage test and `flows:format` script

**Files:**
- Create: `flows/flows.test.ts`, `scripts/format-flows.mts`
- Modify: `package.json`

**Interfaces:**
- Consumes: `NODE_KINDS`, `NodeKind`, `parseFlow` from `lib/flow-schema.ts` (Task 3) · `canonicalFlowJson` from `lib/flow-convert.ts` (Task 3) · the three files Tasks 12a-12c committed.
- Produces: a regression test guarding the three seed flows, and a `pnpm flows:format` command later tasks and any future hand edit to a flow file run before committing.

Tasks 12a-12c already wrote schema-valid, canonical JSON, so this test is a regression guard rather than a red-green cycle: it is expected to pass the moment it is written, and its job is to catch the day a hand edit to one of the three files drifts from the schema, drops a kind, or breaks canonical formatting.

- [ ] **Step 1: Write `flows/flows.test.ts`**

```ts
// flows/flows.test.ts
// @vitest-environment node
import { readFileSync } from "node:fs";
import path from "node:path";
import { describe, expect, it } from "vitest";
import { canonicalFlowJson } from "@/lib/flow-convert";
import { NODE_KINDS, parseFlow, type NodeKind } from "@/lib/flow-schema";

const SEED_SLUGS = ["ship", "new-project", "subagent-build-loop"] as const;

function readSeed(slug: string): string {
  return readFileSync(path.join(process.cwd(), "flows", `${slug}.json`), "utf8");
}

describe("seed flows", () => {
  it("every seed flow parses", () => {
    for (const slug of SEED_SLUGS) {
      const result = parseFlow(JSON.parse(readSeed(slug)));
      if (!result.ok) throw new Error(`${slug}.json: ${result.errors.join("; ")}`);
      expect(result.ok).toBe(true);
    }
  });

  it("the union of kinds across the seed flows equals NODE_KINDS", () => {
    const kinds = new Set<NodeKind>();
    for (const slug of SEED_SLUGS) {
      const result = parseFlow(JSON.parse(readSeed(slug)));
      if (!result.ok) throw new Error(`${slug}.json failed to parse`);
      for (const node of result.flow.nodes) kinds.add(node.kind);
    }
    expect(kinds).toEqual(new Set(NODE_KINDS));
  });

  it.each(SEED_SLUGS)("%s.json is already canonical", (slug) => {
    const raw = readSeed(slug);
    const result = parseFlow(JSON.parse(raw));
    if (!result.ok) throw new Error(`${slug}.json failed to parse`);
    expect(raw).toBe(canonicalFlowJson(result.flow));
  });
});
```

- [ ] **Step 2: Run it**

Run: `pnpm test -- flows/flows.test.ts`
Expected: `Test Files  1 passed (1)`, `Tests  5 passed (5)` (1 parse check, 1 coverage check, 3 from `it.each` over the three slugs) - PASS on the first run, confirming Tasks 12a-12c's output rather than driving it.

- [ ] **Step 3: Write `scripts/format-flows.mts`**

```ts
// scripts/format-flows.mts
// Rewrites every flows/*.json file through canonicalFlowJson, so committed
// seeds and any later hand edits always match what flows/flows.test.ts checks.
// Run through tsx (a Task 1 dev dependency), NOT bare node. Node 24 strips
// types but resolves modules like plain ESM, and lib/flow-schema.ts imports
// parseHandleId from "@/components/flow-kit/flow-types" at runtime. That "@/"
// alias is a tsconfig paths mapping, which only tsx, vitest and the Next
// bundler resolve; bare node fails with ERR_MODULE_NOT_FOUND.
import { readdir, readFile, writeFile } from "node:fs/promises";
import path from "node:path";
import { canonicalFlowJson } from "../lib/flow-convert.ts";
import { parseFlow } from "../lib/flow-schema.ts";

const FLOWS_DIR = path.join(process.cwd(), "flows");

async function main() {
  const entries = await readdir(FLOWS_DIR);
  const files = entries.filter((f) => f.endsWith(".json")).sort();
  let changed = 0;
  for (const file of files) {
    const full = path.join(FLOWS_DIR, file);
    const raw = await readFile(full, "utf8");
    const result = parseFlow(JSON.parse(raw));
    if (!result.ok) {
      throw new Error(`${file}: ${result.errors.join("; ")}`);
    }
    const canonical = canonicalFlowJson(result.flow);
    if (canonical !== raw) {
      await writeFile(full, canonical);
      changed++;
      console.log(`formatted ${file}`);
    }
  }
  console.log(`flows:format: ${files.length} file(s) checked, ${changed} rewritten.`);
}

main().catch((err) => {
  console.error(err);
  process.exitCode = 1;
});
```

- [ ] **Step 4: Add the `flows:format` package script**

Modify `package.json`'s `"scripts"` block (alongside Task 1's `test`, `lint`, `typecheck`, `check:tokens`, `build`, `e2e`, `start`), adding:

```json
"flows:format": "tsx scripts/format-flows.mts"
```

- [ ] **Step 5: Confirm the seed files are already canonical**

Run:

```bash
pnpm flows:format
git diff --stat flows/
```

Expected: the script logs `flows:format: 3 file(s) checked, 0 rewritten.` and `git diff --stat` prints nothing, since Tasks 12a-12c already wrote canonical JSON.

- [ ] **Step 6: Run the full verification suite**

```bash
pnpm test
pnpm typecheck
pnpm lint
pnpm build
```

Expected:
- `pnpm test` - every test file from Tasks 1 through 11 plus `flows/flows.test.ts` passes.
- `pnpm typecheck` - exits 0.
- `pnpm lint` - exits 0.
- `pnpm build` - exits 0, and for the first time actually statically renders `/` and `/flow/ship`, `/flow/new-project`, `/flow/subagent-build-loop` through `generateStaticParams`, now that `flows/` is populated.

- [ ] **Step 7: Commit**

```bash
git add flows/flows.test.ts scripts/format-flows.mts package.json
git commit -m "$(cat <<'EOF'
test(flows): add seed flow coverage test and flows:format script

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
EOF
)"
```

---
