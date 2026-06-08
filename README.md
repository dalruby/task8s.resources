# task8s.resources

Resources for [task8s](https://task8s.com) — a free, browser-based Kubernetes practice tool.

## Notice

This repository and the associated website are in alpha. Expect changes.

## What is task8s?

[task8s.com](https://task8s.com) lets you practice Kubernetes concepts through interactive scenario sets — no cluster required. You work through scenarios by typing `kubectl` commands, writing YAML manifests, and answering questions, getting instant feedback as you go.

Scenario sets are plain YAML files. You load them into the app directly from your computer or from a URL, and your progress is saved locally in your browser.

## What is it not?

task8s does not simulate a real Kubernetes cluster. It is a scenario and question runner with schema validation support for Kubernetes resource types — nothing more.

**The quality of your experience depends entirely on the quality of the scenario set you use.** A well-crafted scenario set can be a great study aid; a poor one will not be. Since you can generate your own with an AI assistant or write them by hand, you are in full control of that.

If you are studying for exams like CKA, CKAD, or CKS, task8s is not a replacement for hands-on cluster practice. Use dedicated exam preparation platforms such as [killer.sh](https://killer.sh) alongside it.

## What is this repository for?

This repository contains resources to help you get the most out of task8s:

| Folder | Contents |
|---|---|
| `examples` | Contains example schema sets that you can use to explore the tool |
| `prompts/` | AI prompts for generating scenario set YAML files with tools like ChatGPT, Claude, or Codex |
| `schemas`  | Contains the schemas used for scenario sets |

More resources (scenario sets, guides, examples) will be added here over time.

## Getting started

1. Visit [task8s.com](https://task8s.com)
2. Generate or write a scenario set YAML file (see `prompts/` for an AI prompt to help)
3. Upload it in the app, or paste the raw file URL directly into the import field
4. Work through the scenarios and track your progress

## Creating scenario sets

Scenario sets follow a versioned YAML schema (`v1alpha1`). The easiest way to create one is with an AI assistant using the prompt in [`prompts/scenario-set-prompt.md`](prompts/scenario-set-prompt.md).

You can also download the JSON Schema from the app to validate your files as you write them.

## Links

- **App:** [task8s.com](https://task8s.com)
- **AI prompt:** [`prompts/scenario-set-prompt.md`](prompts/scenario-set-prompt.md)
