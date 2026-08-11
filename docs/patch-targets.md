# Patch Targets

Research notes on what we patch, why, and what we might patch in the future.

Last updated: 2026-08-11

---

## Currently Patched

### Tracking & Telemetry

| File | Target | Patch | Why |
|------|--------|-------|-----|
| `includes/tracks/class-wc-site-tracking.php` | `is_tracking_enabled()` | Early return `false` | Master switch — disables all WC tracking regardless of admin setting |
| `includes/tracks/class-wc-tracks-client.php` | `init()` | Early return | Prevents `tk_ai` identity cookie being set on admin sessions |
| `includes/class-woocommerce.php` | `WC_Site_Tracking::init` hook | Comment out | Stops tracking scripts from loading in admin/REST contexts |
| `includes/class-woocommerce.php` | `add_woocommerce_tracker_send_event_wrapper()` | Early return | Prevents cron-scheduled telemetry send to `tracking.woocommerce.com/v1/` |
| `includes/class-woocommerce.php` | `wcTracks` JS stub | Injected in `admin_footer` | Prevents console errors from scripts that reference `window.wcTracks` |
| `includes/react-admin/class-experimental-abtest.php` | `request_assignment()` | Early return empty variations | Blocks A/B test assignment calls to `public-api.wordpress.com` |
| `assets/js/frontend/a8c-address-autocomplete-service.js` | `createStatsdURL()` | Early return `''` | Blocks frontend tracking pixel to `pixel.wp.com/boom.gif` on checkout |
| `includes/class-wc-payment-gateways.php` | `record_gateway_event()` | Early return | Suppresses gateway enable/disable Tracks event **and** fixes a fatal `get_base_country() on null` when a plugin writes a gateway option before `init` priority 0 (e.g. PPCP migration at prio -1) |

### Marketplace & Upsells

| File | Target | Patch | Why |
|------|--------|-------|-----|
| `includes/admin/marketplace-suggestions/class-wc-marketplace-suggestions.php` | `allow_suggestions()` | Early return `false` | Disables marketplace suggestions system |
| `includes/class-woocommerce.php` | Marketplace updater include | Comment out | Prevents `class-wc-marketplace-updater.php` from loading (fetches from `woocommerce.com`) |
| `includes/class-woocommerce.php` | Marketplace promotions include | Comment out | Prevents `class-wc-admin-marketplace-promotions.php` from loading |
| `src/Admin/Features/PaymentGatewaySuggestions/PaymentGatewaySuggestionsDataSourcePoller.php` | `get_data_sources()` | Early return `[]` | Blocks payment gateway suggestion fetches from `woocommerce.com` |
| `src/Internal/Admin/RemoteFreeExtensions/RemoteFreeExtensionsDataSourcePoller.php` | `get_data_sources()` | Early return `[]` | Blocks free extension suggestion fetches from `woocommerce.com` |
| `src/Admin/Features/ShippingPartnerSuggestions/ShippingPartnerSuggestionsDataSourcePoller.php` | `get_data_sources()` | Early return `[]` | Blocks shipping partner suggestion fetches from `woocommerce.com` |
| `src/Admin/Features/MarketingRecommendations/MarketingRecommendationsDataSourcePoller.php` | `get_data_sources()` | Early return `[]` | Blocks marketing tab recommendation fetches from `woocommerce.com` |
| `src/Admin/Features/MarketingRecommendations/MiscRecommendationsDataSourcePoller.php` | `get_data_sources()` | Early return `[]` | Blocks misc marketing recommendation fetches from `woocommerce.com` |
| `src/Admin/RemoteInboxNotifications/RemoteInboxNotificationsDataSourcePoller.php` | `get_data_sources()` | Early return `[]` | Blocks promotional inbox notification fetches (daily cron) from `woocommerce.com` |
| `src/Internal/Admin/Settings/Payments.php` | `get_extension_suggestions` call | Comment out | Prevents extension suggestions appearing on Payments settings page |
| `src/Internal/Admin/Settings/PaymentsProviders.php` | `get_extension_suggestions()` | Early return empty structure | Prevents extension suggestion lookups |
| `src/Internal/Admin/WCAdminAssets.php` | `enqueue_assets()` | Inline style + inline script injected | Suppresses the two Marketplace upsell surfaces on the Extensions screen: `.woocommerce-marketplace__banner` (4-slide woocommerce.com carousel, top) and `.woocommerce-marketplace__footer` (same 4 messages, bottom). Both are hardcoded in the compiled `assets/client/admin/app/index.js` with no PHP hook or filter to unhook, and the bundle is a single 248 KB minified line so it cannot be diffed. The script seeds `localStorage.wc_featuredBannerDismissed = 'true'` — the carousel's own dismiss flag — so it renders `null` and emits no DOM; the CSS only covers the flash before that effect runs, and is the sole surviving exception to the no-CSS-masking rule. Note the flag outlives the patch: clear that localStorage key to restore the carousel. **Both inline snippets must keep their leading `;`** — see the drift watchlist |

