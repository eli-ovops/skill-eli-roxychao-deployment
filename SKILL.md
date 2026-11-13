---
name: eli-roxychao-deployment
description: Governs authorized maintenance, preview deployment, migration, rollback, and acceptance for eli.roxychao.com through its dedicated SSH and atomic-release contract. Use when work mentions Eli preview, eli.roxychao.com, [REDACTED_HOST], eli-site.service, static-current, Next/Payload release, or an Eli deployment.
---

# Eli RoxyChao Deployment

Use this skill together with `website-delivery-sop` for any Eli website action. It is an infrastructure-specific exception, not a general production-deployment method.

## Authority and scope

- The authoritative infrastructure guide is [ELI_ROXYCHAO_CODEX_GUIDE.md](references/ELI_ROXYCHAO_CODEX_GUIDE.md), version 2026-07-24. Its source SHA-256 is recorded in that file.
- The site is shared personal preview infrastructure, never evidence of customer-production deployment or user acceptance.
- Follow current authorization, project `site.yaml`, a unique deploy owner, and the global backup, migration, rollback, and browser-acceptance gates. Static/code preview authorization does not authorize data, full, migration, rollback, server configuration, or customer production.
- Never expose secrets. Do not use root, another account, general sudo, Docker, Caddy, DNS, SSL, firewall/SSH configuration, system upgrades, or other-site resources.

## Temporary preview tiers and evidence

- `static`: authorized static output only; preserve `static-current`/previous target and verify the affected public page and its assets. No service, admin, data, or migration work.
- `code`: authorized code-only Next/Payload release; keep SQLite/media/env outside the package, preserve `current`/previous target, and test only the affected public path plus required health.
- `admin-demo`: isolated-preview admin demonstration only; perform focused smoke of the affected login/protected entry, target flow, related API/media, and blocking console errors. A full route/role matrix needs an explicit task requirement or evidence-backed shared-component risk.
- `data-migration`: any shared SQLite/WAL/SHM, media, schema, or data write; require current explicit authorization, validated DB backup and media snapshot, migration ledger, authorized data-difference check, health, and rollback judgment.

Every new hard stop must cite a concrete clause in this Skill, its guide, project `site.yaml`, DS-002B, or the global rules and state why it applies to the current tier. Unsupported concerns remain risks or `待确认`, not blockers. Use the exact project `packageManager` and lockfile with the normal frozen install path; do not construct an offline toolchain unless network is unavailable and the user or live contract explicitly authorizes a bounded, hash-verified closure. On macOS, inspect PAX/xattr/AppleDouble only in the actual archive bytes and extraction evidence, never by scanning caches or an expanded dependency tree.

## Dedicated access contract

- Only `eli@[REDACTED_HOST]:22`, public-key authentication, and the guide's two permitted source IPs are valid.
- Use explicit identity, `IdentitiesOnly=yes`, `BatchMode=yes`, strict host-key verification, and no agent/X11/port forwarding or user rc. Do not print or persist private-key material.
- Ordinary Shell/PTY is allowed only for this Eli site. There is no forced command, self-hosted publisher, deployment wrapper, or deployment lock; do not create, restore, or invoke any.

## Required workflow

1. Confirm authorization, target, execution environment, project identity, branch, full commit SHA, package manager, lockfile, and one deploy owner.
2. Run the guide's read-only preflight. Stop for concurrent build/migration/switch, unknown Git changes, unclear data/media/env paths, missing migration backup, or failed build/health check.
3. Build in the real Git workspace with the project-declared package manager and lockfile; never upgrade lockfiles or global runtimes by assumption.
4. Deploy static output through `static-staging`/`static-releases` and atomic `static-current`; deploy Next/Payload through separate staging/release and atomic `current`. Never build on `current` or bundle SQLite, media, or env into a release.
5. Before migration, make and verify the SQLite backup and media snapshot. Use only the three exact `eli-site.service` sudo lifecycle commands in the guide; no added parameters or other units.
6. Preserve the previous release target. On failure, atomically restore it; do not overwrite databases with new writes or discard failure evidence.
7. Verify internal/service/public health, release identity, then real-browser desktop/mobile, routes, assets/media, admin flow, API contract, and blocking console errors. HTTP 200, build success, or screenshots are insufficient alone.

## Escalate or stop

Stop and report if the task needs infrastructure changes, credentials, an unapproved high-risk action, ambiguous source IP/identity, an unknown release/data boundary, or a guide/SOP conflict. Keep the Eli infrastructure fact, but preserve the stricter authorization and safety gate.

## References

- [Authoritative Eli guide](references/ELI_ROXYCHAO_CODEX_GUIDE.md)
- [Website delivery SOP](../website-delivery-sop/SKILL.md)
