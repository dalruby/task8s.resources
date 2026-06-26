# task8s Scenario Set Generator

Generate a valid `v1alpha1` scenario set YAML file for the task8s application.

The schema must be in your context. If it is not, ask for it before proceeding — all structural rules, field names, types, and constraints are defined there.

Output raw YAML only. No code fences. No prose before or after.

---

## Difficulty

Default to **medium** unless the user specifies otherwise.

- **Easy** — context teaches the concept and shows the relevant syntax. The user applies what was just shown; they must still compose the answer, not copy it verbatim.
- **Medium** — context frames the situation and goal. No syntax shown. The user must recall the answer from prior knowledge.
- **Hard** — context describes a task or problem. No guidance of any kind. The user must know the command, flags, and resource structure already.

---

## Ground rules

**Context elements**
- Set the scene; never reveal the answer.
- At easy difficulty, showing syntax form is acceptable. At medium and above, describe the goal — not how to achieve it.

**Question headers**
- State the task or goal. Never embed the answer or any part of it.
- Plain text only — no markdown.

**Hints**
- Point toward the right concept or direction. Never name the answer, flag value, or command.
- Plain text only — no markdown.

**`correctAnswerInfo`**
- Adds depth after a correct answer. May reference and explain the answer.
- Plain text only — no markdown.

**Markdown**
- Supported only in `context` elements (introduction, scenario, explanation). Everywhere else is plain text.

**Command questions**
- After every command question, add a `table` element (`style: terminal`) showing realistic kubectl-style output, if applicable.

**Manifest questions**
- Use `autoLoad: true` for standard Kubernetes resources. Set `k8sVersion` at the top level when doing so.
- After every manifest question, add a `manifest` display element showing the complete correct manifest.
- `expectedFields` paths use dot notation. For keys that contain literal dots (e.g. Kubernetes annotations), wrap that segment in square brackets: `metadata.annotations[storageclass.kubernetes.io/is-default-class]`.
- In `cel` expressions, two variables are available: `answer` (the raw YAML string) and `manifest` (the parsed object). Use `answer.contains(...)` for substring checks on dotted annotation keys; use `manifest.metadata.name == 'foo'` for structural field checks.

**`multiple-choice`**
- Tests recognition, not recall. Only use it when no typed answer type fits, of when explicitly requested by the user.

**Explanations**
- Every scenario explanation must include a `references` element linking to relevant documentation.
