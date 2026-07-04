# Threat Model — File Upload

> Filled STRIDE model for accepting, storing, processing, and serving user-uploaded files. Method & risk rating: [README.md](README.md). Blank copy: [threat-model-template.md](threat-model-template.md).
> **Owner:** Security Officer · **Extends:** [04 Security](../standards-kb/04-security.md)

## Feature / asset

Any endpoint that accepts a file from a user (avatars, documents, imports, attachments) and stores, transforms (resize/parse/OCR/thumbnail), or serves it. **Asset:** server integrity, storage, and every user who later downloads the file. **Uploaded content is untrusted input — treat the file as hostile.**

## Data classification

Varies with content; assume up to **Confidential/PII** (documents may contain regulated data). Storage residency & retention → [06 Data Management](../standards-kb/06-data-management.md).

## Actors

| Actor | Trusted? | Notes |
|---|---|---|
| Authenticated uploader | Semi-trusted | may be compromised or malicious |
| **The uploaded file** | **Untrusted** | content, filename, and metadata are all attacker-controlled |
| Downstream viewer | Trusted | victim of stored payloads |
| Processing library / parser | Semi-trusted | large attack surface (image/XML/archive parsers) |

## Trust boundaries

- **B1** upload endpoint (client → app).
- **B5** processing pipeline (parser/converter — often the real vulnerability) and object storage.

## Attack surface

Multipart body, filename, declared `Content-Type`, file bytes/magic, embedded structures (XML/ZIP/EXIF), the storage path, the serving response headers, and any URL/reference the file causes the server to fetch.

## STRIDE analysis

| # | STRIDE | Threat | How (attack path) | Mitigation `[MUST]`/`[SHOULD]` | Standards ref | Residual |
|---|---|---|---|---|---|---|
| S1 | Spoofing | **Content-type / extension spoofing** | `.jpg` that is really an HTML/PHP/SVG payload; `Content-Type` lies | `[MUST]` verify by **magic-byte sniffing**, not extension or client-declared type; allow-list permitted types; re-encode images; store SVG only if sanitized | [04](../standards-kb/04-security.md) | Low |
| T1 | Tampering | **Path traversal** | Filename `../../etc/...` or absolute path writes outside the intended dir / overwrites files | `[MUST]` never use the client filename for the storage path; generate a random server-side key; strip/reject path separators; store outside the webroot | [04](../standards-kb/04-security.md) / [06](../standards-kb/06-data-management.md) | Low |
| T2 | Tampering | **Malware upload / distribution** | Attacker uploads a virus that others download; server becomes a malware host | `[MUST]` scan every upload with an AV/malware engine before it's usable; quarantine on hit; `[SHOULD]` re-scan on signature updates | [04](../standards-kb/04-security.md) | Med |
| R1 | Repudiation | **Denied upload** | User denies uploading a malicious/illegal file | `[MUST]` audit upload events (who/when/hash/size/type/correlation-id); store a content hash | [20](../standards-kb/20-logging-audit-and-traceability.md) | Low |
| I1 | Info disclosure | **XXE via XML/OOXML parsing** | Uploaded XML/DOCX/XLSX/SVG contains external entities that read local files or exfiltrate | `[MUST]` disable DTD/external-entity resolution in every XML parser; use hardened parsers for Office/SVG formats | [04](../standards-kb/04-security.md) | Low |
| I2 | Info disclosure | **SSRF via processing** | File references a remote URL (SVG `<image href>`, XML entity, PDF/URL fetch) that the server fetches — reaching internal metadata/services | `[MUST]` disable remote fetches in processors; if fetching is required, use an allow-list + block private/link-local ranges + no redirects (see [API & Webhooks](api-and-webhooks.md)) | [04](../standards-kb/04-security.md) / [10](../standards-kb/10-api-integration.md) | Med |
| I3 | Info disclosure | **Stored XSS via filename/metadata** | Malicious filename or EXIF rendered unescaped in the UI; SVG with inline script served inline | `[MUST]` output-encode filenames/metadata everywhere they're shown; serve downloads with `Content-Disposition: attachment` + `X-Content-Type-Options: nosniff`; serve from an isolated origin/CDN, never the app origin | [04](../standards-kb/04-security.md) / [11](../standards-kb/11-frontend-ux.md) | Low |
| I4 | Info disclosure | **Broken object authz on download** | Guessing/incrementing a file id/URL fetches another user's or tenant's file (IDOR) | `[MUST]` authorize every download server-side against owner+tenant; serve via short-lived signed URLs, not public/guessable paths | [04](../standards-kb/04-security.md) | Med |
| D1 | DoS | **Decompression bomb** | Tiny ZIP/gzip/image expands to gigabytes, exhausting memory/disk | `[MUST]` cap decompressed size and ratio; cap pixel dimensions before decode; stream with hard limits; time-box processing | [03](../standards-kb/03-performance-scalability.md) | Med |
| D2 | DoS | **Oversized / flood upload** | Huge or many uploads fill storage / saturate the pipeline | `[MUST]` enforce max file size at the edge (reject early, don't buffer whole); per-user rate/quota limits; offload processing to a bounded async queue | [10](../standards-kb/10-api-integration.md) / [03](../standards-kb/03-performance-scalability.md) | Low |
| E1 | Elevation | **RCE via uploaded executable** | Upload lands in a web-executable path; server runs it | `[MUST]` store in object storage / outside webroot; never grant execute; disable script execution in the upload dir | [04](../standards-kb/04-security.md) | Low |

## Top risks

1. **SSRF via processing (I2)** — image/XML/PDF processors that fetch remote refs are a classic pivot into the internal network.
2. **Decompression bomb (D1)** — cheap for an attacker, expensive for you; enforce ratio and dimension caps.
3. **Stored XSS / malware served to other users (I3, T2)** — you become the distribution vector; isolate the serving origin and scan.

## Open risks / decisions

| Item | Decision | Owner | Due |
|---|---|---|---|
| SVG support (script-capable) | Sanitize-and-rasterize, or reject | | |
| Where processing runs | Isolated/sandboxed worker, no internal network egress | | |

---

## Mitigation checklist

- [ ] `[MUST]` Validate type by magic bytes + allow-list; re-encode images.
- [ ] `[MUST]` Random server-generated storage key; client filename never used in the path.
- [ ] `[MUST]` Malware/AV scan before the file is usable; quarantine on hit.
- [ ] `[MUST]` XML/Office/SVG parsed with DTD/external entities disabled (no XXE).
- [ ] `[MUST]` No remote fetches in processors (SSRF); allow-list + private-range block if unavoidable.
- [ ] `[MUST]` Filenames/metadata output-encoded; downloads served `attachment` + `nosniff` from an isolated origin.
- [ ] `[MUST]` Per-object download authorization; signed, expiring URLs.
- [ ] `[MUST]` Decompression ratio/size + pixel-dimension caps; edge size limit; per-user quota.
- [ ] `[MUST]` Stored outside webroot, no execute; async bounded processing.
- [ ] `[MUST]` Upload/download audited with content hash → [20](../standards-kb/20-logging-audit-and-traceability.md).
