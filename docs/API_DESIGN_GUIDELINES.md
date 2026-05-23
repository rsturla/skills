# API Design Guidelines

## General Principles

- APIs are contracts — breaking changes require a new version
- Design for the consumer, not the implementation
- Be consistent across endpoints — same patterns everywhere
- Prefer explicit over clever

## REST APIs

### URL Structure

Resource-oriented, plural nouns, no verbs in paths:

```text
GET    /v1/images                    # list
POST   /v1/images                    # create
GET    /v1/images/{id}               # get
PUT    /v1/images/{id}               # replace
PATCH  /v1/images/{id}               # partial update
DELETE /v1/images/{id}               # delete
GET    /v1/images/{id}/vulnerabilities  # nested resource
```

- Lowercase, hyphen-separated: `/v1/build-configs` not `/v1/buildConfigs`
- Max 2 levels of nesting — beyond that, use query params or top-level resources
- No trailing slashes

### Versioning

Version in URL path, not headers:

```text
/v1/images
/v2/images
```

- Increment major version only for breaking changes
- Support previous version for a documented deprecation period
- Never remove fields from a response without a version bump

### HTTP Methods and Status Codes

| Method | Success | Meaning |
| ------ | ------- | ------- |
| GET | 200 | Resource found |
| POST | 201 | Resource created (return `Location` header) |
| PUT | 200 | Resource replaced |
| PATCH | 200 | Resource updated |
| DELETE | 204 | Resource deleted (no body) |

Common errors:

| Code | When |
| ---- | ---- |
| 400 | Malformed request, validation failure |
| 401 | Missing or invalid authentication |
| 403 | Authenticated but not authorized |
| 404 | Resource not found |
| 409 | Conflict (duplicate, state mismatch) |
| 422 | Request is well-formed but semantically invalid |
| 429 | Rate limited |
| 500 | Server error (never leak internals) |

### Error Format (RFC 7807)

Use [Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807):

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Field 'name' must be between 1 and 255 characters",
  "instance": "/v1/images/abc-123"
}
```

- `type` — URI identifying the error type (stable, documented)
- `title` — human-readable summary (same for all instances of this type)
- `status` — HTTP status code
- `detail` — human-readable explanation specific to this occurrence
- `instance` — URI of the specific resource/request

For validation errors with multiple fields, extend with an `errors` array:

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "errors": [
    {"field": "name", "detail": "must not be empty"},
    {"field": "tag", "detail": "invalid format, expected semver"}
  ]
}
```

### Pagination

Use cursor-based pagination for large or frequently changing datasets. Offset-based for stable, small datasets.

```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTAwfQ==",
    "has_more": true
  }
}
```

Request: `GET /v1/images?cursor=eyJpZCI6MTAwfQ==&limit=25`

- Default limit: 25, max limit: 100
- Always return `has_more` so clients know when to stop
- Never use page numbers for large datasets — they break under concurrent writes

### Filtering and Sorting

```text
GET /v1/images?status=active&sort=-created_at&limit=10
```

- Filter by query params matching field names
- Sort with `-` prefix for descending: `sort=-created_at`
- Multiple sort fields: `sort=-priority,created_at`

### Request/Response Conventions

- Use `snake_case` for JSON field names
- Dates in ISO 8601: `"2026-05-23T14:30:00Z"`
- IDs as strings (even if numeric internally) — avoids JavaScript integer precision issues
- Envelope responses with `data` key for lists, bare object for single resources
- Include `created_at` and `updated_at` on all resources
- Never return internal IDs, database column names, or stack traces

## gRPC APIs

### Protobuf Conventions

- Package names: `com.example.service.v1`
- Service names: PascalCase (`ImageService`)
- RPC methods: PascalCase verb+noun (`ListImages`, `GetImage`, `CreateImage`, `DeleteImage`)
- Message names: PascalCase (`ListImagesRequest`, `ListImagesResponse`)
- Field names: `snake_case`
- Enums: `UPPER_SNAKE_CASE` values, first value is `UNSPECIFIED` (zero value)

```protobuf
syntax = "proto3";
package example.images.v1;

service ImageService {
  rpc ListImages(ListImagesRequest) returns (ListImagesResponse);
  rpc GetImage(GetImageRequest) returns (Image);
  rpc CreateImage(CreateImageRequest) returns (Image);
  rpc DeleteImage(DeleteImageRequest) returns (google.protobuf.Empty);
}

enum ImageStatus {
  IMAGE_STATUS_UNSPECIFIED = 0;
  IMAGE_STATUS_ACTIVE = 1;
  IMAGE_STATUS_DEPRECATED = 2;
}
```

### Versioning (gRPC)

Version in the package name: `example.images.v1`, `example.images.v2`. Never break existing message
fields — add new ones, deprecate old ones.

### Error Handling

Use standard gRPC status codes with detail messages:

| Code | When |
| ---- | ---- |
| `NOT_FOUND` | Resource doesn't exist |
| `ALREADY_EXISTS` | Duplicate create |
| `INVALID_ARGUMENT` | Bad input |
| `PERMISSION_DENIED` | Not authorized |
| `UNAUTHENTICATED` | No credentials |
| `RESOURCE_EXHAUSTED` | Rate limited or quota exceeded |
| `INTERNAL` | Server bug (never leak details) |

### Pagination (gRPC)

```protobuf
message ListImagesRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message ListImagesResponse {
  repeated Image images = 1;
  string next_page_token = 2;
}
```

## Authentication

- Use bearer tokens in `Authorization` header (REST) or metadata (gRPC)
- Never pass credentials in query params — they end up in logs
- Support API keys for service-to-service, OAuth2/OIDC for users
- Always require TLS

## Gotchas

- `PUT` replaces the entire resource — missing fields get zeroed. Use `PATCH` for partial updates.
- 404 vs 403: return 404 for resources the user can't see. Returning 403 leaks existence.
- Pagination cursors must be opaque to clients — don't expose internal IDs or offsets.
- gRPC enum zero value must be `UNSPECIFIED` — proto3 defaults to 0, so a meaningful first value gets
  confused with "not set".
- Don't version individual endpoints — version the entire API surface together.
- Rate limiting: return `Retry-After` header with 429 responses.
