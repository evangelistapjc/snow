# Copilot Instructions for `snow` (WordPress Plugin)

> Purpose: Guide AI pair‑programmers (e.g., GitHub Copilot, ChatGPT) to generate safe, maintainable, production‑ready code for this repository.

## Project Snapshot

* **Name:** Snow — modular experimentation & analytics for WordPress
* **Language:** PHP (7.4+), JavaScript (ES2019+), CSS (Tailwind where applicable)
* **Minimum WP:** 5.0 (tested up to 6.4+)
* **Structure**

  ```text
  /src
    /includes
      class-snow-core.php
    snow.php
  /assets
  /docs
  /tests
  README.md
  LICENSE
  CONTRIBUTING.md
  ```
* **Entry point:** `src/snow.php` loads `includes/class-snow-core.php` and other modules.

## Guardrails (Generate Nothing That Violates These)

1. **Security first**

   * Always validate, sanitize, and escape:

     * `$_GET/$_POST/$_REQUEST/$_COOKIE` → use `sanitize_text_field`, `intval`, `sanitize_email`, etc.
     * Escape output with `esc_html`, `esc_attr`, `esc_url`, `wp_kses_post`.
   * **Nonces**: All AJAX/POST actions must check a nonce via `check_ajax_referer` / `wp_verify_nonce`.
   * **Capabilities**: Gate admin actions with `current_user_can('manage_options')` unless a narrower cap is appropriate.
   * **DB access**: Use `$wpdb->prepare()`; never concatenate unsanitized input.
   * **Encryption/PII**: If storing PII, leverage `openssl_*` with keys from `get_option(...)`, not hardcoded secrets.
   * **No eval, no dynamic `include`/`require` from user input, no remote code.**

2. **Compliance & Privacy**

   * Provide opt‑in consent hooks for tracking/experiments.
   * Respect `do_not_track`/consent state in all front‑end scripts.
   * Log only necessary data; allow admins to export/delete user data.

3. **Performance**

   * Avoid blocking calls on `init`/front‑end; defer heavy work to `admin_init`, cron, or REST endpoints.
   * Use script/style handles and dependencies; enqueue only on relevant admin pages.

4. **Internationalization (i18n)**

   * Wrap strings with `__()`, `_e()`, `_x()`, etc., text domain: `snow`.

5. **Compatibility**

   * Namespaces/classes prefixed `Snow_` to avoid collisions.
   * Do not rely on non‑core PHP extensions.

## Coding Standards

* Follow **WordPress Coding Standards** (PHPCS): spacing, naming, files, and hooks.
* PHPDoc for public methods; function phpDocs must include `@param`, `@return`, `@since` when relevant.
* Keep functions under \~50 lines; refactor into private helpers where needed.

## File & Module Guidelines

* **Core lifecycle** lives in `class-snow-core.php`:

  * Singleton `Snow_Core::init()` bootstraps modules.
  * Activation/Deactivation/Uninstall static methods handle setup/teardown.
* **New features** → add a class in `/src/includes/` (e.g., `class-snow-analytics.php`) and hook in via the core.
* **REST API** routes belong under a module class with `register_routes()` and capability checks.
* **Assets**: enqueue with version `SNOW_VERSION`; localize data via `wp_localize_script`.

## Commits, Branches, and PRs

* **Branch naming**: `feat/<scope>`, `fix/<scope>`, `docs/<scope>`, `chore/<scope>`
* **Conventional commits** preferred (e.g., `feat(experiments): add variant assignment via cookies`).
* **PR checklist** (Copilot must help author fill):

  * [ ] Security: input sanitized, output escaped, nonce/caps checked
  * [ ] Performance: no unnecessary queries; enqueue scope reduced
  * [ ] Tests: unit or integration added/updated when feasible
  * [ ] Docs: README/inline docs updated; user‑visible changes documented
  * [ ] i18n: strings wrapped; new POT generated (if applicable)

## Preferred Patterns

