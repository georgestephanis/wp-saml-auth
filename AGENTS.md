# WP SAML Auth (Pantheon)

**Plugin:** WP SAML Auth
**Version:** 2.3.1
**License:** GPLv2 or later
**Author:** Pantheon
**Requires PHP:** 7.4
**Tested up to:** WordPress 6.9

## Overview

SAML 2.0 Single Sign-On for WordPress. IdP-agnostic — works with any SAML 2.0 compliant Identity Provider (Okta, Azure AD/Entra ID, Google Workspace, ADFS, Shibboleth, etc.). No upsell, no license required, no feature gating. All features are available in the single free plugin.

Uses `onelogin/php-saml` for SAML processing and `robrichards/xmlseclibs` for XML signature validation. Both are bundled in `vendor/`.

## Directory Structure

```
wp-saml-auth.php                   Main plugin file; bootstraps autoloader and main class
inc/
  class-wp-saml-auth.php           Core controller: ACS endpoint, login flow, user lookup/creation
  class-wp-saml-auth-cli.php       WP-CLI commands
  class-wp-saml-auth-options.php   Options abstraction; reads from DB or wp_saml_auth_option filter
  class-wp-saml-auth-settings.php  Admin settings UI (Settings → WP SAML Auth)
vendor/
  onelogin/php-saml/               SAML 2.0 library (AuthnRequest, Response validation, SLO)
  robrichards/xmlseclibs/          XML digital signature / encryption primitives
languages/                         .pot + French translations
```

## Authentication Flow

1. User hits the login page. If `permit_wp_login` is false, plugin immediately initiates SSO.
2. `$provider->login()` builds a signed `AuthnRequest` and redirects to the IdP SSO URL.
3. IdP authenticates the user and POSTs a `SAMLResponse` back to `wp-login.php`.
4. Plugin calls `$provider->processResponse()` — validates XML signature against the IdP's X.509 certificate, checks timestamps, extracts the assertion.
5. Attributes are pulled via `$provider->getAttributes()` and mapped to WordPress user fields per configuration.
6. User is looked up by email or login. If found, logged in. If not found and `auto_provision` is enabled, a new WordPress user is created.
7. On logout, if SLO is configured, `$provider->logout()` initiates a SAML logout request to the IdP.

## Configuration

Two approaches — **only one can be active at a time**. If the `wp_saml_auth_option` filter is registered, the admin UI is fully disabled.

### Approach 1: Admin UI

Settings → WP SAML Auth. Values are stored in `wp_options` under `wp_saml_auth_settings`. Use this for simple deployments.

### Approach 2: Filter (code-based)

Place in a mu-plugin or `wp-config.php`. Gives access to all `onelogin/php-saml` options.

```php
add_filter( 'wp_saml_auth_option', function( $value, $option_name ) {
    $config = [
        'connection_type' => 'internal',
        'internal_config' => [
            'strict'  => true,
            'debug'   => false,
            'sp'      => [
                'entityId'                 => 'urn:' . parse_url( home_url(), PHP_URL_HOST ),
                'assertionConsumerService' => [
                    'url'     => wp_login_url(),
                    'binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST',
                ],
                'singleLogoutService'      => [
                    'url'     => wp_logout_url(),
                    'binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
                ],
                'NameIDFormat' => 'urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress',
                'x509cert'     => '',
                'privateKey'   => '',
            ],
            'idp' => [
                'entityId'            => '',   // IdP Issuer URI
                'singleSignOnService' => [
                    'url'     => '',           // IdP SSO endpoint
                    'binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
                ],
                'singleLogoutService' => [
                    'url'     => '',           // IdP SLO endpoint (optional)
                    'binding' => 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect',
                ],
                'x509cert' => '',              // IdP public certificate (no headers)
            ],
        ],
        'auto_provision'    => true,           // Create WP user on first SSO login
        'permit_wp_login'   => true,           // false = disable WP login form entirely
        'get_user_by'       => 'email',        // 'email' or 'login'
        'user_login_attribute'   => 'uid',
        'user_email_attribute'   => 'mail',
        'display_name_attribute' => 'display_name',
        'first_name_attribute'   => 'first_name',
        'last_name_attribute'    => 'last_name',
        'default_role'           => get_option( 'default_role' ),
    ];
    return $config[ $option_name ] ?? $value;
}, 10, 2 );
```

## Okta Configuration

Values to copy from Okta into the plugin:

| Okta admin field                  | Plugin setting                                                   |
| --------------------------------- | ---------------------------------------------------------------- |
| Identity Provider issuer          | `idp.entityId`                                                   |
| IdP Single Sign-On URL            | `idp.singleSignOnService.url`                                    |
| IdP Single Logout URL             | `idp.singleLogoutService.url`                                    |
| X.509 Certificate (from metadata) | `idp.x509cert` (strip `-----BEGIN/END CERTIFICATE-----` headers) |
| (you set this) ACS URL            | `sp.assertionConsumerService.url` → `wp_login_url()`             |
| (you set this) SP Entity ID       | `sp.entityId` → e.g. `urn:yourdomain.com`                        |

Okta's default attribute names differ from the plugin's defaults. Either update the attribute mapping config or configure Okta to send attributes with these names:

| Okta attribute statement | Plugin default expects                          |
| ------------------------ | ----------------------------------------------- |
| `email` or `login`       | `mail` (change `user_email_attribute` to match) |
| `firstName`              | `first_name`                                    |
| `lastName`               | `last_name`                                     |

Okta sends unencrypted assertions by default — no special handling needed.

## Key Filters & Actions

| Hook                                       | Purpose                                                                                                     |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| `wp_saml_auth_option`                      | Override any config option (disables admin UI when present)                                                 |
| `wp_saml_auth_pre_authentication`          | Runs after SAML validation, before WP login. Return `WP_Error` to reject. Use for group/role authorization. |
| `wp_saml_auth_insert_user`                 | Filter the user data array before `wp_insert_user()` on auto-provision                                      |
| `wp_saml_auth_existing_user_authenticated` | Fires after an existing user logs in via SSO                                                                |
| `wp_saml_auth_new_user_authenticated`      | Fires after a new user is auto-provisioned and logged in                                                    |

## Feature Comparison vs. miniOrange Free

| Feature                          | WP SAML Auth                     | miniOrange Free                 |
| -------------------------------- | -------------------------------- | ------------------------------- |
| Okta / any SAML 2.0 IdP          | Yes                              | Yes                             |
| Auto-redirect (no WP login form) | Yes (`permit_wp_login => false`) | No                              |
| Single Logout (SLO)              | Yes                              | No                              |
| Attribute mapping                | Yes, all fields                  | Email + username only           |
| Auto user provisioning           | Yes                              | Yes                             |
| Admin UI                         | Yes                              | Yes                             |
| Upsell nags                      | None                             | Yes                             |
| License cost                     | Free, GPL                        | Free tier; $350/yr for Standard |
| Encrypted assertions             | Yes (via onelogin/php-saml)      | No (hard exception in free)     |

## Debugging Tips

- Set `'debug' => true` in `internal_config` to have `onelogin/php-saml` write SAML messages to the PHP error log.
- Enable WordPress debug logging: `studio site set --debug-log --debug-display`
- Check `wp-content/debug.log` for SAML validation errors (clock skew, certificate mismatch, audience restriction failures).
- The `wp_saml_auth_pre_authentication` filter receives the full `$attributes` array — log it to inspect exactly what the IdP is sending.
- Use the IdP's SAML tracer / test tool to verify the `AuthnRequest` is well-formed before debugging the response side.
