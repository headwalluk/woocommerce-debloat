# WooCommerce Debloat

**Performance and privacy patches for WooCommerce.**

WooCommerce ships with a substantial amount of code that serves Automattic's interests rather than yours: admin-area tracking that runs even when usage tracking is disabled, outbound HTTP calls to WooCommerce.com on every page load, a remote product installation endpoint that lets Automattic push plugins to your site, and marketplace recommendation systems that prioritise their own commercial partners.

This project provides surgical patches to remove that behaviour — without breaking anything you actually need. Plugin update checking, subscription management, and all core ecommerce functionality remain fully intact.

**Important**: This is NOT an attempt to nullify commercial plugins or extensions. I created this patch to improve my clients' WooCommerce admin area performance and to stop unnecessary data being sent to third parties without meaningful consent. If you're looking for something that lets you use commercial WooCommerce extensions without paying, you're looking in the wrong place.

---

## What the patch does

Each patch targets a specific version of WooCommerce. The changes are conservative and commented. Here's what they cover:

**Tracking & telemetry**
- Disables `WC_Site_Tracking` unconditionally, regardless of the admin setting
- Neuters `WC_Tracks_Client::init()` to prevent identity cookies being set on admin sessions
- Stops `WC_Tracker` from initialising via the cron wrapper, even if the `woocommerce_allow_tracking` option is somehow re-enabled
- Injects a `wcTracks` JavaScript stub so the admin UI doesn't generate console errors

**Marketplace & upsells**
- Disables marketplace suggestions and payment provider recommendations
- Removes payment extension suggestions from the Payments settings page
- Comments out the marketplace updater and promotions includes

**WooCommerce.com remote access**
- Disables `WC_WCCOM_Site::load()`, which registers REST endpoints that allow Automattic to remotely install plugins on your site and hooks into `determine_current_user` on every request
- Disables the Jetpack `Users_Connection_Admin` initialisation

**Options enforcement**
- Force-sets the following options to `no` on every load, preventing them from being silently re-enabled by an update:
  - `woocommerce_allow_tracking`
  - `marketplace_suggestions`
  - `woocommerce_show_marketplace_suggestions`
  - `woocommerce_feature_remote_logging_enabled`
  - `woocommerce_feature_blueprint_enabled`
  - `woocommerce_feature_point_of_sale_enabled`

**What the patch deliberately leaves alone**
- WooCommerce.com subscription checking and licence validation
- Plugin update data fetching (via `WC_Helper_Updater`)
- All core ecommerce functionality: orders, products, payments, shipping, checkout
- The WooCommerce REST API

---

## Compatibility

Patches are provided per WooCommerce version. Check the `patches/` directory for your version.

| WooCommerce | Patch file |
|---|---|
| 10.6.1 | `patches/woocommerce-10.6.1.patch` |
| 10.6.0 | `patches/woocommerce-10.6.0.patch` |
| 10.5.x | `patches/woocommerce-10.5.x.patch` |
| Older versions | See `patches/` directory |

Patches are maintained on a best-effort basis, typically updated within a few days of a new WooCommerce release.

---

## How to apply

You'll need the original WooCommerce plugin files and `patch` available on your system.

```bash
# From your WordPress plugins directory
cd /path/to/wp-content/plugins

# Apply the patch
patch -p1 --directory=woocommerce < /path/to/woocommerce-10.6.1.patch

# Verify no rejects
ls woocommerce/**/*.rej 2>/dev/null && echo "Check rejects!" || echo "Clean apply"
```

> **Note:** Always test on a staging environment before applying to production. After applying, check your error log and verify the WooCommerce admin loads correctly.

### Using a deployment workflow

If you manage WooCommerce updates via Composer, Git, or a custom deployment script, the patch can be applied as a post-install step. The `patches/` directory structure makes it straightforward to select the correct patch per version.

---

## After applying

You should see a **PATCHED** badge on the WooCommerce entry in your plugins list (`wp-admin/plugins.php`), showing the date the patch was applied.

To verify the patch is working as expected, you can check that:
- `get_option('woocommerce_allow_tracking')` returns `no`
- `WC_WCCOM_Site::authenticate_wccom` is not registered on the `determine_current_user` filter
- No outbound requests to `tracking.woocommerce.com` or `pixel.wp.com` appear in your server logs

---

## Background

This project started from a real incident: a WooCommerce site with moderate traffic and multiple staff using the admin area simultaneously was grinding to a halt. PHP-FPM workers were exhausted, and frontend response times had climbed to 1–5 seconds. The culprit wasn't server capacity — it was WooCommerce making outbound HTTP calls to Automattic's servers on every admin page load and AJAX heartbeat.

Even with usage tracking disabled in the WooCommerce settings, tracking infrastructure continued to load. The admin area telemetry, the marketplace recommendation fetcher, and the remote site connector all ran regardless of the toggle in the UI.

The patches in this repository are the result of working through the WooCommerce source to identify and remove that behaviour, while preserving everything that legitimate ecommerce operations depend on.

A detailed write-up of the investigation is available at [headwall-hosting.com](https://headwall-hosting.com/a-web-guys-blog/) *(coming soon)*.

---

## Contributing

Found something that should be patched? Open an issue with the file, line number, and what it does. PRs welcome — please include the WooCommerce version and a brief explanation of what the change removes and why it's safe.

---

## Licence

GPL v2 — the same licence as WooCommerce itself.
