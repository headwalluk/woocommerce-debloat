# HAND-OFF: fatal `get_base_country() on null` in `WC_Payment_Gateways::record_gateway_event()`

**Created:** 2026-06-24 · **From:** woocommerce-paypal-payments-fixes session · **Owner repo for the fix:** this one (woocommerce-debloat)
**Disposition:** add a new hunk to the WooCommerce patch (root-cause fix lives in WC, not PPCP). Delete this file once integrated.

---

## TL;DR

WooCommerce ≥ 10.7 added gateway enable/disable **Tracks telemetry** in
`includes/class-wc-payment-gateways.php::record_gateway_event()`. That method does:

```php
'business_country' => WC()->countries->get_base_country(),
```

`WC()->countries` is `null` until `WooCommerce::init()` runs (WP `init`, **priority 0**). If *any* code
writes a payment-gateway option **before** that point, the `update_option_<key>` /`add_option_<key>`
listener fires → `record_gateway_event()` → **fatal: `Call to a member function get_base_country() on null`**.

**Fix:** early-return at the top of `record_gateway_event()`. It's telemetry only (terminates in
`wc_admin_record_tracks_event()`), so disabling it is on-brand for this repo *and* removes the crash.

---

## Where it surfaced

- Site: `example.com` (host hhw7). WooCommerce **10.8.1**, WooCommerce PayPal Payments (PPCP) **4.1.0** (patched).
- Log (`/var/www/example.com/log/error.log`), two near-simultaneous front-end requests:

```
PHP Fatal error: Uncaught Error: Call to a member function get_base_country() on null
in .../woocommerce/includes/class-wc-payment-gateways.php:446
#0 class-wc-payment-gateways.php(202): WC_Payment_Gateways->record_gateway_event()
#1 class-wc-payment-gateways.php(156): WC_Payment_Gateways->payment_gateway_settings_option_changed()
#2-5 WP hooks / option.php(927): add_option()
#6 .../woocommerce-paypal-payments/modules/ppcp-settings/src/Data/PaymentSettings.php(48): update_option()
#7 .../woocommerce-paypal-payments/modules/ppcp-settings/src/SettingsModule.php(562): {closure}
```

## Root cause (full chain)

1. PPCP's **main plugin file** dispatches its settings migration on `init` **priority -1**:
   ```php
   add_action('init', function () {
       ...
       update_option('woocommerce-ppcp-version', $current);
       do_action('woocommerce_paypal_payments_gateway_migrate', $installed_version);   // init, prio -1
   }, -1);
   ```
2. WooCommerce instantiates `WC()->countries` inside `WooCommerce::init()`, hooked on `init` **priority 0**
   (`includes/class-woocommerce.php`: `public $countries = null;` → `$this->countries = new WC_Countries();`).
   **prio -1 runs before prio 0 → `WC()->countries` is still `null`.**
