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
