# Feature Documentation

This directory contains user-facing documentation for Nocturnal features.

## Audience

People **using or configuring** Nocturnal — not people changing its code.

## Shipping

These pages are the **source of truth** for the bundled product-docs skill.
`scripts/gen-docs-skill.sh` copies every `*.md` here (except this README)
into `nocturnal-home/skills/nocturnal/references/` and regenerates the
skill index. Packaging (`scripts/pkg-release.sh`, release CI) runs the
generator before tarring `nocturnal-home/`. After editing a feature page,
re-run:

```bash
scripts/gen-docs-skill.sh
```

so local installs that use `nocturnal-home/` directly stay in sync.

## What goes here

One file per feature. Explain:

- **What** the feature does
- **How** to configure and use it (with examples)
- **Why** it works the way it does, when that's not obvious

Keep it practical. Avoid implementation details — module names, code paths,
and data structures belong with the source code.

## What does NOT go here

- Internal architecture deep-dives (these stay in the developer docs that ship with the source code)
- This README (author meta only — not copied into the skill)

## Naming

Use the feature name, lowercase, hyphenated. One word per feature when possible.

    multi-provider-aliases.md
    auto-compaction.md
    sandbox.md
