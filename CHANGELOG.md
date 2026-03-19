# Changelog

All notable changes to the patch set are documented here, grouped by WooCommerce version.

---

## 10.6.1 — 2026-03-19

**New patches:**
- `WC_Tracks_Client::init()` — added early return to prevent identity cookie hooks being registered on `admin_init`, even when tracking is disabled
- `WC_WCCOM_Site::load()` — added early return to disable the remote product installation REST endpoints and remove the `determine_current_user` filter that ran on every request
- `add_woocommerce_tracker_send_event_wrapper()` — added early return as belt-and-braces guard against `WC_Tracker` initialisation via cron

**Existing patches carried forward:**
- `WC_Site_Tracking::is_tracking_enabled()` unconditional `false`
- Marketplace suggestions and payment provider recommendations disabled
- Jetpack connection config and `Users_Connection_Admin` disabled
- Marketplace updater and promotions includes commented out
- Options enforcement block (7 options forced to `no`)
- `wcTracks` JavaScript stub
- PATCHED badge in plugin list

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
