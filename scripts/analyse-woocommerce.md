# WooCommerce Analysis Runbook

This document guides Claude Code through analysing a WooCommerce release for patch targets — code that phones home, tracks users, pushes upsells, or wastes server resources without benefiting the site operator.

## Prerequisites

Run `./scripts/prepare-analysis.sh` first. You should have:
- `work/woocommerce-X.Y.Z/` — clean source
- `work/woocommerce-X.Y.Z-patched/` — with best-match patch applied

If the patch applied with rejects, resolve those first before continuing.

## How to use this runbook

Start a Claude Code session and say:

> Follow the analysis runbook in `scripts/analyse-woocommerce.md` against the working directories in `work/`.

## Phase 1: Understand the current patch

Read the existing patch that was applied. Identify every file and function it modifies. This is your baseline — you need to know what's already handled so you don't flag known targets.

List each patch target with:
- File path
- Function/method name
- What the patch does (early return, comment out, stub, etc.)

## Phase 2: Search for new outbound HTTP calls

Search the **clean** source directory for patterns that make outbound HTTP requests. These are the highest-value targets — they block PHP workers and leak data.

### Search patterns

```
wp_remote_get
wp_remote_post
wp_remote_request
wp_remote_head
wp_safe_remote_get
wp_safe_remote_post
wp_safe_remote_request
wp_safe_remote_head
Requests::request
WP_Http
```

For each match:
1. Read the surrounding function to understand what it does
2. Determine the destination URL — is it `woocommerce.com`, `pixel.wp.com`, `tracking.woocommerce.com`, or another Automattic domain?
3. Determine when it fires — on every page load? On cron? On specific admin actions?
4. Is this already patched?
5. Is this required for core e-commerce functionality (orders, payments, shipping rates, plugin updates)?

**Flag as a candidate** if the call:
- Goes to a tracking/telemetry endpoint
- Fetches marketing/upsell data
- Runs on every page load or admin load
- Is not essential for e-commerce operations

**Do NOT flag** if the call:
- Fetches plugin/extension update information (`WC_Helper_Updater`)
- Validates subscriptions/licences
- Is part of a payment gateway transaction
- Is triggered only by explicit user action (e.g. clicking "check for updates")

## Phase 3: Search for scheduled tasks (cron)

Search for WP cron registrations that phone home:

```
wp_schedule_event
wp_schedule_single_event
as_schedule_single_action
as_schedule_recurring_action
WC()->queue()->schedule
WC()->queue()->add
```

For each match, determine:
- What action does it schedule?
- Does that action make outbound HTTP calls?
- Is it already patched?

## Phase 4: Search for JavaScript tracking

Search the **clean** source for JS that sends data to external domains:

```
fetch(
XMLHttpRequest
navigator.sendBeacon
pixel.wp.com
tracking.woocommerce.com
stats.wp.com
woocommerce.com/wp-json
.tracks.
wcTracks
```

Search in: `assets/js/`, `src/`, `client/`, and any other JS/TS directories.

## Phase 5: Search for admin AJAX handlers that phone home

Search for `wp_ajax_` actions that make outbound calls:

```
add_action.*wp_ajax_
```

Cross-reference with Phase 2 results — do any of these handlers trigger outbound HTTP calls?

## Phase 6: Search for option defaults that enable tracking

Search for options that default to enabled/yes and relate to tracking or data sharing:

```
add_option.*tracking
add_option.*telemetry
add_option.*marketplace
update_option.*tracking
woocommerce_feature_.*_enabled
```

Check these against the options enforcement block in the current patch. Are there new options that should be forced to `no`?

## Phase 7: Search for data collection and remote logging

```
remote.?log
RemoteLogger
wc_get_logger.*remote
send_log
error_reporting.*remote
```

## Phase 8: Search for new REST endpoint registrations

Look for REST endpoints that allow remote control or data extraction:

```
register_rest_route.*wccom
register_rest_route.*woocommerce.com
register_rest_route.*connect
register_rest_route.*tracker
register_rest_route.*telemetry
```

## Phase 9: Review new/changed files since last patched version

If the clean source version is newer than the patch version, identify files that have changed between versions. Focus on:
- New files in `includes/tracks/`, `includes/wccom-site/`, `src/Internal/Admin/`
- New `DataSourcePoller` classes
- New cron handlers
- New admin page controllers that fetch remote data

## Phase 10: Compile findings

For each new candidate, document:
1. **File** — path relative to WooCommerce root
2. **Function/method** — the specific function to patch
3. **What it does** — brief description of the unwanted behaviour
4. **Impact** — how it affects performance or privacy (e.g. "blocks PHP worker for 2-5s on every admin page load")
5. **Recommended patch** — the specific change (early return, comment out, stub return value)
6. **Risk** — what could break if this is patched incorrectly

## Phase 11: Apply candidate patches and generate diff

For each approved candidate:
1. Edit the file in the `-patched/` directory
2. Follow the patch style conventions from CLAUDE.md
3. Update the PATCHED badge date in `woocommerce.php`

When all changes are applied, generate the candidate patch:

```bash
cd work/
diff -ruN woocommerce-X.Y.Z woocommerce-X.Y.Z-patched > woocommerce-X.Y.Z-candidate.patch
```

The candidate patch will be in `work/woocommerce-X.Y.Z-candidate.patch` for manual testing.
