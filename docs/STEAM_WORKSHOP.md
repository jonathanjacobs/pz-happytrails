# Happy Trails — Steam Workshop Guide

Happy Trails follows the same publication/documentation convention used by Enshrouded Sleep. This document is a release-preparation guide; it does not claim the current pre-alpha build is ready for Workshop publication.

## Current status

**Do not publish as a functional release yet.** SPIKE-001 has not completed the runtime feasibility gate.

## Workshop payload

The distributable Project Zomboid payload lives under:

```text
Contents/mods/pz-happytrails/
```

The repository root also contains project documentation, license/compliance material, and development evidence that should not be confused with the in-game mod payload.

## Before publication

Verify:

- `VERSION` and both package `mod.info` files agree;
- Build 42 minimum version is accurate;
- `README.md`, `CHANGELOG.md`, `docs/ROADMAP.md`, and `docs/RELEASE_CHECKLIST.md` reflect the release state;
- all Lua/source files carry the project licensing convention where appropriate;
- all custom assets have documented provenance in `ASSET_LICENSE.md` / `THIRD_PARTY_NOTICES.md`;
- no decompiled Project Zomboid source or third-party mod source/assets are included;
- the release clearly identifies itself as an unofficial community mod;
- no hidden/unexpected functionality is present;
- the Workshop description matches the actual tested feature set.

## Workshop description source

Maintain the Steam BBCode source in:

```text
workshop-description.bbcode
```

That file should be updated for actual alpha/beta releases rather than allowing the Steam page and repository README to drift independently.

## Preview image

A Workshop-compatible preview image should be added before publication. Keep the release image separate from GitHub social-preview artwork if their format/size requirements differ.