3. Several `gateway_migrate` subscribers in PPCP `SettingsModule.php` (the retroactive APM / card-button
   fixes) call `PaymentSettings::save()` → `update_option()` on a **WC gateway option key**, synchronously,
   during that prio -1 dispatch. (PPCP guards its *main* migrate subscriber with `did_action('woocommerce_init')`
   but **not** these retroactive ones — that's a PPCP bug, reported separately, not our concern here.)
4. The gateway option write fires WC's `payment_gateway_settings_option_changed()` →
   `record_gateway_event()` → builds `business_country => WC()->countries->get_base_country()` → **null deref**.

So PPCP is the *trigger*, but the *defect* is WC dereferencing a possibly-null `WC()->countries`. Any plugin
that writes a gateway option before `init` priority 0 hits this. Fixing it at the WC root protects the whole fleet.

## The fix

Target: `includes/class-wc-payment-gateways.php`, function `record_gateway_event()` (starts at line **423** in 10.8.1).

Add an early return immediately after the opening brace — same pattern this repo already uses for
`get_product_usage_notice_rules()`, `allow_suggestions()`, etc.

### Ready-to-apply hunk (append to `patches/woocommerce-10.8.1.patch`)

> Regenerate via the normal workflow rather than hand-pasting if line numbers have drifted — see below.
> The `+++` timestamp should be whatever your `diff -ruN` produces; the content is what matters.

```diff
diff -ruN woocommerce-10.8.1/includes/class-wc-payment-gateways.php woocommerce-10.8.1-patched/includes/class-wc-payment-gateways.php
--- woocommerce-10.8.1/includes/class-wc-payment-gateways.php	2026-05-27 17:54:04.000000000 +0100
+++ woocommerce-10.8.1-patched/includes/class-wc-payment-gateways.php	2026-06-24 00:00:00.000000000 +0100
@@ -423,6 +423,13 @@
 	private function record_gateway_event( string $name, $gateway ) {
+		// wpatcher: disable gateway enable/disable Tracks telemetry AND fix a fatal.
+		// This method derefs WC()->countries->get_base_country(), but WC()->countries is null
+		// until WooCommerce::init() (WP `init` priority 0). Any plugin that writes a payment-gateway
+		// option before that point reaches here with a null WC()->countries -> "Call to a member
+		// function get_base_country() on null". (Observed trigger: PPCP fires its settings migration
+		// on `init` priority -1.) The method only ends in wc_admin_record_tracks_event(), so a plain
+		// early return both kills the crash and suppresses the phone-home.
+		return;
 		if ( ! function_exists( 'wc_admin_record_tracks_event' ) ) {
 			return;
 		}
```

### Why early-return, not a null-guard

A minimal alternative is `WC()->countries ? WC()->countries->get_base_country() : ''`. It works, but:
- `record_gateway_event()` is purely a `wc_admin_record_tracks_event()` (Automattic Tracks) sink — exactly the
  kind of telemetry this repo strips. Early-return removes it wholesale, consistent with the repo's posture.
- It returns `void` and has no functional side effects on payments — safe to disable entirely.

## Integration steps (this repo's workflow)

1. `./scripts/prepare-analysis.sh` (or reuse `work/woocommerce-10.8.1-patched/` if present).
2. Edit `work/woocommerce-10.8.1-patched/includes/class-wc-payment-gateways.php` — insert the early return above.
3. `php -l` the file.
4. Regenerate: `diff -ruN work/woocommerce-10.8.1 work/woocommerce-10.8.1-patched > work/woocommerce-10.8.1-candidate.patch`,
   confirm the new hunk is the only addition, then fold it into `patches/woocommerce-10.8.1.patch`.
5. Dry-run against a pristine 10.8.1 extraction: `patch -p1 --directory=woocommerce --dry-run < patches/woocommerce-10.8.1.patch`.
6. Bump the `// START : wpatcher` badge **date** in the patch to today.
7. Update `CHANGELOG.md` and `docs/` patch-target notes.
8. Sync the updated patch to the WPatcher tree (the deploy host's `wpatches/plugins/woocommerce/` dir), so
   `wo-update.sh` re-applies it after WooCommerce updates.

## Carry-forward

- `record_gateway_event()` was introduced around **WC 10.7.0** (`payment_gateway_settings_option_changed` is
  `@since 8.5.0`, but the `business_country => WC()->countries->...` tracking is the newer addition). Add the same
  hunk to **every WC ≥ 10.7 patch you maintain** (10.7.0, 10.8.0, 10.8.1, and forward). Verify line numbers per version.
- If any fleet sites still run a 10.7.x/10.8.0 patch, back-port the hunk there too — they're vulnerable.

## Verification

- After patching, the fatal stops (grep the site's `error.log` for `get_base_country` — should be clean).
- Sanity: enabling/disabling a payment gateway in WC admin still saves correctly (it will, the method is telemetry-only).
- Note: on `example.com`, the PPCP retroactive APM migration **crashed mid-way**, so that store's local-APM gateway
  state may be left incomplete — worth a manual check of its PayPal payment-method settings once the fatal is gone.

## Cross-reference

- Trigger-side detail lives in the **woocommerce-paypal-payments-fixes** repo notes. We deliberately chose to fix
  at the WC root here rather than patch PPCP's dispatch priority, because this repo already owns WooCommerce patches
  and the null-deref is genuinely WC's bug.
