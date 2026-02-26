# Workflow Specification

> Fill in the sections below to describe what the CI/CD workflow should do.
> Then ask Copilot: "Implement a GitHub Actions workflow based on workflow-spec.md"

## Trigger

- **Schedule:** Every 2 hours (`cron: '0 */2 * * *'`)
- **Manual:** `workflow_dispatch`
  - Input: `detailed` (boolean, default `false`) — when `true`, graph nodes display image:tag metadata (see Section 7)

## Requirements

### 1. Spin up a Kind cluster

Create an ephemeral Kubernetes cluster using [Kind](https://kind.sigs.k8s.io/).

### 2. Install Radius

Install Radius into the Kind cluster using custom images:

```bash
rad install kubernetes \
    --set rp.image=ghcr.io/nithyatsu/applications-rp,rp.tag=latest \
    --set dynamicrp.image=ghcr.io/nithyatsu/dynamic-rp,dynamicrp.tag=latest \
    --set controller.image=ghcr.io/nithyatsu/controller,controller.tag=latest \
    --set ucp.image=ghcr.io/nithyatsu/ucpd,ucp.tag=latest \
    --set bicep.image=ghcr.io/nithyatsu/bicep,bicep.tag=latest
```

### 3. Verify Radius is ready

Run `rad group create test` and confirm it succeeds. This validates that all Radius pods are healthy and the control plane is operational.

### 4. Generate the application graph

Run `rad app graph <fully-qualified-path-to-app.bicep>`.

- The command requires an **absolute file path** (e.g., `${{ github.workspace }}/app.bicep`).
- The command outputs a structured representation of the application's resources and their connections.
- The command gets updated all the time. Try it out, update the workflow to work with the latest behavior. 

### 5. Build a visual graph from the output

Parse the output from step 4 and construct a renderable graph. Extract:

- **Nodes** — each resource (name, type, source file, line number)
- **Edges** — connections between resources

### 6. Render the graph — Mermaid + SVG + Interactive explorer

The architecture visualization uses a **three-tier approach**:

1. **README Mermaid diagram** — A Mermaid `graph LR` diagram embedded directly in `README.md`. GitHub renders Mermaid natively. Mermaid `click ... href` directives create **working clickable nodes** that open the resource definition in `app.bicep` on GitHub. A footer link leads to the interactive explorer.
2. **SVG overview** — A static SVG generated for the GitHub Pages explorer and for direct viewing in the repo (`graph.svg`). SVG nodes have `<title>` tooltips (image:tag) and `<a href>` links to source definitions. Includes a footer link to the interactive explorer.
3. **GitHub Pages interactive explorer** — A fully interactive Cytoscape.js web page with click-to-expand detail panels, zoom, pan, and drill-down.

#### Why Mermaid for README (not SVG)

GitHub renders SVGs referenced in Markdown as `<img>` tags. This preserves the **Markdown image-title tooltip** — e.g. `![Architecture](graph.svg "working")` shows `"working"` on hover over the whole image — but **strips per-node interactivity** inside the SVG (`<title>` tooltips per element, `<a href>` links, and click handlers do not fire). Mermaid `click ... href` directives **do** create working hyperlinks when GitHub renders the diagram, giving us **per-node click-to-source** in the README.

The SVG is still generated for:
- The GitHub Pages explorer (displayed alongside the graph data)
- Direct viewing when navigating to `graph.svg` in the repo (where per-node `<title>` and `<a>` do work)

#### Deployment model

Generated assets are **not committed** to the repo — they are assembled at runtime and deployed as a GitHub Pages artifact. This keeps the repo clean for end users.

```
.github/pages/
└── index.html          ← Static template (checked in, hidden from casual browsing)

# Generated at CI runtime (never committed, .gitignore'd):
docs/
├── graph-data.json     ← Auto-generated JSON for Cytoscape.js explorer
└── graph.svg           ← Auto-generated SVG overview

_site/                  ← Assembled deploy directory (CI only)
├── index.html          ← Copied from .github/pages/
├── graph-data.json     ← Copied from docs/
└── graph.svg           ← Copied from docs/
```

- The CI workflow generates `docs/graph-data.json` and `docs/graph.svg` from the `rad app graph` output.
- `graph.svg` is also committed to the repo root so the README can reference it.
- A `_site/` directory is assembled from `.github/pages/index.html` + generated files, then uploaded via `actions/upload-pages-artifact`.
- **GitHub Pages** is enabled via `actions/configure-pages@v5` with `enablement: true` and deployed via `actions/deploy-pages@v4`.
- `docs/` and `_site/` are in `.gitignore` — users of the repo never see generated artifacts.
- No build step is required — `index.html` is a static file that loads Cytoscape.js from a CDN.

#### Graph library (interactive explorer)

Use **[Cytoscape.js](https://js.cytoscape.org/)** for the GitHub Pages interactive explorer:
- Purpose-built for graph/network visualization
- Accepts JSON data directly (`{ nodes: [...], edges: [...] }`) — maps naturally to `app-graph.json`
- Single HTML file + CDN (~300KB) — no `npm install` or build pipeline
- Built-in layout algorithms (`dagre` for directed graphs), zoom, and pan

#### Visual style

The same visual style applies to both the Mermaid diagram, SVG overview, and the Cytoscape.js explorer:

| Property        | Value                                          |
|-----------------|-------------------------------------------------|
| Background      | White (`#ffffff`)                               |
| Font color      | Dark (`#1f2328`)                               |
| Node shape      | Rounded-corner rectangles (`rx:6, ry:6` in classDef/SVG; `shape: 'roundrectangle'` in Cytoscape) |
| Container border| Green (`#2da44e`)                               |
| Datastore border| Amber (`#d4a72c`)                               |
| Node fill       | White (`#ffffff`)                               |
| Edge color      | Green (`#2da44e`)                               |
| Font            | `-apple-system, BlinkMacSystemFont, "Segoe UI", "Noto Sans", Helvetica, Arial, sans-serif` |

#### README Mermaid diagram — Interactivity

The Mermaid diagram in README uses `click` directives to make every node a hyperlink:

```mermaid
click frontend href "https://github.com/<owner>/<repo>/blob/<branch>/app.bicep#L<N>" "app.bicep:<N>" _blank
```

| Feature              | Behavior                                        |
|----------------------|-------------------------------------------------|
| **Click node**       | Opens the **source file definition on GitHub** at the resource's line number (`app.bicep#L<N>`). This works because GitHub preserves Mermaid `click href` directives as real hyperlinks. |
| **Footer link**      | Below the diagram, an **"Interactive Graph →"** link opens the GitHub Pages explorer for full interactivity. |

#### SVG overview — Interactivity (direct file viewing)

When viewing `graph.svg` directly in the repo (not via README), individual nodes have:
- `<title>` tooltips showing **image:tag** for containers (or resource type for others)
- `<a href>` links to the **source file definition** on GitHub (`app.bicep#L<N>`)
- A footer link labeled **"Interactive Graph →"** that opens the GitHub Pages explorer

#### GitHub Pages Explorer — Interactivity

The Cytoscape.js interactive explorer is styled as a **floating popup card** — a centered modal-like container with a dimmed backdrop, rounded corners, drop shadow, and a subtle entrance animation. This prevents the page from feeling like a redirect to a separate app.

| Feature              | Behavior                                        |
|----------------------|-------------------------------------------------|
| **Popup card UI**    | The explorer is contained in a `960×700px` max card with a title bar ("Architecture Explorer"), Fit button, and a "Repo" link back to the GitHub repository. The page background is a semi-transparent dark overlay. |
| **Tooltip on hover** | Hovering a node shows a rich floating tooltip (`<div>` overlay positioned near the cursor) with: **resource name**, **image:tag** (for containers) or **resource type** (for others), **last commit short hash**, and **commit author**. Data comes from `git blame` run at generation time. Implemented via Cytoscape `mouseover`/`mouseout` events. |
| **Click to expand**  | Clicking a node opens an **inline detail panel** (slide-in sidebar) showing extended resource information. This is the core interactive feature that cannot work in README. |
| **Auto-focus from README** | The page reads the `?node=<name>` query parameter from the URL. If present, the page auto-selects that node, centers the viewport on it, and opens its detail panel immediately. This creates a seamless flow from README click → interactive exploration. |
| **Click to navigate**| The detail panel includes a **"View source"** link that opens the resource definition on GitHub (`https://github.com/<owner>/<repo>/blob/<branch>/app.bicep#L<N>`) with the line highlighted. |
| **Zoom & pan**       | Built-in Cytoscape.js zoom (scroll wheel) and pan (click-drag background). A **"Fit"** button in the title bar resets the viewport to show all nodes. |
| **Layout**           | Use the `dagre` layout (hierarchical left-to-right) for directed acyclic graphs. The layout runs automatically on load. |
| **Responsive**       | On smaller screens (<700px), the card fills the viewport and the detail panel stacks vertically below the graph. |

#### Detail panel contents

When a user clicks a node (or arrives via `?node=` from README), the detail panel displays:

| Field              | Value                                           |
|--------------------|-------------------------------------------------|
| **Resource name**  | e.g., `frontend`                                |
| **Resource type**  | e.g., `Applications.Core/containers`            |
| **Source**         | `app.bicep`, line N (clickable link to GitHub)  |
| **Connections**    | List of outbound connections (e.g., `→ backend`, `→ database`) |
| **Image:tag**      | _(detailed mode only)_ e.g., `ghcr.io/image-registry/magpie:latest` |

#### README update

The `README.md` Architecture section is updated with a Mermaid diagram and a link to the interactive explorer:

````markdown
> *Auto-generated from `app.bicep` — click any node to jump to its definition in the source.*

```mermaid
<generated mermaid block>
```

[Interactive Graph →](https://<owner>.github.io/<repo>/)
````

- **No `<div>` wrapper** — wrapping the Mermaid fence in `<div align="center">` causes GitHub to treat it as inline HTML, which **breaks Mermaid rendering**. The block is placed directly under the heading.
- Clicking any node opens the resource definition in `app.bicep` on GitHub.
- The **"Interactive Graph →"** footer link opens the GitHub Pages Cytoscape.js explorer.
- The diagram should be the only content between the `## Architecture` heading and the next `##` heading.

#### GitHub Pages setup

The workflow must ensure GitHub Pages is enabled. Use the official GitHub Pages deployment actions with `enablement: true` to auto-create the Pages site if it doesn't exist:

```yaml
- name: Assemble Pages site
  run: |
    mkdir -p _site
    cp .github/pages/index.html _site/
    cp docs/graph-data.json     _site/
    cp docs/graph.svg           _site/

- uses: actions/configure-pages@v5
  with:
    enablement: true

- uses: actions/upload-pages-artifact@v3
  with:
    path: _site/

- uses: actions/deploy-pages@v4
```

### 7. Detailed mode — Image dependency graph

The workflow accepts a `detailed` input variable (boolean, default `false` for the main workflow).

When `detailed` is `true`, graph nodes display **extended metadata** beyond the resource name:

#### Node label format (detailed mode)

Each container node renders as a multi-line label:

```
<resource-name>
<image>:<tag>
```

For example:
```
frontend
ghcr.io/image-registry/magpie:latest
```

#### Image and tag resolution

| Priority | Source | Example |
|----------|--------|---------|
| 1 (highest) | Explicit `image` property in `app.bicep` (including via parameter default values) | `image: 'ghcr.io/image-registry/magpie:latest'` → image = `ghcr.io/image-registry/magpie`, tag = `latest` |
| 2 (fallback) | Derive from resource name | resource named `frontend` → image = `frontend`, tag = `latest` |

- If the `image` property contains a colon, split on the **last** colon to separate image and tag.
- If the `image` property has no colon, the tag defaults to `latest`.
- If no `image` property is found (e.g., non-container resources like datastores), show only the resource name (no image/tag line).

#### Visual style (detailed mode)

- Node labels use `<br/>` for line breaks in Mermaid.
- The image:tag line is rendered in a smaller or secondary style (lighter color `#656d76`) to distinguish it from the resource name.
- All other styling (borders, colors, edges) remains the same as the standard graph.

#### Image dependency graph

In detailed mode, the resulting diagram effectively becomes an **image dependency graph** — it shows which container images depend on (connect to) which other container images. This is useful for understanding the supply chain of container images in the application.

### 8. Commit and push

Auto-commit changes to `README.md`, `graph.svg`, and `.radius/app-graph.json` only if the graph has changed. Generated Pages assets (`docs/`, `_site/`) are **not committed** — they are deployed directly as a workflow artifact.

Then assemble `_site/` from `.github/pages/index.html` + generated files and deploy to GitHub Pages.

---

## User Story 4 — PR Graph Diff (P2)

**Goal:** Show a visual diff of the app graph in PR comments so reviewers can see architectural impact without deploying.

**Depends on:** User Stories 1–3 being stable.

### Operational model

The GitHub Action reads committed `.radius/app-graph.json` files from git history — it does **not** generate graphs on-demand. No Bicep/Radius tooling is needed, keeping the Action lightweight and fast.

### Trigger events

| Event | Behavior |
|-------|----------|
| `pull_request` (every push) | Posts or updates a diff comment on the PR |
| `push` to `main` | Updates the baseline for historical comparison |

The comment is posted on **every push** to the PR, not just the first one. If no Bicep/graph changes exist, the comment says "No app graph changes detected."

### Detailed mode in PRs

The PR workflow runs in **detailed mode by default** (`detailed: true`). This means all graph nodes in the PR comment (side-by-side graphs and the consolidated diff graph) display the extended image:tag metadata described in Section 7 of the main workflow spec. This gives reviewers immediate visibility into which container images changed and how the image dependency chain is affected.

### Clickable nodes in the consolidated diff graph

In the PR comment's consolidated diff graph, **every node must be clickable**. Clicking a node opens the PR's **Files changed** view (`/files`) with the browser scrolled to the resource definition corresponding to that node.

#### Link format

```
https://github.com/<owner>/<repo>/pull/<pr_number>/files#diff-<file_hash>R<line_number>
```

Where:
- `<file_hash>` is the SHA-256 hash of the file path (e.g., `app.bicep`) — this matches GitHub's anchor format for PR diff files.
- `R<line_number>` is the right-side line number in the diff corresponding to the resource definition's start line in the PR head.

#### Mermaid `click` directive

```mermaid
click frontend href "https://github.com/<owner>/<repo>/pull/<pr>/files#diff-<hash>R<line>" "frontend — app.bicep line <N>" _blank
```

Each node in the diff graph (added, removed, modified, or unchanged) should have a `click` directive. For removed resources, link to the base side of the diff (`L<line>` instead of `R<line>`).

### Monorepo support

Auto-detect all `**/.radius/app-graph.json` files. Each graph is diffed independently with separate comment sections per application.

### PR comment format

The comment includes:

1. **Side-by-side Mermaid graphs** — `main` graph on the left, PR graph on the right, for visual comparison. Both rendered in **detailed mode** (showing image:tag on each container node).
2. **Diff graph** — a single consolidated Mermaid graph using color-coded nodes:
   - 🟢 Green border — added resources
   - 🟡 Amber border — modified resources
   - 🔴 Red border — removed resources
   - Gray border — unchanged resources
   - All container nodes display image:tag metadata (detailed mode).
3. **Clickable nodes in diff graph** — every node in the consolidated diff graph is clickable. Clicking opens the PR's **Files changed** page (`/files`) scrolled to the resource definition. Added/modified/unchanged resources link to the right-side diff line (`R<line>`); removed resources link to the left-side diff line (`L<line>`).
4. **Resources & connections table** — lists added/removed/modified resources and connections.
5. **Footer** — "Powered by [Radius](https://radapp.io/)"

### Acceptance criteria

1. PR includes changes to `.radius/app-graph.json` → Action posts a comment with side-by-side graphs + diff graph.
2. PR has no Bicep or graph changes → Comment says "No app graph changes detected."
3. PR adds a new connection → Diff graph shows the new edge; new resource node is green.
4. PR removes a resource → Diff graph shows the removed node in red (dashed border).
5. PR modifies a resource → Diff graph shows the modified node in amber.
6. PR comment already exists from a previous push → Existing comment is updated, not duplicated.
7. Clicking any node in the consolidated diff graph opens the PR's Files changed page with the resource definition in focus (right-side line for added/modified/unchanged; left-side line for removed).
8. Bicep files changed but `.radius/app-graph.json` was not updated → CI validation fails with instructions.
9. Monorepo with multiple apps → Unified comment with separate sections per application.
10. Comment footer says "Powered by [Radius](https://radapp.io/)".
11. PR workflow runs in detailed mode by default — all container nodes in side-by-side and diff graphs show image:tag metadata.
12. Main workflow with `detailed: true` input → README diagram shows image:tag on each container node.
13. Main workflow with `detailed: false` (default) → README diagram shows only resource names (original behavior).