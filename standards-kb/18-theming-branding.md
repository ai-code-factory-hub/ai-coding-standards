# 18 · Theming, Branding & UI Personalization

A token-driven theming and white-label system that lets every tenant and user personalize the UI without forking code — consistent across app, print, and email.

The full token architecture, palette, and built-in theme catalog live in **[../design-system-kb/](../design-system-kb/README.md)**. This domain mandates the requirements that system must satisfy.

## Token architecture

- **[MUST]** A token-driven theme engine with a clear hierarchy: **primitive → semantic → component** tokens. Components consume semantic/component tokens and **never** reference raw hex values.
- **[MUST]** Themes are data, not code branches — adding or editing a theme changes token values, not component logic.

## Theme catalog

- **[MUST]** Ship light, dark, and a "follow system" mode, plus a catalog of **≥12 built-in themes**, each available in light and dark, including at least **one high-contrast theme** (AAA-level) and at least **one colorblind-safe theme**.
- **[MUST]** Apply themes at **system → tenant → user** precedence (later wins); persist the user's choice, sync it across their sessions/devices, render with **no wrong-theme flash** on load, and fall back safely if a stored theme is missing.

## Navigation & density

- **[MUST]** Navigation is configurable: position **left (default) / top / right / bottom**, items renamable and reorderable, show/hide by role or entitlement, collapsible, with selectable density (comfortable / compact).

## User-defined colors

- **[MUST]** Let users define colors (primary/accent, notification colors, sidebar), each with a default and a reset. **Contrast auto-validation blocks unreadable combinations** before they are saved. Tenants can lock brand colors so users cannot override them.

## White-label

- **[MUST]** Per-tenant white-label: logo (light + dark variants), favicon, app name, and brand color applied consistently across **app, login, email, print, and PDF**; optional custom domain.
- **[MUST]** Theme tokens flow through to generated **PDFs, print templates, and emails**, so branding is identical on every channel, not just the web UI.

### Examples

- **Healthcare:** a lab tenant applies its logo and brand color across the portal, login page, and the generated diagnostic-report PDF; a colorblind-safe theme ships built-in so result status is never conveyed by color alone (reinforcing [13 · Internationalization & Accessibility](13-i18n-accessibility.md)).
- **Fintech / e-commerce:** a merchant white-labels the checkout and transactional emails with its own domain and brand color; contrast auto-validation prevents an admin from setting a low-contrast accent that would make CTAs unreadable.

## Acceptance checklist

- [ ] Token engine with primitive→semantic→component layers; no raw hex in components.
- [ ] Light + dark + follow-system, plus ≥12 built-in themes including a high-contrast and a colorblind-safe theme.
- [ ] System→tenant→user precedence; persisted + synced; no wrong-theme flash; safe fallback.
- [ ] Navigation position / rename / reorder / role-hide / density configurable.
- [ ] User-definable colors with default + reset + contrast auto-validation + tenant brand-lock.
- [ ] White-label (logo/favicon/name/brand color/custom domain) consistent across app, login, email, print, and PDF.
