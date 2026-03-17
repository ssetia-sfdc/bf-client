# bf-client User Guide

bf-client is a terminal UI for monitoring and managing a [Buildfarm](https://github.com/buildfarm/buildfarm) remote execution cluster. It connects to a Buildfarm instance via gRPC and provides real-time visibility into queue depths, worker status, and individual operations.

## Starting bf-client

```bash
bf-client <reapi-host> [ca-cert-path]
```

- `reapi-host` — Buildfarm gRPC endpoint. Prefix with `grpcs://` for TLS (port defaults to 443 for TLS)
- `ca-cert-path` — Optional path to a CA certificate file for TLS verification

Examples:

```bash
# Insecure connection
bf-client buildfarm.local:8980

# TLS connection with custom CA
bf-client grpcs://buildfarm.example.com /path/to/ca.pem
```

## Global Navigation

bf-client uses vim-style navigation throughout. These keys work in most views:

| Key | Action |
|-----|--------|
| `j` / Down | Move selection down |
| `k` / Up | Move selection up |
| `h` / Left | Move left / collapse / go back |
| `l` / Right | Move right / expand |
| `Enter` | Open / drill into selected item |
| `q` / `Escape` / `Ctrl-C` | Go back to previous view (or quit from main) |

## Cluster Dashboard (Queue View)

This is the main screen you see on startup. It shows an overview of the entire cluster.

### What it shows

- **Connection info** — gRPC host with its last measured latency
- **Workers** — Total number of active execute workers
- **Prequeue** — Number of operations waiting to be validated and queued
- **Queue** — Number of operations waiting to be dispatched to workers (broken down by provision/platform)
- **Dispatched** — Number of operations currently running on workers

The left panel is a tree showing these stats. The right panel shows either a time-series plot of queue depths or a worker list, depending on what is selected.

### Navigating the dashboard

| Key | Action |
|-----|--------|
| `j` / `k` | Move selection between Workers, Prequeue, Queue, Dispatched |
| `l` / Right | Expand a tree node (e.g. see Queue broken down by provision) |
| `h` / Left | Collapse a tree node, or toggle focus between the stats tree and worker list |
| `L` / Shift-Right | Recursively expand all children of the selected node |
| `H` / Shift-Left | Recursively collapse all children of the selected node |
| `Enter` | Open an operation list for the selected queue, or open a worker detail view |
| `Tab` | When Workers is selected, toggle worker list display between Slots and Actions view |
| `>` / `<` | When Workers is selected, cycle the worker sort order (Executions or Name) |
| `J` / PageDown | Page down in the worker list |
| `K` / PageUp | Page up in the worker list |
| `/` | Open the search dialog |
| `s` | Open settings |
| `D` | Open a test document view |

### Queue depth plots

When Prequeue, Queue, or Dispatched is selected, the right panel displays a real-time scatter plot of that metric over the last 60 sample windows. This gives you a quick visual of whether the queue is growing, shrinking, or stable.

- Prequeue plot is red
- Queue plot is yellow
- Dispatched plot is cyan

### Worker list

When Workers is selected, the right panel shows a bar chart for every worker. Each worker row displays three color-coded stage bars:

- **Blue** — InputFetch stage (slots used / configured)
- **Red** — ExecuteAction stage (slots used / configured)
- **Green** — ReportResult stage (slots used / configured)

When a stage is fully occupied, the bar inverts to a filled background. Workers marked "stale" haven't responded to a profile request recently.

Press `Enter` on a worker row to open the Worker Detail view.

## Worker Detail View

Shows a single worker's pipeline stages with all operations currently assigned to it.

### What it shows

- **Title bar** — Worker address, CAS entry count, CAS size and utilization percentage, unreferenced entry percentage
- **Match** — Operations in the match stage
- **InputFetch** — Operations fetching inputs (slots used / configured)
- **Execute** — Operations currently executing (slots used / configured, total count, average slots per execution)
- **ReportResult** — Operations uploading results (slots used / configured)

Each operation in a stage list shows its name (or target/mnemonic/build, depending on the display field) and elapsed time. Operations are color-coded:

- Normal text — actively running in this stage
- Blue background — stalled (waiting between sub-stages)
- Red background — errored

### Navigation and controls

| Key | Action |
|-----|--------|
| `Tab` | Cycle focus between Match, InputFetch, Execute, and ReportResult stages |
| `j` / `k` | Scroll within the focused stage list |
| `h` / `l` | Cycle the display field for operations: name, target, mnemonic, build |
| `>` / `<` | Reverse the sort order |
| `Enter` | Open the operation document view for the selected operation |
| `X` | Cancel the selected operation (sends a cancel request to the server) |
| `P` | Pause/unpause the focused stage on this worker (sends a pipeline change request). A paused stage's border turns red |
| `+` | Increase the slot width (concurrency) of the focused stage by 1 |
| `-` | Decrease the slot width of the focused stage by 1 (minimum 1) |

## Operation List View

Shows a list of operations filtered by status (prequeued, queued, dispatched) or by a custom filter.

### How to get here

- From the dashboard: select Prequeue, Queue, or Dispatched and press `Enter`
- From search results
- From an operation's tool invocation or correlated invocations link

### What it shows

The title bar shows the mode (Prequeue / Queue / Dispatched / Filter), the current display field, any active selection filters, and the total operation count.

Each operation row displays one of four fields (cycleable), and elapsed time:

- **name** — Operation ID
- **target** — Build target ID
- **mnemonic** — Action mnemonic (e.g. CppCompile, Javac)
- **build** — Correlated invocations ID

Operations with blue highlighting are stalled (queued but not yet picked up by a worker). Completed operations show their total wall time.

### Navigation and controls

| Key | Action |
|-----|--------|
| `j` / `k` | Scroll through operations |
| `h` / `l` | Cycle display field: name, target, mnemonic, build |
| `>` / `<` | Reverse the sort order |
| `G` | Toggle grouped mode — aggregates operations by the current display field and shows counts |
| `Enter` | Open the selected operation's document view. In grouped mode, filters the list to that group. For tool invocations, drills into that invocation's operations |
| `D` | Toggle debug info (fetch token, stall state) |

### Grouped mode

Press `G` to group operations by the currently displayed field. This is useful for answering questions like "which build target has the most queued actions?" or "which mnemonic dominates the queue?". Results are sorted by count (descending). Press `Enter` on a group to filter the list to just those operations.

## Operation Document View

A detailed, HTML-rendered view of a single operation's full state, with navigable links.

### How to get here

- From an operation list: select an operation and press `Enter`
- From a worker detail: select an operation and press `Enter`
- From search: find an operation by name

### What it shows

- **Request Metadata** — Tool name/version, action ID, tool invocation ID, correlated invocations ID, action mnemonic, target ID, configuration ID
- **Execute Operation Metadata** — Current stage (QUEUED, EXECUTING, COMPLETED), action digest, and partial execution metadata if in progress
- **Queued Operation** — Digest of the queued operation (if applicable)
- **Response** (when complete) — Exit code, stdout/stderr digests, output files and directories, symlinks, execution timing breakdown, cached result status, error messages

### Navigation and controls

| Key | Action |
|-----|--------|
| `j` / `k` / `Tab` | Cycle focus between navigable links (highlighted) |
| `Enter` | Follow the focused link — opens the relevant sub-view (action, worker, tool invocation list, correlated invocations list) |
| `u` | Toggle between rendered HTML and source HTML view |
| `U` | Toggle raw text rendering mode |
| `p` / `Space` | Pause/unpause auto-refresh (useful for inspecting a completed operation without it updating) |

### Available links

Links in the document view are interactive. Depending on the operation state, you can navigate to:

- **Action digest** — Opens the Action view
- **Tool Invocation ID** — Opens an operation list filtered to that invocation
- **Correlated Invocations ID** — Opens a tool invocations list for that build
- **Worker** — Opens the worker detail view
- **stdout / stderr digests** — (placeholder for content view)
- **Output file digests** — (placeholder for content view)
- **Output directory digests** — (placeholder for content view)

## Action View

Shows the details of a REAPI Action, fetched from CAS by digest.

### What it shows

- **Command digest** — Link to view the full command
- **Input Root digest** — Link to browse the input directory tree
- **Platform properties** — Key-value pairs describing the execution platform requirements

### Navigation

| Key | Action |
|-----|--------|
| `j` / `k` / `Tab` | Cycle between links |
| `Enter` | Follow the focused link (Command or Input Root) |

## Command View

Shows the full protobuf text representation of a REAPI Command, fetched from CAS by digest. This includes the argument list, environment variables, output paths, and working directory.

Press `q` or `Escape` to go back.

## Input Root Browser

A tree view for navigating the input directory structure of an action, fetched recursively from CAS.

### What it shows

The full directory tree with file counts per directory. Directories are sorted by total file count (heaviest first), making it easy to find the largest parts of the input tree.

### Navigation

| Key | Action |
|-----|--------|
| `j` / `k` | Move up/down through the tree |
| `l` / Right | Expand the selected directory |
| `h` / Left | Collapse the selected directory |
| `L` / Shift-Right | Recursively expand all children |
| `H` / Shift-Left | Recursively collapse all children |
| `Enter` | Toggle expand/collapse on the selected node |
| `E` | Expand the entire tree |
| `C` | Collapse the entire tree |

## Search

A dialog for finding operations, tool invocations, or correlated invocations by various criteria.

### How to open

Press `/` from the dashboard.

### Using search

The search dialog has three fields you cycle through with `Tab`:

1. **Text input** (focused first) — Type your search value. `Backspace` to delete, `Ctrl-U` to clear
2. **Resource dropdown** — What to search for:
   - `executions` — Individual operations
   - `toolInvocations` — Tool invocation IDs
   - `correlatedInvocations` — Correlated invocation IDs (builds)
3. **Filter dropdown** — What field to match against:
   - `toolInvocationId`
   - `correlatedInvocationsId`
   - `username`
   - `hostname`
   - `name` (direct operation lookup by name)

Use `Up` / `Down` to open and navigate dropdown options, `Enter` to confirm a selection.

Press `Enter` while in the text field to execute the search.

### Search results

Results appear in a scrollable list. The behavior of `Enter` depends on the resource type:

- **executions** — Opens the operation document view
- **toolInvocations** — Drills into that invocation's executions
- **correlatedInvocations** — Drills into that build's tool invocations

Use `PageDown` / `PageUp` for fast scrolling.

### Common search workflows

**Find all operations for a specific build:**
1. Press `/` from the dashboard
2. Tab to Resource, select `correlatedInvocations`
3. Tab to Filter, select `username` (or `hostname`)
4. Tab to Text, type the username, press `Enter`
5. Select a correlated invocation, press `Enter` to see its tool invocations
6. Select a tool invocation, press `Enter` to see its operations

**Look up a specific operation by name:**
1. Press `/`, keep Resource as `executions`
2. Tab to Filter, select `name`
3. Tab to Text, type the operation name, press `Enter`

**Find all operations for a tool invocation:**
1. Press `/`, keep Resource as `executions`
2. Keep Filter as `toolInvocationId`
3. Tab to Text, type the invocation ID, press `Enter`

## Settings

Adjust the UI refresh rate.

### How to open

Press `s` from the dashboard.

### Available settings

| Setting | Description |
|---------|-------------|
| **Frame Limit** | Maximum frames per second for the UI render loop (default: 60). Lower values reduce CPU usage |
| **Update Frame Skip** | Number of frames to skip between data fetches (default: 0). Set higher to reduce API call frequency at the cost of freshness |

Use `Tab` to switch between fields. Type a number and press `Enter` to apply. Press `Escape` to return to the dashboard.