* **Actions & Filters**: Expose extension points. Example:

  ```php
  /**
   * Filter experiment assignment payload before persist.
   * @param array $payload
   */
  $payload = apply_filters( 'snow_experiment_payload', $payload );
  ```
* **AJAX**: Use both `wp_ajax_` and `wp_ajax_nopriv_` when needed; always check nonce and rate‑limit sensitive routes.
* **Settings**: Use `register_setting`, `add_settings_section`, `add_settings_field`; autoload only tiny options.
* **Database**: Tables use `$wpdb->prefix . 'snow_*'`; migrations versioned via `SNOW_DB_VERSION`.

## Anti‑Patterns (Copilot should refuse/suggest fixes)

* Writing unescaped HTML to the admin page.
* Direct use of `$_REQUEST` without sanitization.
* Storing API keys/plain PII unencrypted.
* Blocking remote HTTP calls on page render; prefer WP Cron or async queues.
* Global functions for new features; always prefer classes under `Snow_*`.

## Example Stubs Copilot Can Expand

### Minimal Module Class

```php
<?php
class Snow_Example_Module {
	public function __construct() {
		add_action( 'init', [ $this, 'init' ] );
	}

	public function init() : void {
		// Register CPT/Taxonomy or hooks.
	}
}
```

### Secure AJAX Handler

```php
add_action( 'wp_ajax_snow_record_event', 'snow_record_event' );
add_action( 'wp_ajax_nopriv_snow_record_event', 'snow_record_event' );

function snow_record_event() {
	check_ajax_referer( 'snow_ajax_nonce', 'nonce' );

	if ( ! isset( $_POST['event'] ) ) {
		wp_send_json_error( [ 'message' => 'missing event' ], 400 );
	}

	$event = sanitize_text_field( wp_unslash( $_POST['event'] ) );

	global $wpdb;
	$wpdb->insert(
		$wpdb->prefix . 'snow_events',
		[ 'event' => $event, 'created_at' => current_time( 'mysql', true ) ],
		[ '%s', '%s' ]
	);

	wp_send_json_success( [ 'ok' => true ] );
}
```

### Enqueue Admin Scripts (Scoped)

```php
add_action( 'admin_enqueue_scripts', function( $hook ) {
	if ( strpos( $hook, 'snow' ) === false ) {
		return;
	}
	wp_enqueue_style( 'snow-admin', SNOW_PLUGIN_URL . 'assets/css/admin.css', [], SNOW_VERSION );
	wp_enqueue_script( 'snow-admin', SNOW_PLUGIN_URL . 'assets/js/admin.js', [ 'jquery' ], SNOW_VERSION, true );
} );
```

## Documentation Prompts for Copilot

Use these prompts in comments or commit messages to steer generation:

* "Create a `Snow_Experiments` class that assigns a variant per user using a cookie + hash of user ID. Provide filters for assignment overrides."
* "Add REST endpoint `/snow/v1/experiments` secured with `manage_options` that lists active experiments."
* "Refactor `Snow_Analytics` to batch events and send via AJAX; add nonce & DNT checks."
* "Write PHPCS‑compliant unit tests for `Snow_Database::create_tables()`."

## Testing

* Prefer PHPUnit + WP Test Suite; place tests under `/tests` mirroring `src` structure.
* For JavaScript, use simple smoke tests if adding build steps later.

## Release & Versioning

* Semantic Versioning (SemVer). Bump `SNOW_VERSION` and `SNOW_DB_VERSION` together when DB changes.
* Tag releases; update `README.md` with changelog summary.

## Issue & PR Templates (suggested)

Place under `.github/`:

* `ISSUE_TEMPLATE/bug_report.md`
* `ISSUE_TEMPLATE/feature_request.md`
* `PULL_REQUEST_TEMPLATE.md`

## Final Word

Copilot should favor **clarity over cleverness**, minimize surface area, and always propose the secure, testable path—even if it adds a few lines.
