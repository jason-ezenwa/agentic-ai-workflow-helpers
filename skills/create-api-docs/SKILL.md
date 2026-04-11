---
name: create-api-docs
description: Generates comprehensive API documentation for REST endpoints with request/response examples, error handling, and usage notes.
---

# Create API Documentation

This skill guides the creation of API documentation using a standardized format that ensures consistency, clarity, and completeness.

## Instructions

1.  **Analyze the API**: Review the controller, service, and DTO files for the feature to understand all endpoints, request/response shapes, authentication requirements, and error cases.
2.  **Determine File Path**: The API documentation must be saved in the `docs` folder at the root of the workspace.
    -   Target Directory: `docs/api/` (Create this directory if it doesn't exist)
    -   Filename Format: `<feature-name>-api.md` (e.g., `docs/api/essay-api.md`, `docs/api/user-api.md`)
3.  **Generate Content**: Use the template below to structure the document.
    -   **Complete Coverage**: Document ALL endpoints for the feature, grouped logically by functionality.
    -   **Realistic Examples**: Use realistic, domain-appropriate example data in JSON payloads.
    -   **Professional Tone**: Maintain a clear, concise, and professional technical writing style.
4.  **Save File**: Write the generated content to the target file.

## API Documentation Template

````markdown
# [Feature Name] API Documentation

[Brief 1-2 sentence description of what this API provides and its primary use cases.]

## [Logical Grouping] Endpoints

### [Endpoint Name]
**Purpose**: [Clear description of what this endpoint does and when to use it.]

**Authentication**: [Required | None | Optional]
**Request**:
```
[METHOD] /path/:param
```

**Path Parameters**:
- `param` (required): [Description of the parameter]

**Query Parameters**:
- `queryParam` (optional): [Description, default value if any]

**Request Body**:
```json
{
  "field": "example value",
  "nested": {
    "subField": "value"
  }
}
```

**Response**:
```
HTTP/1.1 [STATUS_CODE] [STATUS_TEXT]
Content-Type: application/json
```
```json
{
  "_id": "507f1f77bcf86cd799439011",
  "field": "example value",
  "createdAt": "2026-01-24T12:00:00.000Z",
  "updatedAt": "2026-01-24T12:00:00.000Z"
}
```

**Error Responses**:
- 400 Bad Request - [Specific reason, e.g., "Missing required fields"]
- 401 Unauthorized - [Specific reason, e.g., "Not authenticated"]
- 403 Forbidden - [Specific reason, e.g., "Access denied"]
- 404 Not Found - [Specific reason, e.g., "Resource not found"]
- 500 Internal Server Error - [Specific reason, e.g., "Database operation failed"]

**Notes**:
- [Important implementation details, edge cases, or behavior notes]
- [Side effects or triggers, e.g., "Sends notification email"]
- [Default values or automatic field population]

---

[Repeat for each endpoint...]
````

## Documentation Standards

### Endpoint Structure
Each endpoint MUST include:
1. **Purpose**: What the endpoint does (action-oriented)
2. **Authentication**: Required, None, or Optional with details
3. **Request**: HTTP method and path with parameter placeholders
4. **Response**: Status code, headers, and JSON response body
5. **Error Responses**: All possible error codes with reasons
6. **Notes**: Implementation details, edge cases, and important behaviors

### Grouping Strategy
Group endpoints logically:
- **CRUD Operations**: Create, Read, Update, Delete for a resource
- **Action Endpoints**: Special operations like review, generate, submit
- **Public/Token-Based**: Endpoints that bypass standard authentication
- **Admin/Special Access**: Endpoints with elevated permissions

### Example Data Guidelines
- Use realistic, domain-specific example values
- Include MongoDB ObjectIds in proper format: `507f1f77bcf86cd799439011`
- Use ISO 8601 timestamps: `2026-01-24T12:00:00.000Z`
- Include `createdAt` and `updatedAt` where applicable
- Show nested objects and arrays when relevant

### Error Response Guidelines
Document ALL possible error responses:
- **400**: Invalid input, missing fields, validation failures
- **401**: Authentication required but not provided
- **403**: Authenticated but not authorized for this action
- **404**: Resource not found or does not belong to user
- **409**: Conflict (e.g., duplicate resource)
- **500**: Server-side errors, external service failures

### Notes Section Guidelines
Include when applicable:
- Default values for fields
- Automatic status changes
- Pagination behavior
- Sorting order
- Security considerations (e.g., "Token intentionally omitted for security")
- Side effects (emails, notifications, background jobs)
- Rate limiting
- Caching behavior

### Special Endpoint Types

#### Streaming/SSE Endpoints
````markdown
**Response**:
```
HTTP/1.1 200 OK
Content-Type: text/event-stream
```

Streamed response in Server-Sent Events (SSE) format.

**Example SSE Stream**:
```
data: {"type":"text","content":"First chunk..."}

data: {"type":"text","content":"Second chunk..."}

data: {"type":"done"}
```
````

#### Token-Validated (Public) Endpoints
````markdown
**Authentication**: None (token in path validates access)

**Notes**:
- Uses `@AllowAnonymous()` decorator to bypass authentication
- Token validates access instead of Bearer authentication
````

#### List Endpoints with Pagination
````markdown
**Query Parameters**:
- `page` (optional): Page number, default: 1
- `limit` (optional): Items per page, default: 20, max: 100
- `sort` (optional): Sort field, prefix with `-` for descending
- `filter` (optional): Filter criteria

**Response**:
```json
{
  "items": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```
````

## Validation Checklist
Before finalizing documentation, verify:
- [ ] All endpoints are documented
- [ ] Request/response examples are valid JSON
- [ ] All error codes are listed with specific reasons
- [ ] Path parameters are documented
- [ ] Query parameters include defaults
- [ ] Authentication requirements are clear
- [ ] Notes cover edge cases and important behaviors
- [ ] Examples use realistic data
- [ ] HTTP status codes are accurate
- [ ] Content-Type headers are specified
