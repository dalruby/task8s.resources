# task8s.resources

Resources for [task8s](https://task8s.com) — a free, browser-based Kubernetes practice tool.

## Notice

This repository and the associated website are in alpha. Expect changes.

## What is task8s?

[task8s.com](https://task8s.com) lets you practice Kubernetes concepts through interactive scenario sets — no cluster required. You work through scenarios by typing `kubectl` commands, writing YAML manifests, and answering questions, getting instant feedback as you go.

Scenario sets are plain YAML files. You load them into the app directly from your computer or from a URL, and your progress is saved locally in your browser. Alternatively, you can browse a remote directory right from the application in your browser, see [task8s.com - Directory](https://task8s.com/directory).

## What is it not?

task8s does not simulate a real Kubernetes cluster. It is a scenario and question runner with schema validation support for Kubernetes resource types — nothing more.

**The quality of your experience depends entirely on the quality of the scenario set you use.** A well-crafted scenario set can be a great study aid; a poor one will not be. Since you can generate your own with an AI assistant or write them by hand, you are in full control of that.

If you are studying for exams like CKA, CKAD, or CKS, task8s is not a replacement for hands-on cluster practice. Use dedicated exam preparation platforms such as [killer.sh](https://killer.sh) alongside it.

## What is this repository for?

This repository contains resources to help you get the most out of task8s:

| Folder | Contents |
|---|---|
| `examples` | Contains example schema sets that you can use to explore the tool |
| `prompts` | AI prompts for generating scenario set YAML files with tools like ChatGPT, Claude, or Codex |
| `schemas`  | Contains the schemas used for scenario sets |
| `skills`   | Claude Code skills for generating scenario sets interactively |

More resources (scenario sets, guides, examples) will be added here over time.

## Getting started

### I want to create a new scenario set

1. Download the schema (see `schemas`)
2. Generate or write a scenario set YAML file (see `prompts/` for an AI prompt to help)

### I want to work through a scenario set

Navigate to [task8s.com](https://task8s.com) and execute one of these steps:
- Upload a scenario set to the application, or
- Paste the link to the raw scenario set yaml file found on a GitHub repository like this one and press Import, or
- Browse a [directory](https://task8s.com/directory) such as this one through the application and click Load on the scenario set you wish to run

## Creating scenario sets

Scenario sets follow a versioned YAML schema (`v1alpha1`). There are two ways to create one:

### Option 1 — Claude Code skill (recommended)

If you use [Claude Code](https://claude.ai/code), the [`skills/generate-scenario-set.md`](skills/generate-scenario-set.md) skill guides you through an interactive dialogue and then generates a complete, valid YAML file. To use it:

1. Copy [`skills/generate-scenario-set.md`](skills/generate-scenario-set.md) into `.claude/skills/` in your project.
2. In a Claude Code session, run `/generate-scenario-set`.
3. Answer the prompts (topic, difficulty, question types, sub-topics, name).
4. Claude generates the YAML and optionally saves it to a file.

The skill fetches the latest `scenario-set-prompt.md` from this repository at generation time, so it always uses the current schema.

### Option 2 — AI prompt (any assistant)

Use the prompt in [`prompts/scenario-set-prompt.md`](prompts/scenario-set-prompt.md) with any AI assistant (ChatGPT, Claude, Gemini, etc.). Paste it into a new conversation and follow the instructions at the top of the file.

You can also download the JSON Schema from the app to validate your files as you write them.

## Links

- **App:** [task8s.com](https://task8s.com)
- **Claude Code skill:** [`skills/generate-scenario-set.md`](skills/generate-scenario-set.md)
- **AI prompt:** [`prompts/scenario-set-prompt.md`](prompts/scenario-set-prompt.md)
