---
description: Generate feature documentation with mermaid diagrams from implemented code. Produces a reference doc in docs/features/ describing the feature's user flow, system flow, data model, and key files. Use standalone or as the close-out step of /plan-feature.
allowed-tools: Bash, Read, Glob, Grep, Write, Edit
argument-hint: "<feature-name>"
---

Invoke the `doc-feature` skill. Pass the feature name as the argument (kebab-case matches the plan file and the output file name). Follow the skill's workflow: analyze the implementation, generate the document, review with user.
