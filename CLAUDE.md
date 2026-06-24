# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

See `README.md` for project overview, what the patches disable, what they leave alone, and how to apply them.

## Creating/Updating Patches

When a new WooCommerce version is released, a new patch file is created by adapting the most recent patch:

1. Copy the latest patch file as a starting point
2. Update line numbers and context lines to match the new WooCommerce source
3. Add new patch targets if Automattic has introduced new tracking mechanisms
4. Verify the patch applies cleanly against the new WooCommerce version

Patches use unified diff format with `a/` and `b/` prefixes (git-style). Paths inside the patch are relative to the `woocommerce/` plugin directory (applied with `--directory=woocommerce`).

## Patch Style Conventions

- Use early returns to disable functions rather than deleting code
- Comment out `include`/`require` lines rather than removing them
- Add inline comments explaining why each change is made
- Keep patches conservative and minimal — only disable what is necessary
- The "PATCHED" badge block should be updated with the current date for each new patch

## Analysis Workflow

To analyse a new WooCommerce release for patch targets:

1. Run `./scripts/prepare-analysis.sh` — downloads latest WooCommerce, extracts, detects version, creates clean + patched directories in `work/`, and applies the best-match patch
2. Follow the runbook in `scripts/analyse-woocommerce.md` to search for new targets
3. Candidate patches are saved to `work/woocommerce-X.Y.Z-candidate.patch` for manual testing before promotion to `patches/`

## Masking Client/Customer Data

This repo is public. Before committing anything, mask all client- or customer-identifying details
that come from real sites — most often when pasting sample log data, error traces, file paths, or
incident notes (e.g. from a hand-off):

- **Public/client domain names → `example.com`** (and `example.org`/`example.net`/`example1.com`
  etc. if you need to distinguish more than one). This is the primary rule.
- Apply it everywhere it lands: `patches/`, `CHANGELOG.md`, `docs/`, and any note moved into
  `archive/`. The masked value is what gets committed — never the real domain.
- Internal host aliases (e.g. `hhw7`) are not client-identifying and may be left as-is.
- If a real domain slips into a commit, it must be scrubbed from git **history** (interactive-style
  rewrite + `git push --force-with-lease`), not just fixed in a follow-up commit — the old commit
  still exposes it otherwise.
