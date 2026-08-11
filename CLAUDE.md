# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

See `README.md` for project overview, what the patches disable, what they leave alone, and how to apply them.

## Creating/Updating Patches

When a new WooCommerce version is released, a new patch file is created by adapting the most recent patch:

1. Copy the latest patch file as a starting point
2. Update line numbers and context lines to match the new WooCommerce source
3. Add new patch targets if Automattic has introduced new tracking mechanisms
4. Verify the patch applies cleanly against the new WooCommerce version

Patches use plain unified diff format, generated with `diff -ruN woocommerce-X.Y.Z woocommerce-X.Y.Z-patched` from inside `work/`. There are **no** git-style `a/` and `b/` prefixes: the header paths are the two directory names, so `-p1` strips the leading `woocommerce-X.Y.Z/` component and the rest is relative to the plugin directory (applied with `patch -p1 --directory=woocommerce`).

## Patch Style Conventions

- Use early returns to disable functions rather than deleting code
- Comment out `include`/`require` lines rather than removing them
- Add inline comments explaining why each change is made
- Keep patches conservative and minimal — only disable what is necessary
- The "PATCHED" badge block should be updated with the current date for each new patch

### No CSS masking

Remove things properly or leave them alone. A `display: none` rule is tape over a warning light: the
code still runs, still fetches, still fires its tracking events, and the next release can move the
class out from under the rule with nothing to tell us. Work down this order and stop at the first one
that works:

1. **Unhook it server-side.** Comment out the hook, early-return the method, force the option.
2. **Make the component render nothing.** Seed the flag it already checks, or cut off the data it
   gates on, so it returns `null` and emits no DOM. The `Options.php` hunk is the model: blanking one
   value in a REST response stops the banner mounting at all.
3. **CSS, and only with a written reason.** Say in the changelog why the two options above were not
   available.

The `.woocommerce-marketplace__footer` rule is the sole surviving exception, kept by explicit
decision. Do not treat it as precedent.

### State the removal tier in the changelog

Each new target should say which of the three tiers above it lands in. It is the difference between
a fix that survives an upgrade and one that quietly stops working, and it is the first thing worth
knowing when a surface comes back.

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
