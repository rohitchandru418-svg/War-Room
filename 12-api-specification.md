# 12 — API Specification

**Base URL:** `/api/v1`  
**Auth:** JWT Bearer token required on all endpoints except `/auth/*`  
**Client isolation:** `client_id` derived from authenticated user's JWT; never accepted as a user-supplied parameter  
**Formats:** JSON request/response; multipart/form-data for file uploads  
**Errors:** Standard error envelope: `{ "error": { "code": "ERROR_CODE", "message": "...", "details": {} } }`

---

## 1. Authentication

### POST `/auth/login`
```json
Request:  { "email": "...", "password": "..." }
Response: { "access_token": "...", "refresh_token": "...", "expires_in": 900, "user": { "id": "...", "role": "...", "display_name": "...", "language_pref": "en" } }
```

### POST `/auth/refresh`
```json
Request:  { "refresh_token": "..." }
Response: { "access_token": "...", "expires_in": 900 }
```

### POST `/auth/logout`
```json
Response: 204 No Content
```

---

## 2. Dashboard

### GET `/dashboard/summary`
Returns all data needed for the dashboard home screen.
```json
Response: {
  "entity_sentiment": [
    {
      "entity_id": "...",
      "entity_name": "TVK",
      "is_primary": true,
      "sentiment_weighted_pct": 64.2,
      "sentiment_raw_pct": 61.0,
      "mention_count": 312,
      "top_contributors": [{"account": "...", "contribution_pct": 4.8}],
      "last_updated_at": "2026-11-05T09:42:00Z",
      "has_enough_data": true,
      "is_stale": false,
      "sparkline": [60, 62, 61, 65, 64]
    }
  ],
  "ally_mood": [
    {
      "entity_id": "...",
      "party_name": "Congress",
      "relationship_tag": "ally",
      "mood": "negative",
      "mention_count": 48,
      "has_alert": true
    }
  ],
  "active_alerts": [...],  // top 3, see Alert structure below
  "rising_stories": [...], // top 5, see Story structure
  "source_health": [
    { "source": "x", "status": "healthy", "last_fetched_at": "..." },
    { "source": "Puthiya Thalaimurai", "status": "delayed", "delayed_since": "..." }
  ]
}
```

### GET `/dashboard/entity/:entity_id/sentiment`
Drill-down: sentiment over time for one entity.
```
Query params: ?window=24h|7d|30d&source_type=all|x|news|epaper
```
```json
Response: {
  "entity_id": "...",
  "window": "24h",
  "sentiment_pct": 64.2,
  "mention_count": 312,
  "is_sampled": false,
  "time_series": [{ "hour": "2026-11-05T08:00:00Z", "positive": 60, "negative": 30, "neutral": 10, "count": 100 }],
  "top_contributors": [...],
  "mentions": [...]  // paginated
}
```

---

## 3. Mentions

### GET `/mentions`
```
Query params:
  entity_id (optional)
  source_type (optional): x_post|news_article|epaper|clipping|youtube
  stance (optional): positive|negative|neutral|sarcastic
  language (optional): ta|en|tanglish|mixed
  story_id (optional)
  date_from, date_to (ISO 8601)
  sort: published_at|influence_score
  page, page_size (default 50)
```
```json
Response: {
  "mentions": [{
    "id": "...",
    "source_type": "x_post",
    "author_name": "...",
    "author_handle": "...",
    "author_verified": true,
    "author_follower_count": 1200000,
    "snippet": "...",
    "published_at": "...",
    "source_url": "...",
    "language": "en",
    "influence_score": 0.87,
    "is_bot_suspected": false,
    "labels": [{
      "entity_id": "...",
      "entity_name": "CM Vijay",
      "stance": "negative",
      "claim_type": "on_record",
      "reason": "Direct criticism of CM Vijay's decision on...",
      "is_corrected": false
    }],
    "outlet_tier": "national_media",
    "outlet_tier_reason": "Verified, 1.2M followers, national media"
  }],
  "total": 312,
  "page": 1,
  "page_size": 50
}
```

### GET `/mentions/:id`
Full mention detail including all labels (current + history) and correction history.

---

## 4. Labels & Corrections

### POST `/mentions/:mention_id/corrections`
```json
Request: {
  "new_stance": "positive",
  "new_claim_type": "satire",
  "new_entity_id": "optional-if-changing",
  "reason": "This is sarcastic support, not negative — context makes it clear"
}
Response: {
  "correction_id": "...",
  "requires_second_review": true,
  "status": "pending_review",
  "message": "Correction submitted — awaiting second review"
}
```

### GET `/corrections/pending-review`
War Room Lead / Admin only. Lists corrections awaiting second review.

