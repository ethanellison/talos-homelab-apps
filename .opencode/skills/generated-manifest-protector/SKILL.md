---
name: generated-manifest-protector
description: Use this skill to prevent unsafe edits to generated Flux bootstrap manifests such as gotk-* artifacts.
---

# generated-manifest-protector

## When this skill applies

- Any change touching `clusters/*/flux-system/`.

## Protection rules

- Treat `gotk-components.yaml` and `gotk-sync.yaml` as generated artifacts.
- Block direct edits to `gotk-*` files unless the user explicitly requested generated file edits.
- Prefer changing authoritative source manifests instead of generated outputs.

## Output contract

If blocked, report:

- generated file path
- reason for block
- nearest authoritative path to edit
