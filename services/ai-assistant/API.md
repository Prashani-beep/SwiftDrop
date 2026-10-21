# AI Assistant API Specifications (`/api/v1/ai`)

> **Service:** `services/ai-assistant` | **Owner:** Member 10 | **Status:** Baseline

---

## 1. Chat with Assistant
Processes a user query, invokes relevant backend tools, and returns the grounded answer.

- **Method / Path:** `POST /api/v1/ai/chat`
- **Auth Required:** Optional (Public with rate limit, or Bearer JWT for customer tracking)
- **Headers:** `Content-Type: application/json`

### Request Body
```json
{
  "message": "Where is my parcel SD-20261021-04921?",
  "conversation_id": "conv_01HXYZ99"
}
```

### Success Response (`200 OK`)
```json
{
  "data": {
    "conversation_id": "conv_01HXYZ99",
    "response": "Your parcel SD-20261021-04921 is currently out for delivery in Springfield, IL with our courier. It is estimated to arrive today by 2:00 PM.",
    "tools_used": ["track_parcel_public"]
  }
}
```

### Error Responses
- `400 Bad Request`: Empty message or invalid JSON.
- `429 Too Many Requests`: Rate limit exceeded for AI completions.
