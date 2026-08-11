# Changelog

All notable changes to the patch set are documented here, grouped by WooCommerce version.

---

## 11.0.1 — 2026-08-11

Bump-only. 11.0.1 is a security and maintenance point release shipped six days after 11.0.0. The 11.0.0 patch applied cleanly against it with zero rejects, and the per-version patch was regenerated so it applies with no offset on a fresh extraction. All 185 changed lines are byte-identical to the 11.0.0 patch apart from the `PATCHED` badge, which is bumped to `2026-08-11`. The file count stays at 22.

**Only two patched files were touched upstream, and neither collides.** `woocommerce.php` changed the version string only, well clear of the badge block appended at the end. `includes/class-woocommerce.php` took three changes: the version string, a new `OrderLogsCleanupHelper` container registration, and the fix behind "Fixed admin settings initialization" ([#67532](https://github.com/woocommerce/woocommerce/pull/67532)) — `register_wp_admin_settings()` is now hooked to `admin_init` as well as `rest_api_init`, with a re-entrancy guard inside the method. That landed a few lines below our `WC_Site_Tracking` hunk and shifted our five hunks by 1 to 13 lines, but no hunk overlaps it. Both files were re-checked after applying: the upstream additions and our changes are all present in the patched tree.

**Analysis — no new phone-home targets.** Full-tree pattern counts are identical between clean 11.0.0 and clean 11.0.1: `wp_remote_*` (47 files), `pixel.wp.com` (3), `tracking.woocommerce.com` (1), `public-api.wordpress.com` (10), `WC_Tracks::` (22), `DataSourcePoller` (20). None of the changed files adds an outbound call, a Tracks event, or an Automattic endpoint. The changelog is mostly hardening: authorization checks on the Marketplace subscription activate endpoint and the review-order shortcode, Store API cart-token resolution, session cookie hashing, and Analytics argument validation. The marketplace fix is in `includes/admin/helper/class-wc-helper-subscriptions-api.php`, which we do not patch; `class-wc-helper.php`, which we do, is unchanged from 11.0.0.

**Noted, not actioned — per-order place-order debug logging.** `wc_log_order_step()` (`includes/wc-order-step-logger-functions.php`) writes a DEBUG log entry with a `debug_backtrace()` for each checkout step, stamps `_debug_log_source` meta on the order, and on a clean run schedules the file for deletion. It has been there since 9.9.0, and it is on by default: the logging `level_threshold` default is `none`, which `WC_Log_Levels::get_level_severity()` maps to severity `0`, so `should_handle()` passes every level including DEBUG. 11.0.1 adds a supported off switch, `woocommerce_order_step_logging_enabled` (default `true`), and bounds the cleanup so the daily job reschedules until the backlog drains instead of stopping at 100 files ([#67410](https://github.com/woocommerce/woocommerce/pull/67410)). This is local disk churn on the checkout critical path, not telemetry, so it sits outside what this patch set targets and is left alone. Worth revisiting as a one-line `add_filter( 'woocommerce_order_step_logging_enabled', '__return_false' )` if log volume becomes a problem on a busy store.

Verified with a dry-run against a pristine 11.0.1 extraction: zero rejects, zero fuzz, zero offset, and `php -l` clean across all 21 patched PHP files. Both inline-script blocks in `WCAdminAssets.php` still carry the load-bearing leading `;`. Not yet validated on a live site.

---

## 11.0.0 — 2026-08-04

**Revised 2026-08-10** to add one new target (below). The WooCommerce-source analysis in the following paragraphs is unchanged.

**New target — Marketplace upsell surfaces (`src/Internal/Admin/WCAdminAssets.php`).** The Extensions screen (`admin.php?page=wc-admin&path=/extensions`) carries the same woocommerce.com advert twice. At the top, a four-slide rotating carousel (`.woocommerce-marketplace__banner`): "30-day money-back guarantee", "Get help when you need it", "Products you can trust", "Support the ecosystem". At the bottom, a footer (`.woocommerce-marketplace__footer`) headed "Hundreds of vetted products and services. Unlimited potential." repeating those four with longer blurbs. The first three link out to woocommerce.com in both places, firing `marketplace_banner_link_click` / `marketplace_footer_link_click` Tracks events; dismissing the carousel fires `marketplace_features_banner_dismissed`.

Confirmed pure upsell, not an admin notification surface. Both are hardcoded in the compiled `assets/client/admin/app/index.js` bundle — the carousel as a `const` array of static titles plus inline base64 SVG icons, the footer as a static component in the same webpack module — with no `apiFetch`, no remote spec, no `DataSourcePoller`, no PHP hook and no filter feeding either (`grep` for `featuredBanner` / `marketplace_features_banner` across all PHP returns nothing). The carousel renders only when the route path contains `/extensions`, and both sit outside the notices/store-alerts subtree, so there is no code path by which a store notice or admin alert could ever appear in them.

There is nothing to unhook server-side — both live entirely in the app bundle, which is minified onto a single 248 KB line and so is not practically patchable as a diff. Suppressed on the client instead: an inline style on `WC_ADMIN_APP` hides both classes, and an inline script (`before`) additionally seeds `localStorage.wc_featuredBannerDismissed = 'true'` — the carousel's own dismissal flag — so it returns `null` and emits no DOM at all, the CSS merely covering the flash before that effect runs. Equivalent to the merchant having clicked the built-in X, minus the Tracks event. Each class has exactly one use in the bundle and one rule block in the stylesheet, so the CSS cannot catch anything else. Note the flag persists in the browser: if the patch is later removed, clear the `wc_featuredBannerDismissed` localStorage key to bring the carousel back. The `PATCHED` badge is bumped to `2026-08-10`. Re-verified with a dry-run and a real apply against a fresh 11.0.0 extraction: zero rejects, zero fuzz, `php -l` clean, and confirmed on a live 11.0.0 dev site.

**Both inline snippets open with a leading `;`, and it is load-bearing.** WordPress concatenates every `before` (and every `after`) inline script for a handle with newlines, and `src/Admin/Features/Features.php:275` emits `'window.wcAdminFeatures = ' . wp_json_encode( … )` with **no trailing semicolon**. Automatic semicolon insertion does not break that statement: `{…}` followed by a newline and `(` parses as a call expression with the object literal as the callee. The first revision of this patch omitted the `;` and took the whole Extensions screen down with `Uncaught TypeError: ({'activity-panels':true, …}) is not a function` — `window.wcAdminFeatures` never got assigned, so wc-admin died on first property access and rendered "Oops, something went wrong". Fixed the same day. The same guard was added to the 10.9.4 Jetpack `apiFetch` interceptor, which shares the failure mode but had not been hit because nothing else registers an `after` script on `wp-api-fetch`. Do not strip either semicolon.

Not bump-only, but close: **two hunks retired, two new ones added**, and no new phone-home targets in a major release that ships a lot of new code.

**Two hunks dropped — Automattic fixed this upstream (`src/Internal/Admin/Settings.php`, `src/Internal/Admin/Loader.php`).** Both files carried a single hunk commenting out `$preload_data_endpoints['jetpackStatus'] = '/jetpack/v4/connection'`. 11.0.0 removes that REST preload and replaces it with a direct in-process call to `REST_Connector::connection_status( false )`, citing the same performance problem we were patching around ([#41092](https://github.com/woocommerce/woocommerce/pull/41092)). The replacement reads local options only — no HTTP — so the target no longer exists and both files drop out of the patch (23 files → 21, before the additions below). The client-side interceptor added in 10.9.4 is **still required**: the wc-admin `isJetpackConnected` resolver is unchanged and still calls `apiFetch` for that path, building it from a `"/jetpack/v4"` constant rather than a literal, so the regex in `WCAdminAssets.php` still matches.

**New target — variation gallery canary rollout (`src/Internal/VariationGallery/Package.php`).** 11.0.0 changed the `variation_gallery` feature's `enabled_by_default` from a plain `false` to `Package::is_in_canary_cohort()`, which returns true for variant buckets 1–6 of 120 — so roughly **5% of stores silently switch their product gallery UI on upgrade, with no merchant action and no notice**. Patched with an early `return false` in `is_in_canary_cohort()`, which restores the 10.9.x default in three places at once: the `is_enabled()` fallback, the `FeaturesController` `enabled_by_default` value, and the telemetry cohort label. Deliberately patched there rather than in `is_enabled()` — an explicit `yes` on `wc_feature_woocommerce_additional_variation_images_enabled` still wins, so a merchant who opts in via Settings → Advanced → Features keeps the feature. `Package::init()` early-returns when disabled, so the Additional Variation Images DB migration never fires either.

**New target — `woocommerce_remote_variant_assignment` forced to `0`.** This option is the sticky random 1–120 bucket assigned by `add_woocommerce_remote_variant()` (hooked to `woocommerce_installed` **and** `woocommerce_updated`) that Remote Spec Engines test with a numeric `range` rule to roll things out to a percentage of stores. Despite the name nothing about the value is fetched remotely — it is a local `wp_rand( 1, 120 )`; what is remote is the spec that tests it. `0` sits below every range core defines, so the store matches no cohort: the gallery canary (1–6), the MailPoet/Klaviyo recommendation split (1–84 / 85–120) and the TikTok/Pinterest split (1–60 / 61–120) all stop matching. Note this is a slightly wider blast radius than the gallery fix alone — previously one of each recommendation pair would show, now neither does, which is consistent with what the rest of the patch set does to marketplace suggestions. Added as its own block rather than to the `$options_to_no` loop, whose `FILTER_VALIDATE_BOOLEAN` guard would read `"42"` as false and skip it. The guard writes when the option is **absent** as well as when it is non-zero: `add_woocommerce_remote_variant()` only rolls when `get_option()` returns `false`, so storing `'0'` pre-empts the roll permanently instead of letting every update re-roll it. The string comparison keeps it idempotent — no write on every page load.

**Analysis — no new phone-home targets.** 11.0.0 adds a lot of new code (`src/Internal/AbandonedCartRecovery`, `POS`, `CustomerEmailVerification`, `Tax`, `ShopperLists/Privacy`, `ProductFeed/Mapping`, `Caches`, `CLI/Migrator/Platforms/Webflow`). None contains a single `wp_remote_*`, Tracks call, or Automattic endpoint. Feature gating confirms the rest: abandoned cart recovery is `is_experimental => true` / `enabled_by_default => false`; Point of Sale was already enabled by default in 10.9.4 and 11.0.0 only hides its UI (`disable_ui => true`) and marks it `deprecated_since => 11.0.0`. Full-tree delta sweeps found no new `DataSourcePoller` classes, no new tracking-pixel domains, and no remote-logging changes. Only two files gained outbound HTTP, neither a target:
- `src/Internal/Admin/SiteHealth.php` — a loopback `wp_safe_remote_get` against the store's *own* uploads directory to check it is protected, transient-cached for a day. This is a **net win**: WooCommerce moved a batch of admin notices into Site Health tests (hence 14 deleted `html-notice-*.php` views and `includes/admin/class-wc-admin-notices.php` losing its outbound HTTP entirely), so the check moved off every admin page load onto the Site Health screen.
- `src/Internal/CLI/Migrator/Platforms/Webflow/WebflowClient.php` — WP-CLI migrator, fires only on explicit user command.

`src/Internal/Admin/TaxSettingsRecommendations.php` registers a new REST route but it is a local dismissal endpoint with no outbound call; left alone.

The load-bearing `record_gateway_event()` fatal-fix carried forward intact and was verified applied. The `PATCHED` badge is bumped to `2026-08-04`. Patch verified with a dry-run **and** a real apply against a fresh 11.0.0 extraction: zero rejects, and `php -l` clean across all 21 patched PHP files.

---

## 10.9.4 — 2026-07-15

**Revised 2026-07-15** to add one new target (below). The original 2026-07-08 patch was bump-only; the WooCommerce-source analysis in the following paragraphs is unchanged.

**New target — client-side Jetpack connection interceptor (`src/Internal/Admin/WCAdminAssets.php`).** Because our patch disables Jetpack connection init (`init_jetpack_connection_config` commented out), the `jetpack/v4/connection` REST route is never registered. But the wc-admin `isJetpackConnected` resolver calls `apiFetch( '/jetpack/v4/connection' )` **unconditionally on every wc-admin page load** (Analytics, etc.) — so that call returned a **404** on every visit: access-log noise, and a needless full WordPress/PHP boot to answer a request whose route we deliberately removed. Note this is a *local* endpoint (`/wp-json/…` is served by the site itself); it is not a phone-home to Jetpack/wp.com.

The fix adds an `@wordpress/api-fetch` middleware (via `wp_add_inline_script( 'wp-api-fetch', … )` in `enqueue_assets()`) that short-circuits **only** the exact `/jetpack/v4/connection` path to a static "not connected" object matching core's `REST_Connector::connection_status()` shape; every other path falls through to `next()` unchanged. The request never leaves the browser — no network round-trip, no PHP, no 404. A stub REST route was considered and rejected: any `/wp-json/` route still boots the full WP stack before the handler runs, so killing the request client-side is strictly better on every axis. Verified on a live 10.9.4 install — the `jetpack/v4/connection` hit count in the access log stays flat across repeated Analytics reloads (no new requests logged). The `PATCHED` badge is bumped to `2026-07-15`.

---

Bump-only (WooCommerce source). Automattic's changelog lists a single fix (VAT-exempt not applied during block checkout for logged-in users, [#66342](https://github.com/woocommerce/woocommerce/pull/66342)). The 10.9.3 patch applied cleanly against 10.9.4 with zero rejects and zero fuzz/offset; the per-version patch was regenerated so it applies with no offset on a fresh extraction. The load-bearing `record_gateway_event()` fatal-fix carried forward intact — `includes/class-wc-payment-gateways.php` is unchanged from 10.9.3, so its line numbers are identical.

A full clean-tree diff (10.9.3 → 10.9.4) confirmed the change is confined to non-target code: only two source files carry real changes — `includes/class-wc-customer.php` (the VAT fix — removes the `save()` no-op-change guard added in 10.9.0) and `includes/class-woocommerce.php` (version string only, not inside any hunk). The remaining diffs are the version bump (`woocommerce.php`), regenerated i18n `.pot`, and Composer/Jetpack autoload maps. No new outbound-HTTP, tracking, or telemetry targets. No review action needed beyond the bump.

## 10.9.3 — 2026-07-04

Bump-only release. Automattic skipped 10.9.2 as a public build (its only trace is a `10.9.2` DB migration entry). The 10.9.1 patch applied cleanly against 10.9.3 with zero rejects and zero fuzz; the per-version patch was regenerated so it applies with no offset on a fresh extraction. The load-bearing `record_gateway_event()` fatal-fix carried forward intact — `includes/class-wc-payment-gateways.php` is unchanged from 10.9.1, so its line numbers are identical.

Review of the 10.9.1 → 10.9.3 changes found no new patch targets active on a default install. The full-tree phone-home pattern counts (`wp_remote_*`, `pixel.wp.com`, `tracking.woocommerce.com`) are identical between the two versions. The meaningful diffs:
- `src/Internal/PushNotifications/PushNotifications.php` + `FeaturesController.php` — the `push_notifications` feature flag was deprecated in 10.9.2 and the feature is now "always enabled" (no longer gated by the experimental flag). **Still not a target:** `should_be_enabled()` now gates purely on `JetpackConnectionManager->is_connected()`, and our patch already severs Jetpack connection init (`init_jetpack_connection_config` commented out, `Users_Connection_Admin` disabled), so the dispatcher stays inert. A new `woocommerce_enhanced_push_notifications_disabled` filter also lets a store force it off. The `wc_update_10902_remove_deprecated_push_notifications_option()` migration just deletes the now-unused option.
- `src/Admin/API/Settings.php` (new file) — a deliberate 30-line no-op compatibility stub (`register_routes()` is empty); it exists only so a stale in-memory 10.8 controller list doesn't fatal on the deleted class during an update. Registers nothing.
- `src/Internal/Admin/Settings.php` — defensive `try/catch` + `class_exists` guard around `SettingsUIRequestContext::get_current()`; does not touch our `jetpackStatus` preload hunk.
- Remaining diffs (`class-wc-settings-page.php`, `class-wc-settings-payment-gateways.php`, `class-wc-email.php`, `WCAdminAssets.php`, `SettingsUIRequestContext.php`, version string, `.pot`/composer autoloads) carry no phone-home behaviour.

---

## 10.9.1 — 2026-06-24

Bump-only release. The 10.8.1 patch applied cleanly against both 10.9.0 and 10.9.1 with no rejects (line offsets only). Per-version patches were regenerated so each applies with zero fuzz/offset on a fresh extraction. The load-bearing `record_gateway_event()` fatal-fix carried forward intact.

Automattic shipped 10.9.0 and 10.9.1 back-to-back on the same day; **10.9.1 is functionally identical to 10.9.0** — the only source change is the version string (`10.9.0` → `10.9.1`) plus a no-op docblock move on the `ProductFeed` `get_entry_count()` interface method. There is effectively one release to review here, not two.

Review of the 10.8.1 → 10.9.x changes found no new patch targets active on a default install. The minor introduced several large new subsystems, but each is gated behind an experimental feature flag that defaults to off (`enabled_by_default => false`, `is_experimental => true`):
- `src/Api/` + `src/Internal/Api/` — code-first GraphQL API, gated by `dual_code_graphql_api`.
- `mcp_integration` (WooCommerce MCP) and `agentic_checkout` (Agentic Checkout API for AI agents, e.g. ChatGPT) — both off by default.
- `src/Internal/ShopperLists/` + StoreApi `ShopperList*` routes — wishlist feature gated by `product_wishlist`; `cart_save_for_later` similarly off.
- `rest_api_caching` — off by default.
- Expanded `src/Internal/PushNotifications/` (StockNotification, NotificationPreferences) — the WPCOM dispatcher only fires through an active Jetpack connection and registered mobile push tokens; our patch already disables Jetpack connection init.

Other apparent targets reviewed and cleared:
- `src/Internal/StockNotifications/Utilities/UtmHelper.php` — appends `utm_source`/`utm_medium` to the merchant's *own* back-in-stock notification email links (first-party order attribution), not a phone-home.
- `src/Internal/Admin/WCPayPromotion/WCPayPromotionDataSourcePoller.php` — pre-existing (not new in 10.9); already neutralised transitively because it short-circuits to local `DefaultPromotions` when `woocommerce_show_marketplace_suggestions` is `no`, which our options-enforcement block forces.

A patch for 10.9.0 (`patches/woocommerce-10.9.0.patch`) is also published for sites pinned to that build.

---

## 10.8.1 — 2026-06-24

Added one hunk: an early return in `record_gateway_event()`
(`includes/class-wc-payment-gateways.php`). This method, added around WC 10.7, sends a Tracks
telemetry event on every payment-gateway enable/disable and builds it with
`WC()->countries->get_base_country()`. `WC()->countries` is `null` until `WooCommerce::init()` runs
(WP `init` priority 0), so any plugin that writes a payment-gateway option *before* that point hits a
fatal `Call to a member function get_base_country() on null`. Observed in the wild on a 10.8.1 site
where WooCommerce PayPal Payments dispatches its settings migration on `init` priority -1
(example.com). The method's only terminal effect is `wc_admin_record_tracks_event()`, so the
early return both removes the phone-home (on-brand for this repo) and eliminates the crash for any
plugin that triggers it. PPCP is merely the trigger; the null-deref is WooCommerce's defect, which is
why the fix lives here rather than in PPCP.

### 10.8.1 — 2026-05-28 (initial)

Bump-only release. The 10.8.0 patch applied cleanly against 10.8.1 with no rejects.

10.8.1 is a hotfix: the only meaningful source change versus 10.8.0 is a defensive `require_once` in `includes/admin/settings/class-wc-settings-general.php` that pre-loads `src/Enums/DefaultCustomerAddress.php` to avoid a "Class not found" fatal during a same-request in-place upgrade ([woocommerce#54657](https://github.com/woocommerce/woocommerce/issues/54657)). It introduces no tracking or outbound HTTP and is not a patch target. Remaining diffs are version-string bumps (`woocommerce.php`, `class-woocommerce.php`) and translation/composer-metadata churn. No new packages or classes.

---

## 10.8.0 — 2026-05-26

Patch carried forward from 10.7.0 with line-number adjustments. The 10.7.0 patch applied cleanly against 10.8.0 with no rejects.

Review of 10.8.0 changes found no new patch targets active on a default install. Several large new subsystems were added but all are gated behind experimental feature flags that default to off:
- `src/Internal/PushNotifications/` — new WPCOM push dispatcher gated by the `push_notifications` feature flag (off by default) and an active Jetpack connection (already disabled by our patch).
- `src/Internal/Api/` + `src/Api/` — new GraphQL API gated by the `dual_code_graphql_api` feature flag (off by default).
- `src/Internal/OrderReviews/` + `includes/emails/class-wc-email-customer-review-request.php` — new customer review request feature gated by the `customer_review_request` feature flag (off by default). Local-only; no outbound HTTP.
- `src/Internal/EmailEditor/WCTransactionalEmails/` — local template housekeeping; no outbound HTTP.

`class-wc-tracker.php`, `class-wc-helper.php`, `class-wc-tracks-client.php`, `class-wc-site-tracking.php`, and the `a8c-address-autocomplete-service.js` frontend file are byte-identical to 10.7.0.

---

## 10.7.0 — 2026-04-14

Patch carried forward from 10.6.2 with minor line-number adjustments in `includes/class-woocommerce.php`.

Review of 10.7.0 changes found no new patch targets:
- New `src/Internal/PushNotifications/Dispatchers/WpcomNotificationDispatcher` (Woo Mobile app push via WPCOM) is gated on an active Jetpack connection, which our patch already disables — the feature is inert on patched installs.
- `src/Admin/Features/Fulfillments` moved from `src/Internal/Fulfillments` but contains no outbound HTTP.
- No new `DataSourcePoller`, tracks, or `wccom-site` endpoints.

---

## 10.6.2 — 2026-03-31

Patch carried forward from 10.6.1 with line-number adjustments.

---

## 10.6.1 — 2026-03-30

**New patches (identified via Claude Code Opus analysis):**
- `Experimental_Abtest::request_assignment()` — early return with empty variations to block A/B test assignment calls to `public-api.wordpress.com`
- `ShippingPartnerSuggestionsDataSourcePoller::get_data_sources()` — early return `[]` to block shipping partner suggestion fetches from `woocommerce.com`
- `MarketingRecommendationsDataSourcePoller::get_data_sources()` — early return `[]` to block marketing tab recommendation fetches
- `MiscRecommendationsDataSourcePoller::get_data_sources()` — early return `[]` to block misc marketing recommendation fetches
- `RemoteInboxNotificationsDataSourcePoller::get_data_sources()` — early return `[]` to block promotional inbox notification fetches (daily cron)
- `a8c-address-autocomplete-service.js` `createStatsdURL()` — early return `''` to block frontend tracking pixel to `pixel.wp.com/boom.gif` on checkout
- Jetpack status preload removed from `Settings.php` and `Loader.php` — eliminates wasted internal REST call to unregistered `/jetpack/v4/connection` route

**Previous 10.6.1 patches (2026-03-19):**
- `WC_Tracks_Client::init()` — early return to prevent identity cookie hooks
- `WC_WCCOM_Site::load()` — early return to disable remote product installation REST endpoints
- `add_woocommerce_tracker_send_event_wrapper()` — early return to guard against cron-based tracker initialisation

**Existing patches carried forward:**
- `WC_Site_Tracking::is_tracking_enabled()` unconditional `false`
- Marketplace suggestions and payment provider recommendations disabled
- Jetpack connection config and `Users_Connection_Admin` disabled
- Marketplace updater and promotions includes commented out
- Options enforcement block (7 options forced to `no`)
- `wcTracks` JavaScript stub
- PATCHED badge in plugin list

---

## Project changes — 2026-03-30

- Added `scripts/prepare-analysis.sh` — automated download, extraction, version detection, and patch application for new WooCommerce releases
- Added `scripts/analyse-woocommerce.md` — 11-phase analysis runbook for Claude Code Opus to systematically identify new patch targets (outbound HTTP calls, cron tasks, JS tracking, AJAX handlers, option defaults, remote logging, REST endpoints)
- Added `docs/patch-targets.md` — research notes documenting all current patch targets, future candidates, and external domains contacted by unpatched WooCommerce
- Added `CLAUDE.md` for Claude Code context
- Added `.gitignore` for `work/` directory

---

## 10.6.0 — 2026-03-05

Patch carried forward from 10.5.3 with line-number adjustments.

---

## 10.5.x (10.5.0 – 10.5.3)

- Added patch for `WC_Tracks_Client` batched pixel requests (introduced in 10.5.0) — new `send_batched_pixels()` shutdown hook identified and neutered
- Added `PaymentsProviders::get_extension_suggestions()` early return
- Added `Payments.php` payment suggestions disable

---

## 10.4.x (10.4.0 – 10.4.3)

- Added options enforcement block, force-setting tracking and feature flags to `no` on every load
- Added `wcTracks` JavaScript stub to prevent admin console errors after tracking disable
- Expanded marketplace suggestion patching to cover `class-wc-admin-marketplace-promotions.php`

---

## 10.3.x (10.3.0 – 10.3.6)

- Added patch for `src/Internal/Admin/Settings/PaymentsProviders.php` — hardcoded payment extension suggestions map
- Added Jetpack vendor patch (`class-plugin.php` — `Users_Connection_Admin` disabled)

---

## 10.2.x (10.2.1 – 10.2.2)

- Added patch for `WC_Marketplace_Suggestions::allow_suggestions()` — unconditional `false`
- Added patch for `class-wc-marketplace-updater.php` include removal

---

## 10.1.x (10.1.0 – 10.1.2)

- Added patch for `class-wc-admin-marketplace-promotions.php` include removal
- Carried forward tracking and helper patches

---

## 10.0.x (10.0.2 – 10.0.4)

- First 10.x series patches
- Structural changes in WooCommerce 10.x required rebase of several hunks
- Core patches: `WC_Site_Tracking`, `WC_Helper::get_product_usage_notice_rules()`, Jetpack connection action

---

## 9.x series (9.3.3 – 9.9.5)

Initial patch series. Core targets:
- `WC_Helper::get_product_usage_notice_rules()` — early return to stop calls to WooCommerce.com
- `WC_Site_Tracking::is_tracking_enabled()` — unconditional `false`
- Jetpack connection init action commented out
- PATCHED badge added to plugin list meta

Patches maintained across: 9.3.3, 9.4.1–9.4.3, 9.5.1–9.5.2, 9.6.0–9.6.2, 9.7.0–9.7.1, 9.8.1–9.8.5, 9.9.3–9.9.5