### WooCommerce.com Remote Access

| File | Target | Patch | Why |
|------|--------|-------|-----|
| `includes/wccom-site/class-wc-wccom-site.php` | `load()` | Early return | Disables REST endpoints that allow Automattic to remotely install plugins, and removes `determine_current_user` filter hook |
| `includes/admin/helper/class-wc-helper.php` | `get_product_usage_notice_rules()` | Early return `[]` | Prevents outbound call to fetch product usage notice rules |

### Jetpack Connection

| File | Target | Patch | Why |
|------|--------|-------|-----|
| `includes/class-woocommerce.php` | `init_jetpack_connection_config` hook | Comment out | Prevents Jetpack connection configuration from initialising |
| `vendor/automattic/jetpack-connection/src/class-plugin.php` | `Users_Connection_Admin` instantiation | Comment out | Prevents Jetpack user connection admin UI from loading |
| `src/Internal/Admin/WCAdminAssets.php` | `enqueue_assets()` | `wp-api-fetch` middleware injected | Short-circuits the wc-admin `isJetpackConnected` resolver's `apiFetch( '/jetpack/v4/connection' )` in the browser, returning a static "not connected" object. The route is unregistered by the hunks above, so the call would otherwise 404 on every wc-admin page load and boot the full WP stack to answer it. Only that exact path is intercepted; everything else falls through to `next()` |

**Retired in WC 11.0.0:** `src/Internal/Admin/Settings.php` and `src/Internal/Admin/Loader.php` each carried a hunk commenting out the `$preload_data_endpoints['jetpackStatus'] = '/jetpack/v4/connection'` server-side preload. 11.0.0 removed that preload upstream in favour of a direct in-process `REST_Connector::connection_status( false )` call, which reads local options only. The target no longer exists, so both hunks were dropped from the 11.0.0 patch. They remain in the ≤10.9.4 patches.

### Feature Rollouts & Cohorts

WooCommerce stages some feature and recommendation rollouts to a percentage of stores using a locally-assigned random bucket, with no merchant action or notice.

| File | Target | Patch | Why |
|------|--------|-------|-----|
| `src/Internal/VariationGallery/Package.php` | `is_in_canary_cohort()` | Early return `false` | WC 11.0.0 made the `variation_gallery` feature's `enabled_by_default` a call to this method, silently switching the product gallery UI on for buckets 1–6 of 120 (~5% of stores) on upgrade. Patched here rather than in `is_enabled()` so an explicit `yes` on `wc_feature_woocommerce_additional_variation_images_enabled` still wins — merchant opt-in is preserved, only auto-enrolment is removed |
| `includes/class-woocommerce.php` | `woocommerce_remote_variant_assignment` option | Force to `0` | See Options Enforcement below |
| `src/Admin/API/Options.php` | `get_options()` | Blank `woocommerce_remote_variant_assignment` in the response | Stops the wc-admin client ever seeing the rollout bucket. Added in the 11.0.1 revision to kill the order attribution install banner on Analytics → Overview (`.woocommerce-order-attribution-install-banner`, "Discover what drives your sales" / **Try it now**), an upsell for the `woocommerce-analytics` plugin that also fires `order_attribution_install_banner_viewed` / `_clicked` / `_dismissed` Tracks events. **Our own `0` is what switched it on** — see the trap below. Returning `false` reaches the client as `NaN` through `parseInt()`, which fails an upper-bound and a lower-bound test alike, so no present or future client-side cohort check can match. The stored value is untouched, so `is_in_canary_cohort()` and the Remote Spec rules still see `0`. Blanked inside the loop rather than dropped from `$legacy_whitelisted_options`, because `get_item_permissions_check()` rejects the **whole** request if any single requested option is unpermitted, which would 403 every batch that includes it |

**The cohort-sentinel trap — read this before adding any similar hunk.** Every server-side `range` rule is bounded at both ends (`ComparisonOperation.php`: `$left >= $right[0] && $left <= $right[1]`), which is why `0` is the correct sentinel for PHP. The wc-admin client is not so careful. The order attribution banner's check is `parseInt( value ) <= threshold`, **upper bound only**, threshold 12 by default, so `0` read as in-cohort and the banner appeared on patched stores that would otherwise have seen it 10% of the time. **A sentinel chosen to sit below every server-side range will pass any client-side check that tests only an upper bound.** When auditing a cohort-gated surface, check both bound directions, and prefer making the value non-numeric at the client boundary over picking a different number — a different number only moves the problem to the other bound.

### Options Enforcement

Forced to `'no'` on every load in `includes/class-woocommerce.php` `init_hooks()`:

