# Changelog

All notable changes to the patch set are documented here, grouped by WooCommerce version.

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
