# How to Generate a task8s Scenario Set

You are an AI agent tasked with generating a **task8s scenario set** — a YAML file that a user will upload to the task8s application to learn Kubernetes concepts interactively.

This document is your complete reference. Read it fully before producing any output. When you are ready, produce a single, valid YAML file. Do not produce JSON. Do not produce markdown code fences around the YAML — just the raw YAML content.

---

## What is a Scenario Set?

A scenario set is a structured learning exercise. The user loads it into task8s and is walked through a series of **scenarios**. Each scenario contains **context text** (background information) and **questions** the user must answer correctly before advancing. No multiple-choice — the user types every answer.

The file format is **YAML**. The schema is strict: unknown properties are rejected, required fields must be present, and types must match exactly.

---

## Top-Level Structure

```yaml
apiVersion: v1alpha1                  # required — must be exactly "v1alpha1"
name: "Your Scenario Set Name"        # required — shown in the UI bottom bar
k8sVersion: "1.29.0"                  # optional — required only if using autoLoad manifest schemas
timing:                               # optional
  enabled: true
  limitSeconds: 3600                  # optional; omit for a count-up timer
introduction:                         # required
  elements: [...]
manifestDefinitions:                  # optional — define reusable manifest types
  - ...
scenarios:                            # required — at least one scenario
  - ...
```

### `apiVersion`
**Required. Must be exactly `v1alpha1`.** This field identifies the schema version of the file, following Kubernetes versioning conventions. The application uses it to apply the correct parsing and validation logic. Always place it as the first field in the file. Do not omit it — files without `apiVersion` will fail validation.

> **BREAKING SCHEMA CHANGE (alpha)** — The `command` question schema has been redesigned. `subcommands` is now an array of `SubcommandToken` objects (not bare strings). `subcommandAliases` has been removed; aliases are now declared inline on each token. `FlagSpec` no longer has `shortForm` or `valueIfPresent` — use `short` and `value`/`oneOf`/`regex` instead. Existing scenario sets using the old schema will fail validation. See the updated `command` section below for the new shape and YAML examples.

### `name`
The display name shown in the UI. Required. Non-empty string.

### `k8sVersion`
A Kubernetes version string (e.g. `"1.29.0"`). Required only when at least one `manifestDefinition` uses `autoLoad: true`. The app uses this to fetch the JSON schema for that manifest kind from the official kubernetes-json-schema repository.

### `timing`
Optional. If omitted, no timer is shown.
- `enabled: true` — activates the timer display
- `limitSeconds` — optional countdown target in seconds. If omitted with `enabled: true`, the timer counts up instead. The timer never blocks the user even if it reaches zero.

---

## Introduction

The introduction is shown on a page between the home screen and the first scenario. Use it to explain what the scenario set covers, set expectations, and link to external resources.

```yaml
introduction:
  elements:
    - type: context
      text: |
        Welcome text here. Explain the topic, what the user will practice,
        and any prerequisites.
    - type: break
    - type: context
      text: "A second paragraph after a visual break."
    - type: references
      links:
        - url: https://kubernetes.io/docs/home/
          text: Kubernetes Documentation
```

### Introduction element types

| Type | Required fields | Description |
|---|---|---|
| `context` | `text` | A paragraph of text. Supports multi-line YAML (`\|`) and **markdown**. |
| `break` | _(none)_ | Adds vertical spacing between context blocks. |
| `image` | `data` | A base64-encoded image data URI. Optional `alt` text. |
| `references` | `links` | A list of external links, each with `url` and `text`. |

---

## Manifest Definitions

If any scenario contains a `manifest` question, you must define the manifest type here. This is a top-level array that maps an `id` to a Kubernetes resource kind so the app knows how to validate it.

```yaml
manifestDefinitions:
  - id: pod-v1           # referenced by manifestDefinitionId in questions
    kind: Pod
    apiVersion: v1
    autoLoad: true       # fetch schema automatically using k8sVersion
```

### Fields

| Field | Required | Description |
|---|---|---|
| `id` | yes | Unique identifier string. Referenced by `manifestDefinitionId` in manifest questions. |
| `kind` | yes | The Kubernetes resource kind (e.g. `Pod`, `Namespace`, `Deployment`). Case-sensitive. |
| `apiVersion` | yes | The API version string (e.g. `v1`, `apps/v1`). |
| `autoLoad` | no | When `true`, the schema is automatically fetched from the kubernetes-json-schema repository. Requires `k8sVersion` at the top level. |
| `schema` | no | An inline JSON Schema object. Use this instead of `autoLoad` for custom or non-standard resources. |

One of `autoLoad: true` or an inline `schema` should be provided for manifest validation to work. If neither is present, the app will report an error when the user tries to validate.

**Use `autoLoad: true` whenever the resource is a standard Kubernetes resource.** This avoids embedding large schema objects in the file.

---

## Scenarios

Each scenario is a self-contained exercise. The user must complete all questions in a scenario before they can advance to the next one.

```yaml
scenarios:
  - title: "Scenario Title"
    category: "Category Name"
    hintsEnabled: true          # optional, default true
    elements:
      - ...
    explanation:
      elements:
        - ...
```

### Scenario fields

| Field | Required | Description |
|---|---|---|
| `title` | yes | Displayed as the scenario heading. |
| `category` | yes | Groups scenarios for statistics. No `<` or `>` characters allowed. |
| `hintsEnabled` | no | Defaults to `true`. Set to `false` to disable hints for this scenario. |
| `elements` | yes | Array of context and question elements. At least one required. |
| `explanation` | yes | Shown after all questions are answered. |

### Category guidance
Use consistent category names across scenarios so the completion statistics are meaningful. Good examples: `"Core Concepts"`, `"Workloads"`, `"Networking"`, `"Storage"`, `"Cluster Architecture"`, `"Security"`.

---

## Scenario Elements

