# Pulse API Onboarding (Public)

## 1) Overview
Pulse API provides multimodal UGC auditing for campaign compliance.

Base URL (production):

```txt
https://pulse.drafteye.app/api
```

Main outcome: each analysis returns `analysis.v2` with stable blocks:
- `expectations`
- `evidences`
- `matches`
- `violations`
- `compliance_summary`

---

## 2) Authentication
All business endpoints require headers:

- `X-Tenant-Id`
- `X-API-Key`

For analysis creation, also send:

- `Idempotency-Key` (required)

Example shell setup:

```bash
export API_URL="https://pulse.drafteye.app/api"
export TENANT_ID="acme"
export API_KEY="test-key"
```

---

## 3) Quickstart (5 Steps)

### Step 1 — Create/Update campaign

```bash
curl -sS -X POST "$API_URL/v1/campaigns" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: $TENANT_ID" \
  -H "X-API-Key: $API_KEY" \
  -d '{
    "campaign_external_id": "camp-andorinha-2026-05",
    "name": "Páscoa Andorinha 2026",
    "platforms": ["tiktok", "instagram_reels", "youtube_shorts"],
    "niche": "culinaria",
    "briefing": {
      "source": {"type": "inline_text", "mime_type": "text/plain"},
      "text": "Objetivo da campanha...",
      "format_hint": "plain",
      "language": "pt-BR",
      "version": "v1"
    }
  }' | jq
```

### Step 2 — Create script

```bash
export CAMPAIGN_ID="camp-andorinha-2026-05"

curl -sS -X POST "$API_URL/v1/scripts" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: $TENANT_ID" \
  -H "X-API-Key: $API_KEY" \
  -d '{
    "campaign_id": "'"$CAMPAIGN_ID"'",
    "creator": {
      "name": "Larissa",
      "external_id": "creator_larissa"
    },
    "source": {
      "type": "text",
      "text": "Roteiro do creator..."
    }
  }' | tee /tmp/script.json | jq

export SCRIPT_ID="$(jq -r '.script_id' /tmp/script.json)"
```

### Step 3 — Create asset

```bash
curl -sS -X POST "$API_URL/v1/assets" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: $TENANT_ID" \
  -H "X-API-Key: $API_KEY" \
  -d '{
    "campaign_id": "'"$CAMPAIGN_ID"'",
    "creator_external_id": "creator_larissa",
    "asset_source": {
      "type": "direct_url",
      "url": "https://drive.google.com/uc?export=download&id=VIDEO_ID"
    }
  }' | tee /tmp/asset.json | jq

export ASSET_ID="$(jq -r '.asset_id' /tmp/asset.json)"
```

### Step 4 — Create analysis

```bash
export IDEMP_KEY="analysis-$(date +%s)"

curl -sS -X POST "$API_URL/v1/analyses" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: $TENANT_ID" \
  -H "X-API-Key: $API_KEY" \
  -H "Idempotency-Key: $IDEMP_KEY" \
  -d '{
    "campaign_id": "'"$CAMPAIGN_ID"'",
    "script_id": "'"$SCRIPT_ID"'",
    "asset_id": "'"$ASSET_ID"'",
    "metrics": {"views": 1000, "likes": 100, "comments": 20, "shares": 15}
  }' | tee /tmp/analysis-create.json | jq

export ANALYSIS_ID="$(jq -r '.analysis_id' /tmp/analysis-create.json)"
```

### Step 5 — Poll analysis until completed

```bash
watch -n 3 "curl -sS '$API_URL/v1/analyses/$ANALYSIS_ID' \
  -H 'X-Tenant-Id: $TENANT_ID' \
  -H 'X-API-Key: $API_KEY' \
  | jq '{analysis_id,status,processing_stage,error_message,updated_at,result_schema}'"
```

When `status=completed`, fetch full result:

```bash
curl -sS "$API_URL/v1/analyses/$ANALYSIS_ID" \
  -H "X-Tenant-Id: $TENANT_ID" \
  -H "X-API-Key: $API_KEY" | jq
```

---

## 4) Result Contract (`analysis.v2`)

### Top-level response

```json
{
  "analysis_id": "uuid",
  "tenant_id": "acme",
  "campaign_id": "camp_x",
  "status": "queued|processing|completed|failed|invalid_schema",
  "result_schema": "analysis.v2",
  "result": {"...": "..."},
  "error_message": null
}
```

### `result` structure

```json
{
  "schema_version": "analysis.v2",
  "policy_context": {},
  "expectations": [],
  "evidences": [],
  "matches": [],
  "violations": [],
  "compliance_summary": {
    "status": "approved|partial|rejected",
    "score": 0.0
  },
  "observability": {}
}
```

### Status semantics (`compliance_summary.status`)
- `approved`: required expectations met, no critical violations
- `partial`: partially met, with gaps/non-critical violations
- `rejected`: critical violations or low required coverage

---

## 5) Bulk Analyses

Endpoint:

```txt
POST /v1/analyses/bulk
```

Minimal example:

```bash
curl -sS -X POST "$API_URL/v1/analyses/bulk" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: $TENANT_ID" \
  -H "X-API-Key: $API_KEY" \
  -H "Idempotency-Key: bulk-$(date +%s)" \
  -d '{
    "items": [
      {
        "item_id": "item_001",
        "campaign_id": "'"$CAMPAIGN_ID"'",
        "script_id": "'"$SCRIPT_ID"'",
        "asset_source": {
          "type": "direct_url",
          "url": "https://drive.google.com/uc?export=download&id=VIDEO_ID"
        },
        "metrics": {"views": 1000, "likes": 100, "comments": 20, "shares": 15}
      }
    ]
  }' | tee /tmp/bulk.json | jq
```

---

## 6) Error Handling

Common cases:

- `401 credenciais ausentes`
  - Missing `X-Tenant-Id` or `X-API-Key`
- `403 credenciais invalidas`
  - Wrong tenant/key pair
- `400 Idempotency-Key obrigatorio`
  - Missing `Idempotency-Key` in `POST /v1/analyses`
- `404 campaign_id/script_id/asset_id nao encontrado`
  - Wrong IDs or tenant mismatch
- `500 falha no processamento`
  - Check analysis status and `error_message` in `GET /v1/analyses/{id}`

---

## 7) Health and Docs

- Healthcheck: `GET /api/healthz`
- Interactive docs:
  - Swagger UI: `https://pulse.drafteye.app/api/docs`
  - ReDoc: `https://pulse.drafteye.app/api/redoc`

---

## 8) Notes for Integration

- `Idempotency-Key` should be generated by client per create-analysis request.
- Persist `analysis_id` and poll status until terminal state.
- Treat `result.observability` as internal/debug metadata; primary business contract is `expectations/evidences/matches/violations/compliance_summary`.