| Option | Why |
|--------|-----|
| `woocommerce_allow_tracking` | Master tracking toggle — prevents silent re-enablement by updates |
| `marketplace_suggestions` | Marketplace suggestions toggle |
| `woocommerce_show_marketplace_suggestions` | UI toggle for marketplace suggestions |
| `woocommerce_feature_remote_logging_enabled` | Sends error logs to `public-api.wordpress.com/rest/v1.1/logstash` |
| `woocommerce_feature_blueprint_enabled` | Bulk import/export — attack surface with no clear benefit for most stores |
| `woocommerce_feature_point_of_sale_enabled` | POS feature — unnecessary overhead for most stores |
| `woocommerce_feature_reactify-classic-payments-settings_enabled` | New payments settings UI — forced on by update functions |

Forced to `0` in the same block (separate from the loop above, whose `FILTER_VALIDATE_BOOLEAN` guard would read a value like `"42"` as false and skip it):

| Option | Why |
|--------|-----|
| `woocommerce_remote_variant_assignment` | Sticky random 1–120 bucket assigned by `add_woocommerce_remote_variant()` (hooked to `woocommerce_installed` **and** `woocommerce_updated`). Remote Spec Engines test it with a numeric `range` rule to target a percentage of stores. Despite the name the value is *not* fetched remotely — it is a local `wp_rand( 1, 120 )`; what is remote is the spec that tests it. `0` sits below every range core defines, so the store matches no cohort: the variation gallery canary (1–6), the MailPoet/Klaviyo recommendation split (1–84 / 85–120) and the TikTok/Pinterest split (1–60 / 61–120) all stop matching. The guard writes when the option is **absent** as well as non-zero — `add_woocommerce_remote_variant()` only rolls when `get_option()` returns `false`, so storing `'0'` pre-empts the roll instead of letting every update re-roll it |

### Visual Indicator

| File | Target | Patch | Why |
|------|--------|-------|-----|
| `woocommerce.php` | `plugin_row_meta` filter | Appended | Adds purple "PATCHED" badge with date to plugin list for at-a-glance verification |

### Cosmetic

| File | Target | Patch | Why |
|------|--------|-------|-----|
| `src/Blocks/DependencyDetection.php` | Comment referencing webpack path | Updated | Removes reference to `client/blocks/bin/webpack-helpers.js` that doesn't exist in release builds |

---

## Drift Watchlist

A clean apply proves a hunk still *fits*. It does not prove the hunk still *does* anything. These are the targets that can go silently dead on a WooCommerce upgrade, with the anchor to check and the baseline count as of clean 11.0.1. Nothing in `patch`, `php -l` or the apply log will flag any of them.

Run from the **clean** extraction, not the patched one.

| Anchor | Check | 11.0.1 baseline | Fails silently if |
|--------|-------|-----------------|-------------------|
| `.woocommerce-marketplace__banner` | `grep -o` in `assets/client/admin/app/index.js` | 3 (plus 9 in `app/style.css`) | Class renamed in a rebuilt bundle. CSS hides nothing, carousel returns |
| `.woocommerce-marketplace__footer` | same | 4 (plus 7 in `app/style.css`) | As above |
| `wc_featuredBannerDismissed` | same | 2 | Dismissal flag renamed or moved off `localStorage`. Banner then relies on the CSS alone, so it still emits DOM |
| `isJetpackConnected` | same | 3 | Resolver renamed. **Grepping for the literal `jetpack/v4/connection` is a false negative** — the resolver builds the path from a `"/jetpack/v4"` constant, so the literal never appears in the bundle |
| `woocommerce_remote_variant_assignment` | `grep -o` in `app/index.js` **and** `embed/index.js` | 1 + 1 | A second client-side consumer appears (count rises), or the banner moves to a different gate (count drops to 0, making the `Options.php` hunk dead weight) |
| `order_attribution_install_banner_dismissed` | same | 2 + 2 | Banner reworked. Re-check which bound its cohort test uses |
| `case 'range'` in `RemoteSpecs/RuleProcessors/ComparisonOperation.php` | Confirm it is still `>= $right[0] && <= $right[1]` | both-bounded | Upstream adds a one-sided operator. `0` would then start *matching* cohorts instead of avoiding them |
| `record_gateway_event()` in `includes/class-wc-payment-gateways.php` | Method still exists, signature unchanged | 1 | **Load-bearing.** This one also guards a fatal, so losing it is worse than losing a telemetry strip |
| `createStatsdURL` in `assets/js/frontend/a8c-address-autocomplete-service.js` | `grep -c` | 2 | File rebuilt or renamed. It is a shipped JS asset, so it churns more than the PHP |
| `is_in_canary_cohort()` in `src/Internal/VariationGallery/Package.php` | Method still exists and is still referenced by `enabled_by_default` | 2 | Cohort logic inlined or moved. Auto-enrolment returns |
| Forced feature options | Each slug still in `FeaturesController::get_feature_definitions()` **without** an `option_key` override | `remote_logging`, `blueprint`, `point_of_sale` all present, none overridden | A feature gaining an `option_key` orphans our forced `woocommerce_feature_<slug>_enabled` write. Note the literal option name does **not** appear in source — it is built by `sprintf( 'woocommerce_feature_%s_enabled', $slug )` at `FeaturesController.php:706`, so grepping for the full option name returns 0 hits and proves nothing |
| Leading `;` on both `WCAdminAssets.php` inline scripts | Read the hunk | present | Not drift, but the highest-consequence regression in the set. `Features.php` emits `window.wcAdminFeatures = {…}` with no trailing semicolon, so ASI does not break the statement and the next `(` parses as a call on the object. This has taken the whole Extensions screen down once |