Each element in `elements` is one of four types: **context**, **question**, **manifest**, or **table**.

### Visibility and flow

All four element types follow a single unified rule: **an element is visible only if every question that appears before it in the list has been correctly answered.**

- **Questions** gate what comes after them. Nothing that follows a question is shown until that question is answered correctly. Questions do not gate themselves — the active (unanswered) question is always shown so the user can answer it.
- **Context, manifest, and table** elements never gate what comes after them. They appear as soon as the questions before them are answered. If nothing precedes them, they are visible immediately.

This means the position of an element in the list is significant. A table placed directly after a command question will be hidden until that command is answered — which is the intended effect of making the scenario feel interactive.

**Concrete example** — given this element list:

```
1. context    → "You are working in the kube-system namespace."
2. manifest   → (shows an example pod manifest)
3. command    → "List all pods in kube-system" (question — user must answer)
4. table      → (shows the pod list output)
5. context    → "Notice the coredns pods are Running."
6. command    → "Describe the coredns pod" (question — user must answer)
```

**What the user sees initially (nothing answered yet):**  
Elements 1, 2, and 3 — the context, the manifest, and the first command input.

**After answering command 3 correctly:**  
Elements 4, 5, and 6 appear — the table, the follow-up context, and the second command input.

**Placement guidance:**
- Put context that introduces a task *before* the question it relates to.
- Put tables and manifest displays that show the *result* of a command *after* that command question.
- Put context that comments on a result *after* the question that produces the result — it will appear together with the result table or manifest.

### Context element

```yaml
- type: context
  text: |
    Background information for the user. Explain what they are
    about to do and why it matters.
```

