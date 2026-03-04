# Admin API Contract: Ingestion Pipeline (P2)

**Feature**: 002-document-ingestion | **Date**: 2026-03-03
**Priority**: P2 — Implemented after CLI (P1)

## Base Path

```
/api/v1/admin/knowledge-base
```

All endpoints require authenticated admin or super-admin role via API Gateway headers.

---

## Endpoints

### GET /api/v1/admin/knowledge-base/documents

List all indexed documents with metadata.

**Query Parameters**:
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| limit | number | 20 | Max 100 |
| offset | number | 0 | Pagination offset |
| status | string | — | Filter: PENDING, INDEXED, FAILED, REMOVED |
| job_id | uuid | — | Filter by ingestion job (useful for viewing failed documents in a specific run) |
| category | string | — | Filter by RagDocumentCategory |
| country_code | string | — | Filter: AE, SA |
| jurisdiction_code | string | — | Filter: UAE_MAINLAND, FREE_ZONE, KSA_MAINLAND |

**Response** (200):
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "uuid",
        "documentId": "uae_business-setup_1.3_companyTypes",
        "fileName": "1.3_UAE_CompanyTypes.docx",
        "title": "UAE Company Types",
        "jurisdictionCode": "UAE_MAINLAND",
        "countryCode": "AE",
        "category": "BUSINESS_SETUP",
        "sourceType": "INTERNAL",
        "version": 1,
        "chunkCount": 12,
        "status": "INDEXED",
        "lastIndexedAt": "2026-03-03T14:30:00Z"
      }
    ],
    "total": 67,
    "limit": 20,
    "offset": 0
  },
  "error": null
}
```

---

### POST /api/v1/admin/knowledge-base/ingest

Trigger ingestion for a single document or batch.

**Request Body**:
```json
{
  "paths": ["UAE/1-Business Setup/1.3_UAE_CompanyTypes.docx"],
  "force": false
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| paths | string[] | Yes | Relative paths from KB root. Use `["*"]` for full corpus. |
| force | boolean | No | Skip confirmation for full corpus re-ingestion |

**Response** (202 — Accepted):
```json
{
  "success": true,
  "data": {
    "jobId": "uuid",
    "jobType": "BATCH",
    "totalDocuments": 1,
    "status": "PENDING"
  },
  "error": null
}
```

---

### GET /api/v1/admin/knowledge-base/jobs/:jobId

Get ingestion job status and progress.

**Response** (200):
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "jobType": "FULL_CORPUS",
    "status": "PROCESSING",
    "totalDocuments": 109,
    "processedDocuments": 45,
    "failedDocuments": 2,
    "skippedDocuments": 12,
    "totalChunksCreated": 487,
    "startedAt": "2026-03-03T14:30:00Z",
    "completedAt": null
  },
  "error": null
}
```

---

### DELETE /api/v1/admin/knowledge-base/documents/:documentId

Remove a document and all its chunks from the index.

**Response** (200):
```json
{
  "success": true,
  "data": {
    "documentId": "uae_business-setup_1.3_companyTypes",
    "chunksDeleted": 12,
    "status": "REMOVED"
  },
  "error": null
}
```

---

### GET /api/v1/admin/knowledge-base/stats

Get knowledge base overview statistics.

**Response** (200):
```json
{
  "success": true,
  "data": {
    "collectionName": "knowledge_base",
    "vectorDimension": 3072,
    "totalPoints": 798,
    "totalDocuments": 67,
    "byJurisdiction": {
      "UAE_MAINLAND": { "documents": 45, "chunks": 612 },
      "KSA_MAINLAND": { "documents": 22, "chunks": 186 }
    },
    "byCategory": {
      "BUSINESS_SETUP": 18,
      "TAX_COMPLIANCE": 12
    },
    "lastIngestionJob": {
      "id": "uuid",
      "status": "COMPLETED",
      "completedAt": "2026-03-03T14:34:12Z"
    }
  },
  "error": null
}
```

---

## Error Responses

All errors follow the standard envelope:

```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "INGESTION_IN_PROGRESS",
    "message": "An ingestion job is already running. Wait for it to complete or use force=true."
  }
}
```

| HTTP Status | Error Code | Description |
|-------------|-----------|-------------|
| 400 | INVALID_PATH | File path does not exist or is not a supported format |
| 404 | DOCUMENT_NOT_FOUND | Document ID not found in index |
| 409 | INGESTION_IN_PROGRESS | Another ingestion job is currently running |
| 503 | QDRANT_UNAVAILABLE | Cannot connect to Qdrant |