### POST `/corrections/:correction_id/review`
War Room Lead / Admin only.
```json
Request: { "decision": "approved|rejected", "reason": "..." }
Response: { "correction_id": "...", "status": "approved|rejected" }
```

---

## 5. Stories

### GET `/stories`
```
Query params: entity_id, source_type, date_from, date_to, response_queue=true, sort=last_mention_at|mention_count|velocity, page, page_size
```
```json
Response: {
  "stories": [{
    "id": "...",
    "title": "...",
    "total_mentions": 1200,
    "outlet_count": 9,
    "x_post_count": 1191,
    "first_mention_at": "...",
    "last_mention_at": "...",
    "is_rising": true,
    "severity": "high",
    "response_queue": false,
    "response_status": "none",
    "campaign_id": null,
    "entity_tags": ["CM Vijay", "TVK"]
  }],
  "total": 42, "page": 1, "page_size": 20
}
```

### GET `/stories/:id`
Full Story detail: header, mentions (paginated), timeline data, response status.

### POST `/stories/:id/merge`
War Room Admin / Lead only.
```json
Request: { "merge_into_story_id": "..." }
Response: { "merged_story_id": "..." }
```

### POST `/stories/:id/split`
War Room Admin / Lead only.
```json
Request: { "mention_ids_for_new_story": ["...", "..."] }
Response: { "new_story_id": "..." }
```

### PATCH `/stories/:id/response`
```json
Request: { "status": "acted_on|not_needed" }
Response: { "story_id": "...", "response_status": "acted_on", "marked_by": "...", "marked_at": "..." }
```

---

## 6. Alerts

### GET `/alerts`
```
Query params: status=active|acknowledged|resolved, alert_type, severity_tier, entity_id, page, page_size
```
```json
Response: {
  "alerts": [{
    "id": "...",
    "alert_type": "influence_trigger",
    "entity_id": "...",
    "entity_name": "CM Vijay",
    "severity_tier": 1,
    "explanation": "A verified national journalist...",
    "status": "active",
    "fired_at": "...",
    "acknowledged_by": null,
    "assigned_to": null,
    "whatsapp_delivered": true,
    "story_id": "..."
  }]
}
```

### GET `/alerts/:id`
Full alert detail including evidence mentions and history.

### POST `/alerts/:id/acknowledge`
```json
Response: { "alert_id": "...", "acknowledged_by": "...", "acknowledged_at": "..." }
```

### POST `/alerts/:id/assign`
```json
Request: { "assign_to_user_id": "..." }
Response: { "alert_id": "...", "assigned_to": "...", "assigned_by": "...", "assigned_at": "..." }
```

---

## 7. Campaigns

### GET `/campaigns`
```
Query params: status=possible|confirmed|dismissed, entity_id, page, page_size
```

### GET `/campaigns/:id`
Full campaign detail: accounts, phrasing, timing, spread timeline.

### POST `/campaigns/:id/confirm`
War Room Lead only.
```json
Request: { "attributed_to_entity_id": "...", "reason": "..." }
Response: { "campaign_id": "...", "attribution_status": "confirmed", "attributed_to": "...", "confirmed_by": "..." }
```

### POST `/campaigns/:id/dismiss`
War Room Lead only.
```json
Request: { "reason": "..." }
Response: { "campaign_id": "...", "attribution_status": "dismissed" }
```

---

## 8. Co-worker

### POST `/coworker/chat`
```json
Request: {
  "question": "ஏன் Congress mentions negative ஆனது?",
  "conversation_id": "optional-for-multi-turn"
}
Response: {
  "answer": "Congress mentions turned negative today because...",
  "answer_language": "ta",
  "citations": [
    { "mention_id": "...", "snippet": "...", "published_at": "...", "author": "..." }
  ],
  "data_note": "Based on 64 posts",
  "conversation_id": "...",
  "latency_ms": 3200
}
```

### POST `/coworker/chat/:interaction_id/rate`
```json
Request: { "rating": "useful|not_useful" }
```

### GET `/coworker/suggestions`
Returns contextual suggested questions based on current dashboard state.
```json
Response: { "suggestions": ["Why did Congress mentions turn negative today?", "..."] }
```

### POST `/coworker/briefs/handover`
Triggers handover draft generation. Returns draft for lead to review.
```json
Response: {
  "draft": "Open alerts: ...\nWhat to watch overnight: ...",
  "generated_at": "...",
  "open_alert_count": 2,
  "needs_review": true
}
```

### POST `/coworker/briefs/cm-summary`
Triggers CM summary generation.
```json
Response: {
  "changes": [
    { "description": "...", "cause": "...", "evidence_mention_ids": ["..."], "needs_ally_review": false }
  ],
  "language": "en",
  "generated_at": "..."
}
```

---

## 9. Uploads