Context elements are always visible and do not block questions from appearing. Use them liberally to give the user enough information to answer the following question. The `text` field supports **markdown** — see the [Markdown support](#markdown-support) section.

### Question element

```yaml
- type: question
  question:
    questionType: <type>
    header: "The question the user sees"
    hint: "Shown after the first wrong attempt"
    correctAnswerInfo: "Extra explanation shown after a correct answer"
    # ... type-specific fields
```

### Manifest element

A read-only syntax-highlighted YAML editor used to **display** a manifest to the user. This is not a question — the user cannot edit it. Use it to show an example manifest, a resource definition, or the result of a previous step.

```yaml
- type: manifest
  header: "The resulting Pod manifest looks like this:"
  content: |
    apiVersion: v1
    kind: Pod
    metadata:
      name: webserver
      namespace: default
    spec:
      containers:
        - name: webserver
          image: nginx:latest
  footer: "Notice that Kubernetes automatically fills in default values for omitted fields."
```

**Fields:**

| Field | Required | Description |
|---|---|---|
| `content` | yes | The YAML text to display. Displayed verbatim with syntax highlighting. |
| `header` | no | Text displayed above the editor. Use it to frame what the user is looking at. |
| `footer` | no | Text displayed below the editor. Use it for additional notes or caveats. |

**When to use it:**
- Show an example manifest before asking the user to write a similar one.
- After a correct answer, show the "ideal" solution or the full resource that would be applied.
- Explain what a manifest looks like for a concept being taught.

### Table element

Displays tabular data inline in the scenario. Use it to simulate the kind of output a user would see after running a command. A table element placed after a command question makes the scenario feel interactive — the user enters the command, it is accepted, and then the "result" appears.

```yaml
- type: table
  style: terminal
  columns: [NAME, READY, STATUS, RESTARTS, AGE]
  rows:
    - [coredns-5d78c9869d-tqfzk, "1/1", Running, "0", 12d]
    - [coredns-5d78c9869d-x9bpz, "1/1", Running, "0", 12d]
    - [etcd-control-plane,        "1/1", Running, "0", 12d]
    - [kube-apiserver-control-plane, "1/1", Running, "0", 12d]
```

**Fields:**

| Field | Required | Description |
|---|---|---|
| `columns` | yes | Array of column header strings. |
| `rows` | yes | Array of row arrays. Each row must have one entry per column. Use `""` for empty cells. |
| `style` | no | `"default"` (styled HTML table, default) or `"terminal"` (kubectl-style monospace output). |

**Styles:**

- **`default`** — A standard HTML table with styled headers and alternating hover states, matching the application's dark theme. Best for structured reference data (e.g. a comparison of resource types, flag descriptions).
- **`terminal`** — Resembles `kubectl` output: monospace font, left-aligned columns with space-based alignment, no borders. Best for simulating command output (e.g. `kubectl get pods`, `kubectl get nodes`).

**When to use it:**
- After a `kubectl get` command question is answered correctly, show the output the user would see.
- Display a reference table of values relevant to the scenario (e.g. namespace names, node statuses).
- Show before/after state to illustrate the effect of a command.

**Row data tips:**
- Values with special characters (slashes, colons, spaces) should be quoted: `"1/1"`, `"0"`.
- Use realistic-looking data — fake pod names, ages, and statuses that match what kubectl would actually show.
- Align terminal-style data mentally: the component computes column widths automatically based on the longest value in each column.

---

## Question Types

Available question types: `single-line`, `multiline`, `command`, `manifest`, `sequence`, `multiple-choice`.

**Preferred types** (use by default): `single-line`, `multiline`, `command`, `manifest`.  
**Use with care**: `sequence` (only when steps are tightly coupled with no intermediate feedback).  
**Not recommended** (available but discouraged): `multiple-choice` (tests recognition, not recall).

### `single-line`

A plain text input. The user's answer must exactly match `expectedAnswer` (or pass the optional CEL expression).

```yaml
questionType: single-line
header: "What flag shorthand is used for --all-namespaces?"
expectedAnswer: "-A"
hint: "It's a single capital letter."
correctAnswerInfo: "-A is shorthand for --all-namespaces and lists resources across every namespace."
```

**Fields:**

| Field | Required | Description |
|---|---|---|
| `expectedAnswer` | yes | The exact string the user must type. |
| `header` | no | The question text shown above the input. |
| `hint` | no | Shown after the first incorrect attempt. |
| `correctAnswerInfo` | no | Shown below the success message after a correct answer. |
| `cel` | no | CEL expression for custom validation (see CEL section). |

---

### `multiline`

A multi-line textarea. Same as `single-line` but for answers that span multiple lines.

```yaml
questionType: multiline
header: "Write the two commands needed to drain and delete a node."
expectedAnswer: |
  kubectl drain node-1 --ignore-daemonsets
  kubectl delete node node-1
```

**Fields:** Same as `single-line`.

---

### `command`

A terminal-style input with a `$` prompt. Validates that the user typed the correct kubectl command with the correct subcommands, positional arguments, and flags. Argument order does not matter — `kubectl get pods -n foo` and `kubectl get pods --namespace foo` are both valid if the spec allows them.

**Top-level fields:**

| Field | Required | Description |
|---|---|---|
| `questionType` | yes | Must be `"command"`. |
| `command` | yes | The `CommandSpec` object describing what is accepted. |
| `initialValue` | no | Pre-filled text shown in the command input when the question loads. Use for cloze-style questions where part of the command is provided and the user must complete it. |
| `header` | no | The question text shown above the input. |
| `hint` | no | Shown after the first incorrect attempt. |
| `correctAnswerInfo` | no | Shown below the success message after a correct answer. |
| `cel` | no | CEL expression for additional validation (see CEL section). |

#### `CommandSpec` object

| Field | Required | Description |
|---|---|---|
| `executable` | yes | The base command (e.g. `kubectl`, `kubeadm`). |
| `strictness` | no | `strict` (default) — rejects unknown flags and unexpected positionals. `permissive` — allows extra flags and positionals beyond those in the spec. |
| `subcommands` | no | Ordered array of `SubcommandToken` objects. The count determines the boundary between subcommands and positional arguments. |
| `positionals` | no | Array of `PositionalSpec` objects checking non-flag tokens after the subcommands. |
| `flags` | no | Array of `FlagSpec` objects. May be empty or omitted. |

#### `SubcommandToken` object

Each entry in `subcommands` describes one expected token in the subcommand chain.

| Field | Required | Description |
|---|---|---|
| `name` | yes | The canonical subcommand name (e.g. `pods`). |
| `aliases` | no | Additional accepted spellings (e.g. `[pod, po]`). The user may type any of these. |

#### `PositionalSpec` object

Each entry in `positionals` constrains one non-flag argument that appears after all subcommands.

| Field | Required | Description |
|---|---|---|
| `index` | no | 0-based position among positional tokens. Defaults to `0`. |
| `required` | no | Whether the positional must be present. Defaults to `true`. |
| `value` | no | The positional must exactly equal this string. |
| `oneOf` | no | The positional must be one of these strings. |
| `regex` | no | The positional must match this regex pattern (partial match). |
| `description` | no | Human-readable label used in validation error messages. |

#### `FlagSpec` object

Each entry in `flags` describes one flag the validator checks for.

| Field | Required | Description |
|---|---|---|
| `name` | yes | The canonical long-form name **without dashes** (e.g. `namespace`, not `--namespace`). |
| `short` | no | Single-character short form **without dash** (e.g. `n` for `-n`). |
| `aliases` | no | Additional accepted long-form names (without dashes). |
| `required` | no | When `true`, the flag must be present. Defaults to `false`. |
| `boolean` | no | When `true`, the flag takes no value (e.g. `--all-namespaces`). Defaults to `false`. |
| `value` | no | When set, the flag's value must exactly equal this string. |
| `oneOf` | no | The flag's value must be one of these strings. |
| `regex` | no | The flag's value must match this regex pattern (partial match). |
| `repeatable` | no | Allow the flag to appear more than once. Defaults to `false`. |
| `minCount` | no | The flag must appear at least this many times. |
| `maxCount` | no | The flag must appear at most this many times. |
| `forbidden` | no | When `true`, the flag must not appear at all. |

**Value constraint rules:**
- `value`, `oneOf`, and `regex` are checked on every occurrence of the flag, whether or not it is `required`.
- This means `required: false` + `value: "default"` expresses "optional, but if supplied the value must equal 'default'".
- `forbidden` blocks a flag regardless of value — use it to prevent a user from bypassing a constraint (e.g. forbidding `--all-namespaces` when a specific namespace is required).
- Flag entries are checked independently; order does not matter.

#### Command question examples

**Example 1 — `kubectl get pods -n default`**

Requires both a subcommand chain and a namespace flag; `pod` and `po` are accepted as aliases for `pods`:

```yaml
questionType: command
header: "List all pods in the default namespace"
hint: "Use the -n flag to target a specific namespace."
correctAnswerInfo: "-n is shorthand for --namespace."
command:
  executable: kubectl
  subcommands:
    - name: get
    - name: pods
      aliases: [pod, po]
  flags:
    - name: namespace
      short: "n"
      required: true
      value: default
```

**Example 2 — `kubectl drain node01 --ignore-daemonsets --delete-emptydir-data`**

Uses `positionals` for the node name and boolean flags (no value):

```yaml
questionType: command
header: "Drain node01 safely, ignoring DaemonSet pods and deleting emptyDir data"
command:
  executable: kubectl
  subcommands:
    - name: drain
  positionals:
    - index: 0
      required: true
      value: node01
      description: node name
  flags:
    - name: ignore-daemonsets
      boolean: true
      required: true
    - name: delete-emptydir-data
      boolean: true
      required: true
```

**Example 3 — `kubectl logs pod-name -c container --previous`**

Short form flag, a positional for the pod name, and a boolean flag:

```yaml
questionType: command
header: "Fetch logs from the 'web' container in pod 'myapp', showing the previous container run"
command:
  executable: kubectl
  subcommands:
    - name: logs
  positionals:
    - index: 0
      required: true
      value: myapp
      description: pod name
  flags:
    - name: container
      short: "c"
      required: true
      value: web
    - name: previous
      short: "p"
      boolean: true
      required: true
```

**Example 4 — `kubectl label nodes node1 env=production --overwrite`** (permissive mode)

Uses `regex` to accept any node name, a positional with `regex` for the label pair, and permissive mode to allow extra flags the author does not want to enumerate:

```yaml
questionType: command
header: "Label node 'node1' with env=production (allow any extra flags)"
command:
  executable: kubectl
  strictness: permissive
  subcommands:
    - name: label
    - name: nodes
  positionals:
    - index: 0
      required: true
      regex: "^node"
      description: node name
    - index: 1
      required: true
      regex: "^env="
      description: env label pair
  flags:
    - name: overwrite
      boolean: true
      required: true
```

**Example 4 — cloze (fill-in-the-blank) using `initialValue`**

The user is given `kubectl get pods` pre-filled and must add the flag that shows output across all namespaces:

```yaml
questionType: command
header: "Add the flag that lists pods across all namespaces"
initialValue: "kubectl get pods "
hint: "There is a boolean flag for this — check kubectl get --help."
correctAnswerInfo: "--all-namespaces (or -A) tells kubectl to show resources from every namespace."
command:
  executable: kubectl
  subcommands:
    - name: get
    - name: pods
      aliases: [pod, po]
  flags:
    - name: all-namespaces
      short: "A"
      boolean: true
      required: true
```

The `initialValue` is displayed in the input when the question loads. The user edits it freely — the final submitted value is validated exactly like any other command answer. Keep the initial value clearly incomplete so the user knows what to add; a trailing space after the partial command is a useful visual cue.

**Caution:** `initialValue` is a difficulty tool, not a shortcut. Do not pre-fill so much of the command that the answer requires no thought. The filled portion should establish context (e.g. the subcommand), not supply the part being tested.

#### When to use `sequence` vs ordered standalone questions

A `sequence` treats all steps as a single question element: nothing after the sequence in the element list is revealed until **every step** is complete. Use a `sequence` only when the steps are tightly coupled and there is nothing meaningful to show between them — for example, a set of commands that together accomplish one atomic task, where showing intermediate state would be confusing.

If you want a context block, table, or manifest element to appear after each individual step is answered correctly, **do not use a sequence**. Write each step as a separate standalone question element instead. The normal element-visibility rule will reveal the follow-up content after each question is answered.

---

### `manifest`

A YAML editor pre-configured for writing a Kubernetes manifest. The user's YAML is validated against the manifest definition's JSON schema and against any `expectedFields` you define.

```yaml
questionType: manifest
header: "Write a Namespace manifest for 'staging'"
manifestDefinitionId: namespace-v1
initialContent: |
  apiVersion: v1
  kind: Namespace
  metadata:
    name:
expectedFields:
  - path: metadata.name
    expectedValue: staging
hint: "Fill in the name field under metadata."
correctAnswerInfo: "A Namespace only needs apiVersion, kind, and metadata.name."
```

**Fields:**

| Field | Required | Description |
|---|---|---|
| `manifestDefinitionId` | yes | Must match the `id` of a `manifestDefinition` entry. |
| `initialContent` | no | Pre-filled YAML shown in the editor when the question loads. Use this to scaffold the structure. |
| `expectedFields` | no | Array of field checks run after schema validation passes. |
| `header` | no | Question text. |
| `hint` | no | Shown after first wrong attempt. |
| `correctAnswerInfo` | no | Shown after correct validation. |

#### `expectedFields`

Each entry checks that a specific field in the manifest has the right value.

```yaml
expectedFields:
  - path: metadata.name
    expectedValue: staging
  - path: spec.containers.0.image
    expectedValue: nginx
```

- `path` uses dot notation. Array indices are numeric (e.g. `spec.containers.0.name`).
- `expectedValue` can be any YAML scalar (string, number, boolean).

**Guidance:** Whether to provide `initialContent` is a deliberate difficulty choice. Providing a scaffold (with the required fields left blank) reduces friction and guides the user toward the correct structure. Omitting it entirely requires the user to recall and type the full manifest from memory, which is harder and more realistic for advanced sets. Match the difficulty to the intent of the scenario.

---

### `sequence`

A multi-step question where each step must be completed before the next is revealed. Steps can be `single-line`, `multiline`, or `command`. Manifest steps are not supported.

```yaml
questionType: sequence
header: "Complete the following kubectl commands in order"
hint: "The -o flag controls output format."
correctAnswerInfo: "Common output formats: wide, json, yaml, jsonpath."
steps:
  - questionType: command
    header: "Get all pods in all namespaces with wide output"
    command:
      executable: kubectl
      subcommands:
        - name: get
        - name: pods
          aliases: [pod, po]
      flags:
        - name: all-namespaces
          short: "A"
          boolean: true
          required: true
        - name: output
          short: "o"
          required: true
          value: wide
  - questionType: single-line
    header: "What flag shorthand is used for --all-namespaces?"
    expectedAnswer: "-A"
```

**Fields:**

| Field | Required | Description |
|---|---|---|
| `steps` | yes | Array of at least 2 child questions. |
| `header` | no | Overall question heading shown above all steps. |
| `hint` | no | Shown after the first wrong attempt on any step. One hint for the whole sequence. |
| `correctAnswerInfo` | no | Shown after all steps are complete. |

**Rules:**
- A sequence must have at least 2 steps.
- Steps support `header` (recommended — helps the user understand each step).
- Steps can have individual `hint` values — these are shown per-step if `hintsEnabled` is on.
- Steps cannot have `cel` expressions at the sequence level. Individual steps can.
- The `hint` on the `sequence` object itself is shared across all steps.

**When not to use a sequence:**
A sequence is treated as a single question element. This means that any context, manifest, or table elements that follow the sequence in the element list are hidden until **all steps** of the sequence are complete — not until each individual step is complete. Completing step 1 does not reveal the next element; only completing the final step does.

If you want to show a table or manifest element after the user correctly answers each individual step, do **not** use a sequence. Instead, write each step as a separate, standalone question element. The natural element visibility flow will then reveal the follow-up table or manifest after each question is answered.

Use a sequence only when the steps are tightly coupled and there is no meaningful intermediate feedback to show between them — for example, a series of commands that together accomplish one atomic task, where showing intermediate state would be confusing or where the result is only meaningful after all steps are done. If any context, manifest, or table element needs to appear between steps, use standalone questions instead.

---

### `multiple-choice`

> **Not recommended as a primary question type.** Multiple-choice questions test recognition rather than recall. For Kubernetes practice, typed answers (commands, manifests, single-line) are almost always a better choice because they mirror real-world usage. Use `multiple-choice` sparingly — for example, to test conceptual understanding where there is no single typed answer, or as a quick knowledge check at the end of a scenario.

A question where the user selects from a list of options. Three modes are supported, controlled by the `answerType` on each choice:

| Mode | How to configure | Behaviour |
|---|---|---|
| **Single correct answer** | Exactly one choice has `answerType: Solution` | Selecting that choice immediately succeeds. Selecting any other choice immediately fails. |
| **Any of several correct** | Multiple choices have `answerType: Solution` | Selecting any one Solution succeeds immediately. Selecting an Invalid fails immediately. |
| **Combined correct answer** | Two or more choices have `answerType: PartOfSolution`, rest are `Invalid` | User selects multiple options then submits. All PartOfSolution options must be selected with no Invalid options. |

> `PartOfSolution` and `Solution` must not be mixed in the same question.

**Fields:**

| Field | Required | Description |
|---|---|---|
| `questionType` | yes | Must be `multiple-choice`. |
| `choices` | yes | Array of at least 2 choice objects. |
| `header` | no | Question text shown above the choices. |
| `hint` | no | Shown after the first wrong attempt (if `hintsEnabled`). |
| `correctAnswerInfo` | no | Shown after a correct selection. |

**Choice fields:**

| Field | Required | Description |
|---|---|---|
| `text` | yes | The label shown on the option button. |
| `description` | no | Optional sub-text shown below the label. |
| `answerType` | yes | `Solution`, `PartOfSolution`, or `Invalid`. |

**Examples:**

Single correct answer:
```yaml
- type: question
  question:
    questionType: multiple-choice
    header: "Which kubectl flag outputs in YAML format?"
    hint: "Think about how you'd inspect a resource in full detail."
    correctAnswerInfo: "'-o yaml' outputs the resource as YAML. '-o json' outputs JSON."
    choices:
      - text: "-o yaml"
        answerType: Solution
      - text: "-o wide"
        answerType: Invalid
      - text: "--format yaml"
        answerType: Invalid
      - text: "-o json"
        answerType: Invalid
```

Combined answer (all must be selected):
```yaml
- type: question
  question:
    questionType: multiple-choice
    header: "Which two flags are required to safely drain a node?"
    hint: "You need to handle both DaemonSets and local storage."
    choices:
      - text: "--ignore-daemonsets"
        description: "Skips pods managed by DaemonSets."
        answerType: PartOfSolution
      - text: "--delete-emptydir-data"
        description: "Allows deletion of pods using emptyDir volumes."
        answerType: PartOfSolution
      - text: "--force"
        description: "Forces deletion of unmanaged pods."
        answerType: Invalid
      - text: "--grace-period=0"
        answerType: Invalid
```

---

### `ordering`

A drag-and-drop question where the user arranges a list of items into the correct sequence. Good for topics like rollout strategies, container lifecycle, kubeadm bootstrap steps, or any process with a defined order.

Items are displayed in a randomised order that is **guaranteed not to equal the correct order**, so the question is never trivially correct on first view.

```yaml
- type: question
  question:
    questionType: ordering
    header: "Put these Deployment rollout steps in the correct order"
    hint: "Think about what Kubernetes checks before it scales down the old ReplicaSet."
    correctAnswerInfo: "Kubernetes waits for the readiness probe to pass before terminating old pods — this ensures zero-downtime rollouts."
    items:
      - id: apply
        text: "kubectl apply is run"
      - id: new-rs
        text: "New ReplicaSet is created"
      - id: ready
        text: "Readiness probe passes on new pods"
      - id: old-down
        text: "Old pods are terminated"
    correctOrder: [apply, new-rs, ready, old-down]
```

**Fields:**

| Field | Required | Description |
|---|---|---|
| `questionType` | yes | Must be `ordering`. |
| `items` | yes | Array of at least 2 items, each with a stable `id` and display `text`. |
| `correctOrder` | yes | Array of `id` values in the correct sequence. Every ID must exist in `items`. |
| `header` | no | Question text shown above the list. |
| `hint` | no | Shown after the first incorrect submission. |
| `correctAnswerInfo` | no | Shown after a correct submission. |

**Item fields:**

| Field | Required | Description |
|---|---|---|
| `id` | yes | Stable internal key used in `correctOrder`. Never shown to the user. Use short, descriptive slugs. |
| `text` | yes | The label shown on the draggable card. |

**Rules:**
- `items` must have at least 2 entries.
- Every ID in `correctOrder` must match an `id` in `items`, and the lengths must match.
- The `id` values are never exposed to the user — they are only used to define and check the order.
- Do not use ordering for questions with a single obviously correct starting point that makes the rest trivial to derive. It works best when multiple steps have plausible alternative orderings.

---

## Explanation

Every scenario requires an `explanation` block. It is displayed after all questions in the scenario are answered correctly. Use it to explain why the answers were correct, provide deeper context, and link to official documentation.

```yaml
explanation:
  elements:
    - type: context
      text: |
        Explain what the user just did and why it works.
        Be thorough — this is a teaching moment.
    - type: references
      links:
        - url: https://kubernetes.io/docs/...
          text: Official documentation link
```

### Explanation element types

| Type | Required fields | Description |
|---|---|---|
| `context` | `text` | A paragraph of explanatory text. Supports **markdown**. |
| `image` | `data` | Base64-encoded image data URI. Optional `alt` text. |
| `references` | `links` | One or more external links (`url` + `text`). |

Note: The explanation does not have a `break` element (unlike the introduction).

---

## Markdown support

**Only `context` elements support markdown.** This applies in three places:
- Introduction `context` elements
- Scenario `context` elements
- Explanation `context` elements

All other text fields — question `header`, `hint`, `correctAnswerInfo`, table headers and cells, reference link text, image `alt`, `name`, `description` — are rendered as **plain text**. Do not use markdown syntax (e.g. `**bold**`, `_italic_`, `## heading`) in those fields; it will appear as literal characters.

### Supported markdown

| Feature | Syntax | Notes |
|---|---|---|
| **Bold** | `**text**` | |
| _Italic_ | `_text_` or `*text*` | |
| Inline code | `` `code` `` | Monospace, lightly highlighted |
| Fenced code block | ` ```yaml ... ``` ` | Syntax-highlighted block |
| Headings | `## Heading` | h1–h3 most useful |
| Unordered list | `- item` | |
| Ordered list | `1. item` | |
| Table | `\| col \| col \|` | Standard GFM table |
| Blockquote | `> text` | Indented, muted style |
| Horizontal rule | `---` | |
| Link | `[text](url)` | Opens in same tab — use sparingly |

### Tips

- Use a YAML block scalar (`|`) for any context that spans multiple lines or contains markdown:
  ```yaml
  - type: context
    text: |
      The pod is in a **CrashLoopBackOff** state. This usually means:

      - The container exits immediately after starting
      - Kubernetes keeps restarting it with exponential back-off

      Run `kubectl describe pod <name>` to see the exit code.
  ```
- Prefer bullet lists over long prose paragraphs for step-by-step explanations.
- Use inline code (backticks) for all command names, flag names, resource names, and field paths.
- Do not nest markdown inside question `header` or `hint` — those fields are plain text.

---

## CEL Expressions (Advanced)

Any question except `sequence` can define a `cel` field. The behavior differs slightly by question type:

- **`single-line` / `multiline`**: CEL *replaces* the default `expectedAnswer` check entirely.
- **`command`**: CEL runs *after* the structural validator passes. Both must succeed — the structural check validates executable, subcommands, positionals, and flags first; CEL is an additional gate for logic the structural validator cannot express. If the structural check fails, CEL is never evaluated.
- **`manifest`**: CEL runs alongside `expectedFields` checks (both must pass).

For `command` questions, CEL receives a rich context — not just `answer`:

| Variable | Type | Description |
|---|---|---|
| `answer` | string | The raw input string. |
| `executable` | string | The parsed executable (e.g. `"kubectl"`). |
| `subcommands` | list | Ordered list of subcommand tokens after the executable. |
| `positionals` | list | Positional arguments after subcommands. |
| `flags` | map | Canonical flag name → value (`true` for boolean, string, or list for repeats). |
| `unknownFlags` | list | Flag names that did not match any FlagSpec. |
| `extras` | list | Tokens after a bare `--` separator. |

The answer is available as the `answer` variable (always a string).

```yaml
questionType: single-line
header: "Name any valid kubectl output format"
cel: "answer in ['json', 'yaml', 'wide', 'name', 'jsonpath']"
```

CEL is useful when:
- Multiple answers are correct (e.g. any of several valid flag combinations)
- You want case-insensitive matching: `answer.lowerAscii() == 'yes'`
- You want to check a prefix: `answer.startsWith('kubectl')`
- You want to check a suffix: `answer.endsWith('.yaml')`

For `command` questions, CEL is rarely needed — the `command` spec handles most cases. Use CEL as a fallback for edge cases the command validator cannot express.

### Supported CEL operations

task8s uses the [`@bufbuild/cel`](https://github.com/bufbuild/cel-go) runtime with the full CEL string extension library enabled. All standard string methods from the CEL spec are available.

**String methods:**

| Method | Example | Description |
|---|---|---|
| `contains(sub)` | `answer.contains('api')` | True if the string contains the substring. |
| `startsWith(prefix)` | `answer.startsWith('kubectl')` | True if the string begins with the given prefix. |
| `endsWith(suffix)` | `answer.endsWith('.yaml')` | True if the string ends with the given suffix. |
| `matches(regex)` | `answer.matches('pods?')` | True if the string contains a match for the regex (partial match). |
| `lowerAscii()` | `answer.lowerAscii() == 'yes'` | Returns the string lowercased (ASCII only). Useful for case-insensitive equality. |
| `upperAscii()` | `answer.upperAscii() == 'YES'` | Returns the string uppercased (ASCII only). |
| `trim()` | `answer.trim() == 'yes'` | Returns the string with leading/trailing whitespace removed. |
| `size()` | `answer.size() > 0` | Returns the string length as an integer. |
| `indexOf(sub)` | `answer.indexOf('get') >= 0` | Returns the index of the first occurrence of `sub`, or `-1` if not found. |
| `indexOf(sub, start)` | `answer.indexOf('get', 5)` | Returns the index of `sub` starting the search at position `start`. |
| `lastIndexOf(sub)` | `answer.lastIndexOf('/')` | Returns the index of the last occurrence of `sub`, or `-1`. |
| `substring(start)` | `answer.substring(8)` | Returns the suffix starting at `start`. |
| `substring(start, end)` | `answer.substring(0, 7) == 'kubectl'` | Returns the slice `[start, end)`. |
| `replace(old, new)` | `answer.replace('-', '_')` | Returns the string with all occurrences of `old` replaced by `new`. |
| `replace(old, new, n)` | `answer.replace('-', '_', 1)` | Replaces up to `n` occurrences. |
| `split(sep)` | `answer.split(' ').size() == 3` | Splits the string on `sep` and returns a list. |
| `split(sep, n)` | `answer.split(' ', 2)` | Splits into at most `n` parts. |
| `join(sep)` | `['a','b'].join('-')` | Joins a list of strings with `sep`. |
| `charAt(i)` | `answer.charAt(0) == 'k'` | Returns the character at index `i` as a single-character string. |

**Operators and functions:**

| Expression | Description |
|---|---|
| `answer == 'exact'` | Exact equality. |
| `answer != 'exact'` | Inequality. |
| `answer in ['a', 'b', 'c']` | Membership in a list of accepted values. |
| `expr1 && expr2` | Logical AND. |
| `expr1 \|\| expr2` | Logical OR. |
| `!expr` | Logical NOT. |
| `size(answer) > n` | Global `size()` function — equivalent to `answer.size()`. |

**Do not use:**
- `contains()` on lists — use `in` instead: `'value' in someList`
- Any CEL extension beyond the standard string library (e.g. math extensions, URL parsing) — not enabled

---

## Hints

- `hint` is optional on all question types.
- Hints are shown after the user submits a **wrong answer for the first time**.
- If `hintsEnabled: false` is set on a scenario, hints are suppressed for all questions in that scenario.
- A sequence has one shared `hint` that appears after any step fails. Individual steps may also have their own hints.
- Once the user answers correctly, the hint is hidden.

### What makes a good hint

A hint points the user toward *where to look* or *what concept to think about* — it does not contain the answer or any part of it.

**Do not** write hints that give the answer away:
- "Use `kubectl get pods -o wide`" — this is the answer
- "The flag is `-o` and the value is `wide`" — this is the answer broken into two pieces
- "Use the namespace flag with value `kube-system`" — still the answer

**Do** write hints that steer without spoiling:
- "kubectl has a flag to control output format — check `kubectl get --help`"
- "Think about which subcommand describes a resource in detail rather than listing it"
- "The namespace this resource lives in is not the default one"
- "Kubernetes has a short-form alias for most resource types"

The hint should make the user think harder, not stop thinking.

---

## Design Principles

Follow these principles when writing scenario sets:

### 1. Context before questions
Add `context` elements before questions to set the scene and provide necessary background — but **never reveal the answer in the context**. Context explains *what situation the user is in* and *why it matters*. It does not show the command, flag, or value the user is expected to type.

Bad (gives the answer away):
```
To list all pods, run: kubectl get pods
```
```
The -o flag controls the output format. Use -o wide for more columns.
```

Good (sets the scene, requires recall):
```
You need to inspect what is running in the default namespace.
```
```
The output of the previous command is too narrow to see node placement.
```

For introductory (easy) scenarios it is acceptable to explain the concept and its syntax in context — but the exact answer must still require the user to apply that knowledge, not copy it verbatim. At harder difficulty levels, context should describe the *situation* without teaching the solution.

### 2. One concept per scenario
Keep each scenario focused on a single command, concept, or task. A scenario with 1–3 questions is ideal. Longer scenarios are acceptable for complex workflows (e.g. drain + cordon + delete), but avoid padding.

### 3. Teach through the explanation
The explanation is as important as the questions. Use it to explain not just what the answer is, but why it works, what the alternatives are, and what can go wrong. Include references to official Kubernetes documentation.

### 4. Use `correctAnswerInfo` for depth
After a correct answer, `correctAnswerInfo` is shown inline. Use it to add a short note that enriches the answer without overwhelming the question itself.

### 5. Questions must require thought
The user should have to think, remember, or understand a concept to answer correctly. A question is too easy if the answer appears verbatim anywhere in the context, hint, `correctAnswerInfo`, or question `header` itself.

- The `header` states the *task or goal*, never the solution. "Run kubectl get pods" is not a question header — "List all pods in the default namespace" is.
- Do not embed the expected value in an example within the `header`. "Use the `-o` flag with value `wide` to show extra columns — what is the full command?" gives the answer away.
- `correctAnswerInfo` is shown *after* a correct answer, so it may explain what the command does and why. It must not be a hint that the user can read before submitting.
- The `hint` must not contain the answer. See the [Hints](#hints) section.

### 6. Prefer `autoLoad` for standard Kubernetes resources
For standard resources (`Pod`, `Namespace`, `Deployment`, `Service`, `ConfigMap`, etc.), always use `autoLoad: true` on the manifest definition and set `k8sVersion` at the top level. Do not embed inline schemas for standard resources — they are very large and the auto-load mechanism handles this cleanly.

### 7. Use `initialContent` / `initialValue` as a difficulty dial
For **manifest** questions, `initialContent` scaffolds the YAML structure. Pre-filling field names but leaving values blank guides the user without giving the answer. No `initialContent` at all requires recall from memory. Never pre-fill the values that `expectedFields` checks.

For **command** questions, `initialValue` pre-fills the input with a partial command. Use it to establish the context (e.g. the subcommand chain) and leave the part being tested blank. The pre-filled portion should make the task clearer, not easier — do not pre-fill the flags or values the question is testing.

### 8. Use manifest and table elements to close the feedback loop
After a command question is answered correctly, place a `table` element (style: `terminal`) showing realistic `kubectl` output. After a manifest question is answered correctly, place a `manifest` element showing the complete ideal manifest. This makes scenarios feel interactive rather than quiz-like — the user sees the consequences of their actions.

### 9. Be precise with flags
For command questions, only specify flags that matter for the question. If a flag is truly optional and any value (or no value) is acceptable, do not include it in the `flags` array. Only define `FlagSpec` entries for flags you actually want to validate.

---

## Difficulty levels

The difficulty of a scenario set is controlled by **how much information the context provides relative to what the questions ask for**. This is a dial, not a binary switch.

Unless the person requesting the scenario set specifies a difficulty, default to **medium**.

The difficulty guidance below applies to the default prompt. The person requesting the scenario set may override it in their own instructions — if they do, their instructions take precedence over these defaults.

### Easy
Context teaches the concept and shows the relevant syntax or command structure. The user applies what was just shown — they must still type the answer themselves, but everything they need is on screen.

- Acceptable to show the flag name and its purpose before asking the user to use it
- Acceptable to show the general form of a command before asking for a specific invocation
- The exact answer must still require composition (combining what was shown with the specific context), not copy-paste

### Medium *(default)*
Context explains the *situation* and the *goal* but does not show the solution. The user must recall or derive the command from their existing knowledge of Kubernetes.

- Describe what needs to be achieved, not how to achieve it
- Acceptable to mention the resource type or concept involved, but not the command or flags
- Hints point toward the right concept area without naming the answer

### Hard
Context describes a situation or problem the user must solve. No syntax guidance is given. The user must already know — or be able to work out — the right approach.

- Context reads like a real-world task or incident: "The payments-api pod is in CrashLoopBackOff. Investigate." or "Create a NetworkPolicy that allows only the frontend to reach the backend on port 8080."
- Do not name the commands or flags involved
- Hints, if present, nudge toward the right Kubernetes concept, not the solution

### Practical rule

Before writing each question, ask: *if the user has only read the context for this scenario, can they answer without already knowing Kubernetes?*

- Easy: yes, because the context taught them
- Medium: no, they must already know the concept — context only frames the task
- Hard: no, they must know the concept *and* the specific command or resource structure

---

## Complete Example

The following is a minimal but complete scenario set with one scenario containing a context block, a command question, and a manifest question:

```yaml
apiVersion: v1alpha1
name: "Kubernetes Basics"
k8sVersion: "1.29.0"

introduction:
  elements:
    - type: context
      text: |
        This set covers the most fundamental Kubernetes operations.
        You will practice listing resources and writing basic manifests.
    - type: references
      links:
        - url: https://kubernetes.io/docs/home/
          text: Kubernetes Documentation

manifestDefinitions:
  - id: pod-v1
    kind: Pod
    apiVersion: v1
    autoLoad: true

scenarios:
  - title: "Your First Pod"
    category: "Core Concepts"
    elements:
      - type: context
        text: |
          A Pod is the smallest deployable unit in Kubernetes.
          It wraps one or more containers that share a network namespace.
      - type: question
        question:
          questionType: command
          header: "List all pods in the default namespace"
          hint: "Use kubectl get with the pods resource type. No namespace flag is needed for the default namespace."
          correctAnswerInfo: "kubectl get pods lists all pods in the current namespace context, which defaults to 'default'."
          command:
            executable: kubectl
            subcommands:
              - name: get
              - name: pods
                aliases: [pod, po]
            flags:
              - name: namespace
                short: "n"
                required: false
                value: default
      - type: question
        question:
          questionType: manifest
          header: "Write a Pod manifest named 'myapp' running the 'nginx' image"
          manifestDefinitionId: pod-v1
          initialContent: |
            apiVersion: v1
            kind: Pod
            metadata:
              name:
            spec:
              containers:
                - name: myapp
                  image:
          expectedFields:
            - path: metadata.name
              expectedValue: myapp
            - path: spec.containers.0.image
              expectedValue: nginx
          hint: "Fill in metadata.name and spec.containers[0].image."
          correctAnswerInfo: "The minimal required fields for a Pod are metadata.name and at least one container with a name and image."
    explanation:
      elements:
        - type: context
          text: |
            Pods are ephemeral — they are not automatically rescheduled if they fail unless
            managed by a higher-level controller like a Deployment or StatefulSet.
            For production workloads, always use a Deployment.
        - type: references
          links:
            - url: https://kubernetes.io/docs/concepts/workloads/pods/
              text: Pods — Kubernetes Documentation
```

---

## Checklist Before Outputting

Before producing the final YAML, verify:

- [ ] `apiVersion: v1alpha1` is the first field in the file
- [ ] `name` is set
- [ ] `k8sVersion` is set if any manifest definition uses `autoLoad: true`
- [ ] Every `manifestDefinition` `id` referenced in a question exists in `manifestDefinitions`
- [ ] Every scenario has `title`, `category`, `elements` (≥ 1), and `explanation`
- [ ] Every `explanation` has at least one element
- [ ] Every `command` question has `executable`; `subcommands` are `SubcommandToken` objects (with `name`, not bare strings); no `subcommandAliases` or `shortForm` fields (use `aliases` and `short` respectively)
- [ ] Every `manifest` question has `manifestDefinitionId`
- [ ] Every `sequence` has at least 2 `steps`
- [ ] Every `multiple-choice` question has at least 2 `choices`; each choice has `text` and `answerType`; `PartOfSolution` and `Solution` are not mixed in the same question
- [ ] Every `ordering` question has at least 2 `items`; every ID in `correctOrder` matches an `id` in `items`; lengths match
- [ ] Every `manifest` element (display type) has `content`
- [ ] Every `table` element has `columns` and `rows`; each row has the same number of entries as `columns`
- [ ] No unknown/extra properties are present (the schema uses `additionalProperties: false` everywhere)
- [ ] The output is valid YAML (correct indentation, quoted strings where needed)
- [ ] Category names are consistent across scenarios
- [ ] Every scenario's `explanation` links to relevant official Kubernetes documentation
- [ ] No question `header` contains the answer or any part of it verbatim
- [ ] No `context` element shows the exact command, flag value, or field value that a following question asks for
- [ ] No `hint` contains the answer — hints point toward a concept or direction, not a solution
- [ ] Difficulty is appropriate to what was requested (default: medium — context frames the task but does not teach the answer)
