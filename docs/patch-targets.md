# Patch Targets

Research notes on what we patch, why, and what we might patch in the future.

Last updated: 2026-03-30

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
| `src/Internal/Admin/Settings.php` | Jetpack status preload | Comment out | Prevents wasted internal REST call to unregistered `/jetpack/v4/connection` route |
| `src/Internal/Admin/Loader.php` | Jetpack status preload | Comment out | Same as above, in the admin loader path |

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

### Visual Indicator

| File | Target | Patch | Why |
|------|--------|-------|-----|
| `woocommerce.php` | `plugin_row_meta` filter | Appended | Adds purple "PATCHED" badge with date to plugin list for at-a-glance verification |

### Cosmetic

| File | Target | Patch | Why |
|------|--------|-------|-----|
| `src/Blocks/DependencyDetection.php` | Comment referencing webpack path | Updated | Removes reference to `client/blocks/bin/webpack-helpers.js` that doesn't exist in release builds |

---

## Future Candidates

Targets identified during analysis that are not yet patched, either because they need more investigation or the fix is non-trivial.

### Jetpack connection REST call (browser-initiated)

- **Symptom:** `GET /wp-json/jetpack/v4/connection?_locale=user` returns 404 on every WC admin page load
- **What we've done:** Commented out the PHP-side `rest_preload_api_request` calls in `Settings.php` and `Loader.php`. This eliminated the server-side preload, but a client-side JS call persists.
- **Likely source:** A bundled JS file in `vendor/automattic/jetpack-connection/dist/` (minified). The dist files `jetpack-connection.js` and `identity-crisis.js` both reference the endpoint.
- **Why not patched yet:** Patching minified JS bundles is fragile and version-sensitive. Needs further investigation to find a cleaner server-side intercept.
- **Impact:** One wasted REST round-trip per admin page load (404 response). Low priority but not zero cost.

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
