# Admin · Localization & Content

> Manage locales, translation keys/strings, per-locale content and template localization — with fallback chains and translation status/coverage so nothing ships as a raw key or an untranslated string.

## What it is / when you need it

The place where admins add languages, translate every user-facing string, localize email/notification templates and marketing content, and see coverage before flipping a locale live. You need it the moment the product serves more than one language or region — and even a single-locale product benefits from externalizing strings so no copy is hard-coded. It operationalizes [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md): externalized strings, ICU pluralization, RTL support, and a defined fallback chain.

## Data model

Tenant-scoped where tenants customize copy (with a `scope` = `system`/`tenant` split for platform defaults vs tenant overrides); standard audit columns.

### Locale

| Field | Type | Notes |
| --- | --- | --- |
| `code` | string | BCP-47, e.g. `en`, `fr-CA`, `ar` |
| `name` | string | Display, e.g. "Français (Canada)" |
| `direction` | enum | `ltr` \| `rtl` |
| `fallback_code` | string? | Next locale in the fallback chain (e.g. `fr-CA` → `fr` → `en`) |
| `status` | enum | `draft` \| `active` \| `hidden` — only `active` is user-selectable |
| `is_default` | bool | The source/base locale (usually `en`) |
| `plural_rules` | string | CLDR plural category set |

### TranslationKey

| Field | Type | Notes |
| --- | --- | --- |
| `key` | string | Namespaced, e.g. `invoice.header.title` |
| `namespace` | string | e.g. `billing`, `emails`, `errors` |
| `source_text` | text | Base-locale string (source of truth) |
| `description` | text? | Context note for translators |
| `is_html` | bool | Rich vs plain |
| `placeholders_json` | jsonb | Named variables + ICU format hints |
| `is_deprecated` | bool | Retired but retained for history |

### TranslationString (per locale)

| Field | Type | Notes |
| --- | --- | --- |
| `key` | fk | → TranslationKey |
| `locale_code` | fk | → Locale |
| `value` | text | Translated string (ICU MessageFormat) |
| `status` | enum | `missing` \| `draft` \| `needs_review` \| `approved` \| `stale` |
| `translated_by` / `reviewed_by` | fk? | Actors |
| `source_hash` | string | Hash of `source_text` when translated — mismatch ⇒ `stale` |
| `updated_at` | timestamp | |

### LocalizedContent (documents / snippets)

| Field | Type | Notes |
| --- | --- | --- |
| `content_key` | string | e.g. `terms_of_service`, `help.getting_started` |
| `locale_code` | fk | |
| `title` / `body` | text | Rich content |
| `status` | enum | `draft` \| `published` \| `archived` |
| `version` | int | For rollback/history |

### TemplateLocalization

| Field | Type | Notes |
| --- | --- | --- |
| `template_key` | string | → notification/email template (see [`../feature-blueprints/notifications-center.md`](../feature-blueprints/notifications-center.md)) |
| `locale_code` | fk | |
| `subject` / `body` | text | Localized template with placeholders |
| `status` | enum | `missing` \| `draft` \| `approved` |

### CoverageSnapshot (rollup)

| Field | Type | Notes |
| --- | --- | --- |
| `locale_code` | fk | |
| `namespace` | string? | Null = whole locale |
| `total_keys` / `translated` / `approved` / `stale` | int | Drives the coverage %/badges |

## Key screens & UX

1. **Locales** — list of locales with direction, fallback chain, status, and coverage %. Add/edit locale; a locale can only go `active` when coverage meets a required threshold (or an explicit override with reason). *Empty:* "Only the default locale exists — add a language." *Error:* retry.
2. **Translation editor** — key list filtered by namespace/status/locale, side-by-side source vs target, ICU/placeholder validation, and per-string status transitions (draft → needs_review → approved). Bulk actions: mark reviewed, import/export (XLIFF/CSV for translation vendors). *Loading:* skeleton; *Empty:* "No keys in this namespace."
3. **Coverage dashboard** — per-locale, per-namespace progress bars: translated %, approved %, and a **stale** count (strings whose source changed since translation). Highlights blockers before a locale goes live.
4. **Content editor** — localized documents/snippets with versioned draft → publish, per-locale, and a fallback-preview showing what users of an untranslated locale will actually see.
5. **Template localization** — per template, per locale subject/body editors with live placeholder preview and a "missing translations" list.
6. **Fallback preview** — pick a locale + key and see the resolved value walking the fallback chain, so gaps are visible, never a raw key.

## Rules & logic

- [MUST] **No raw keys or hard-coded copy** reach users — every user-facing string is an externalized `TranslationKey` resolved at render per [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md).
- [MUST] Missing translations resolve via the **defined fallback chain** (`fr-CA` → `fr` → default) and **never render the raw key**; a locale below its coverage threshold cannot be published without an explicit, audited override.
- [MUST] Strings use **ICU MessageFormat** with CLDR plural rules and named placeholders; the editor validates that target placeholders match the source before `approved`.
- [MUST] Changing a key's `source_text` marks all existing translations **`stale`** (via `source_hash` mismatch) so they surface for re-review — silent drift is not allowed.
- [MUST] Locales carry `direction`; **RTL** locales flip layout correctly (see [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md)).
- [MUST] All add/edit/publish/delete actions are **RBAC-gated** (localization admin/translator roles) and **audited**, tenant-scoped, per [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md).
- [MUST] Tenant overrides shadow **system** defaults per key/locale; the resolution order (tenant > system > fallback) is deterministic.
- [SHOULD] Support **import/export** in a standard exchange format (XLIFF/CSV) for external translation vendors, round-tripping status.
- [SHOULD] Never concatenate translated fragments; keep whole sentences as single keys to allow correct word order across languages.
- [SHOULD] Content and templates are **versioned** with rollback; publishing is an audited event.

## Applicable standards

- Externalized strings, ICU/pluralization, RTL, fallback, locale switching — [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md)
- Localization admin audit trail — [`../standards-kb/20-logging-audit-and-traceability.md`](../standards-kb/20-logging-audit-and-traceability.md)
- Tenant vs system content scope — [`../standards-kb/01-architecture-multitenancy.md`](../standards-kb/01-architecture-multitenancy.md)
- Content/template UX & states — [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- Theming/branding overlap for tenant copy — [`../standards-kb/18-theming-branding.md`](../standards-kb/18-theming-branding.md)
- Related — [`../feature-blueprints/notifications-center.md`](../feature-blueprints/notifications-center.md), [`../admin-kb/README.md`](../admin-kb/README.md)

## Acceptance checklist

- [ ] All user-facing strings are externalized keys; no hard-coded copy or raw keys render.
- [ ] Missing strings resolve via the defined fallback chain, never showing the raw key.
- [ ] A locale cannot be published below its coverage threshold without an audited override.
- [ ] Strings use ICU MessageFormat; the editor validates placeholders before approval.
- [ ] Editing source text marks dependent translations `stale` for re-review.
- [ ] RTL locales flip layout/direction correctly.
- [ ] Coverage dashboard shows translated/approved/stale per locale and namespace.
- [ ] Import/export round-trips a standard format (XLIFF/CSV) with status.
- [ ] Content and templates are versioned with rollback; publish is audited.
- [ ] All actions are RBAC-gated, tenant-scoped, and audited; loading/empty/error states designed.