## Future Candidates

Targets identified during analysis that are not yet patched, either because they need more investigation or the fix is non-trivial.

### ~~Jetpack connection REST call (browser-initiated)~~ — RESOLVED in 10.9.4

Kept for the correction. The original note guessed the caller was one of the minified bundles in `vendor/automattic/jetpack-connection/dist/` and concluded patching it would be too fragile. That diagnosis was **wrong**: the caller is wc-admin's own `isJetpackConnected` resolver, which builds the path from a `"/jetpack/v4"` constant (so the literal string never appears in the bundle, which is why grepping for it found only the vendor dist files). The fix was a `wp-api-fetch` middleware rather than any JS patching — see the `WCAdminAssets.php` row under Jetpack Connection above. Re-verified against 11.0.0: the resolver is unchanged and the interceptor is still required.

**Remaining gap:** the `getJetpackConnectionData` resolver fetches `/jetpack/v4/connection/data`, which the interceptor's `connection$` anchor deliberately does not match. Pre-existing, not a regression, and only fires on surfaces that select that data. Widen the regex if it shows up in access logs.

### WC_Admin_Addons (Addons page fetches)

- **File:** `includes/admin/class-wc-admin-addons.php`
- **What it does:** Multiple methods (`fetch_featured`, `get_sections`, `fetch_product_preview`) make `wp_safe_remote_get` calls to `woocommerce.com` to populate the Extensions/Addons admin page.
- **Why not patched yet:** These only fire when a user visits the Addons page. Since the page is specifically for browsing extensions, disabling the fetches would leave a broken/empty page. Consider patching only if the page is loading data proactively (e.g. via cron or preload) without the user visiting it.

### Marketing Knowledge Base

- **File:** `src/Internal/Admin/Marketing/MarketingSpecs.php`
- **What it does:** Fetches marketing articles from `woocommerce.com/wp-json/wccom/marketing-knowledgebase/v1/posts/` when the Marketing tab is viewed.
- **Why not patched yet:** Only fires when visiting the Marketing tab. Similar reasoning to Addons page — the tab exists to show this content.

### Jetpack Tracks AJAX handler

- **File:** `vendor/automattic/jetpack-connection/src/class-tracking.php`
- **What it does:** Registers `wp_ajax_jetpack_tracks` which sends events to `pixel.wp.com/t.gif`.
- **Why not patched yet:** With `WC_Site_Tracking` disabled and the `wcTracks` stub in place, this handler likely never fires. Needs confirmation via server-side logging before patching.

### Jetpack Error Handler

- **File:** `vendor/automattic/jetpack-connection/src/class-error-handler.php`
- **What it does:** Sends encrypted error reports to `public-api.wordpress.com/wpcom/v2/sites/{blog_id}/jetpack-report-error/`.
- **Why not patched yet:** Only fires on Jetpack connection errors. With Jetpack connection disabled, this should be inert. Low priority.

### Onboarding Products

- **File:** `src/Internal/Admin/Onboarding/OnboardingProducts.php`
- **What it does:** Fetches product data from `woocommerce.com/wp-json/wccom-extensions/1.0/search` during onboarding wizard. Cached daily.
- **Why not patched yet:** Only relevant during initial store setup. Most production stores have completed onboarding.

---

## Domains Contacted by Unpatched WooCommerce

For reference, these are the external domains that WooCommerce contacts:

| Domain | Purpose |
|--------|---------|
| `tracking.woocommerce.com` | Store telemetry (weekly cron) |
| `pixel.wp.com` | Event tracking pixels (admin actions, checkout autocomplete) |
| `stats.wp.com` | Tracking JavaScript library |
| `woocommerce.com` | Marketplace suggestions, extension recommendations, inbox notifications, promotions, onboarding |
| `public-api.wordpress.com` | A/B test assignments, remote logging, Jetpack error reports |
| `download.maxmind.com` | GeoIP database updates (requires licence key — legitimate functionality) |