### POST `/uploads/clippings`
`Content-Type: multipart/form-data`
```
Fields:
  file: image file (JPG/PNG/PDF)
  outlet_name: string
  publication_date: date (YYYY-MM-DD)
  page_number: integer
```
```json
Response: {
  "upload_id": "...",
  "status": "processing",
  "estimated_completion_seconds": 30
}
```

### GET `/uploads/:upload_id/status`
```json
Response: {
  "upload_id": "...",
  "status": "processing|complete|check_scan|error",
  "mention_id": "optional-when-complete",
  "story_id": "optional-if-matched",
  "ocr_flagged": false
}
```

---

## 10. Response Queue

### GET `/response-queue`
```json
Response: {
  "items": [{
    "story_id": "...",
    "title": "...",
    "suggested_response_type": "condemn",
    "who_has_reacted": ["Congress", "AIADMK", "The Hindu"],
    "time_since_start_minutes": 47,
    "tvk_response_status": "none",
    "severity_score": 8.4
  }]
}
```

---

## 11. Admin — Configuration

### GET `/admin/config/watch-terms?entity_id=...`
### POST `/admin/config/watch-terms`
```json
Request: { "entity_id": "...", "term": "...", "term_type": "hashtag" }
```
### DELETE `/admin/config/watch-terms/:id`

### GET `/admin/config/tiers`
### POST `/admin/config/tiers/:tier_id/members`
```json
Request: { "platform": "x", "handle_or_id": "@handle", "display_name": "..." }
```
### DELETE `/admin/config/tiers/:tier_id/members/:member_id`

### GET `/admin/config/outlets`
### PATCH `/admin/config/outlets/:id`
```json
Request: { "is_partisan": true, "partisan_note": "...", "tier_id": "...", "is_active": true }
```

### GET `/admin/config/thresholds`
### PATCH `/admin/config/thresholds/:threshold_key`
```json
Request: { "value_numeric": 3.0 }
```

### GET `/admin/config/entities`
### PATCH `/admin/config/entities/:id`
```json
Request: { "relationship_tag": "opposition" }
```

---

## 12. Admin — Health and Monitoring

### GET `/admin/health/sources`
Returns per-source health status (all outlets, X API, e-paper, AI model).

### GET `/admin/health/api-usage`
```json
Response: {
  "x_api": {
    "reads_this_month": 1800000,
    "monthly_cap": 3000000,
    "pct_used": 60.0,
    "alert_level": "normal"
  }
}
```

### GET `/admin/health/whatsapp`
Returns recent WhatsApp delivery log.

### GET `/admin/accuracy/metrics`
```json
Response: {
  "capabilities": [{
    "capability": "label_ta",
    "stage": "assist",
    "current_f1": 0.72,
    "accuracy_bar": 0.75,
    "is_above_bar": false,
    "week_trend": [0.68, 0.70, 0.72]
  }]
}
```

### GET `/admin/accuracy/correction-rates`
Correction rate per 1,000 labels, broken down by language and entity, week over week.

---

## 13. WebSocket Events

**Endpoint:** `wss://[host]/ws` (authenticated via token in query param or header)

### Events pushed by server:

| Event | Payload | Consumer |
|-------|---------|---------|
| `alert.new` | Alert object (Tier 1 and 2) | All team members |
| `alert.acknowledged` | `{ alert_id, acknowledged_by, at }` | All team members |
| `alert.assigned` | `{ alert_id, assigned_to, assigned_by }` | All team members |
| `sentiment.update` | Entity sentiment for all 3 entities | Dashboard clients |
| `story.new` | New Story summary | Dashboard clients |
| `story.rising` | Story velocity update | Dashboard clients |
| `source.health` | Source health change | All clients |
| `presentation_mode.toggle` | `{ active: true/false }` | All clients (esp. big screen) |
| `coworker.answer` | Streaming answer tokens | Requesting client only |
| `upload.status` | Upload processing status | Uploading client |

---

## 14. Error Codes

| Code | HTTP | Meaning |
|------|------|---------|
| `AUTH_REQUIRED` | 401 | No valid token |
| `AUTH_EXPIRED` | 401 | Token expired |
| `FORBIDDEN` | 403 | Insufficient role |
| `NOT_FOUND` | 404 | Resource not found |
| `VALIDATION_ERROR` | 422 | Invalid request body |
| `SECOND_REVIEW_REQUIRED` | 400 | Correction requires second review (use correction endpoint) |
| `CAMPAIGN_NOT_CONFIRMED` | 400 | Cannot name party — confirmation required |
| `SOURCE_UNAVAILABLE` | 503 | Upstream source down |
| `COWORKER_UNAVAILABLE` | 503 | AI model unavailable |
| `RATE_LIMIT` | 429 | Too many requests |
