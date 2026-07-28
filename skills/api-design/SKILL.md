---
name: api-design
description: Use when designing or reviewing RESTful APIs — defining endpoints, resources, URLs, JSON payloads, status codes, pagination, or OpenAPI specs. Enforces company REST guidelines: resource-oriented URLs, snake_case JSON, problem JSON errors, pagination, compatible evolution.
---

# Designing RESTful APIs

Decoupled microservices expose functionality via RESTful APIs with JSON payloads.
Requirement keywords (MUST, SHOULD, MAY) follow RFC 2119. New APIs MUST respect these
guidelines; existing APIs don't have to be changed (but it's recommended).

Full reference: [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)
— fetch it (WebFetch) when a topic needs more detail than the rules below.

## Meta

- **MUST** provide an API specification using OpenAPI.
- **MUST** write APIs in U.S. English.
- **SHOULD** use semantic versioning: MAJOR for incompatible changes (aligned with
  consumers first), MINOR for backwards-compatible functionality, PATCH for
  backwards-compatible fixes/editorial changes.

## Data formats

- **MUST** encode binary data (internal usage) in base64url.
- **MUST** use standard formats for date/time, country, language, and currency properties.
- **SHOULD** use UUIDs for identifiers — V7 recommended for primary keys, V4 otherwise.

## URLs

- **MUST** pluralize resource names; use domain-specific resource names.
- **MUST** use kebab-case for path segments and URL-friendly resource identifiers.
- **MUST** use normalized paths — no empty segments; **SHOULD** avoid trailing slashes.
- **MUST** identify resources and sub-resources via path segments:
  `/tenants/{tenant_id}/sub-resource`; **MAY** expose compound keys:
  `/tenants/{tenant_id}/sub-resource/{id}`.
- **SHOULD** keep URLs verb-free — model resources, not actions.
- **SHOULD** limit sub-resource nesting to 3 levels; **MAY** use non-nested URLs.
- **SHOULD** use snake_case for query parameters.
- **SHOULD** not use `/api` as base path (internal services).

## JSON payload

- **MUST** use JSON (UTF-8) as the payload interchange format; **MUST** always return JSON
  objects (never arrays) as top-level data structures.
- **MAY** pass non-JSON media types only for business-object-specific standard formats
  (images JPG/PNG/GIF, documents PDF/DOC/ODF/PPT, archives TAR/ZIP); **SHOULD** use
  standard media types.
- **SHOULD** use snake_case property names, pluralized array names, and
  UPPER_SNAKE_CASE for enum values.
- **SHOULD** name date/time properties with an `_at` suffix: `created_at`, `modified_at`,
  `occurred_at` — not `created`, `modified`, `occurred`.
- **SHOULD** not use `null` for booleans or for empty arrays (use `[]`).
- **MUST** use common field names: `id` for the object's identity; references to other
  objects as `<type-or-relationship>_id` (e.g. `partner_id`, not `partner_number`).
- **MUST** return all properties (with null or default value) and parse absence of a
  property as null (or the endpoint-specific default).

## HTTP requests

- **MUST** use HTTP methods correctly and never use GET with a body.
- **MUST** fulfill common method properties: *safe* (read-only, no intended side effects),
  *idempotent* (same intended effect whether executed once or many times — responses may
  differ), *cacheable* (safe requests generally are, unless an authoritative response is
  required).
- **SHOULD** design POST and PATCH idempotent; **MAY** support asynchronous processing.
- **SHOULD** design simple query languages with query parameters:
  `name=test` (equals), `price__le=5` (≤), `price__ge=5` (≥).
- **MAY** design complex query languages using JSON bodies via POST to a `/query` suffix:

  ```json
  {
    "or": [
      {"field": "name", "operator": "equals", "value": ["Alice"]},
      {"field": "age", "operator": "le", "value": [25]}
    ]
  }
  ```

## HTTP status codes

- **MUST** use official status codes, the most specific one applicable, and specify both
  success and error responses in the spec.
- **SHOULD** only use the most common codes: 200, 201, 202, 204, 400, 401, 403, 404, 405,
  409, 429, 500, 501, 502, 503, 504.
- **MUST** support problem JSON (RFC 9457) for errors.
- **MUST** not expose stack traces (security)
- **SHOULD** not use redirection codes.

## Hypermedia

- **MUST** use REST Maturity Level 2.
- **MUST** use common hypertext controls: `href` holds the URI of the linked resource,
  always with HTTPS scheme.

## Performance

- **SHOULD** reduce bandwidth and improve responsiveness; use gzip compression based on
  the `Accept-Encoding` header.
- **MAY** support partial responses via a `fields` query parameter and optional embedding
  of sub-resources.

## Pagination

- **SHOULD** prefer cursor-based pagination over offset-based for large collections, and
  use an `items` response object.
- **SHOULD** avoid a total result count — computing it usually requires a full index scan
  and grows expensive over the service's lifetime.

Cursor-based response:

```json
{
  "items": [],
  "next_page": "MTcxNzE0NDExMC43MDExNjE="
}
```

Page-based pagination **MUST** use `page` & `page_size` query parameters and respond with:

```json
{
  "items": [],
  "current_page": 1,
  "page_size": 50,
  "total_pages": 100
}
```

## Compatibility

- **SHOULD** not break backward compatibility; **SHOULD** avoid versioning and **MUST**
  not use URL versioning.
- **SHOULD** prefer compatible extensions:
  - Add only optional fields, never mandatory ones.
  - Never change a field's semantics (e.g. customer-number → customer-id).
  - Never make validation logic more restrictive; define all constraints clearly.
  - Enum ranges: may be reduced for input (if the server still accepts old values) and for
    output; may be extended for input, but never for output — clients may not handle it.
- **SHOULD** design conservatively: ignore unknown input fields, define input constraints
  precisely (formats, ranges, lengths), validate them, and return dedicated error
  information on violations. Prefer specific/restrictive definitions — they simplify
  implementation and leave room for compatible extension.

## Deprecation

- **MUST** reflect deprecation in the API specification.
- **MAY** add `Deprecation` and `Sunset` response headers.
- **MUST** not start using deprecated APIs.
