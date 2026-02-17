# Recall.ai Integration Status

> **Last Updated:** 2026-02-02 (night session)
> **Status:** Pipeline integration complete, ready for testing

## What Was Done Tonight

### 1. CRITICAL: Pipeline Trigger Fix ✅
**Problem:** Meetings were created from Recall transcripts but never processed.

**Solution:** Added `pipelineQueue.enqueue(meeting.id)` in `recall-webhook.ts`.

**Result:** When `transcript.done` webhook arrives:
1. Transcript is fetched from Recall API
2. Meeting is created in database
3. **Pipeline is triggered** → AI processing begins

### 2. Recording Import Endpoints ✅
New endpoints for manual processing:

```
GET  /api/recordings/importable     - List importable recordings
POST /api/recordings/import         - Batch import recordings
POST /api/recordings/:id/reprocess  - Reprocess single recording
```

### 3. UI Improvements ✅
- Delete buttons for scheduled/failed recordings
- Removed broken "Bekijk alle" link
- Proper cache invalidation

### 4. Bug Fixes ✅
- Bot ID now saved correctly (metadata parsing)
- Microsoft Graph timezone parsing fixed
- Cache invalidation for recordings queries

## How It Works

```
Meeting Recording Flow:
──────────────────────

1. Calendar Sync → Creates ScheduledRecording + Recall Bot

2. Bot joins meeting at scheduled time
   └─ Webhooks: joining_call → in_waiting_room → in_call_recording

3. Meeting ends → Bot leaves
   └─ Webhook: recording_done

4. Transcript processed
   └─ Webhook: transcript.done

5. handleTranscriptReady() triggered:
   ├─ Fetch transcript from Recall API
   ├─ Create Meeting record
   ├─ Link ScheduledRecording → Meeting
   └─ pipelineQueue.enqueue(meeting.id)  ← NEW!

6. Pipeline processes meeting:
   ├─ determine_owner
   ├─ categorize
   ├─ generate_notes (Claude AI)
   ├─ generate_documents
   ├─ upload_sharepoint
   ├─ create_draft (Outlook)
   └─ create_tasks (Productive.io)

7. Meeting appears in dashboard! ✅
```

## Testing Tomorrow

### Automatic Flow Test
1. Schedule a meeting with Teams/Zoom link
2. Let the Schakel bot join and record
3. End the meeting
4. Wait for transcript webhook (~2-5 minutes after meeting ends)
5. Check: Meeting should appear in dashboard with status "pending" then "completed"

### Manual Import Test
If meetings recorded but not processed:

```bash
# List importable recordings
GET /api/recordings/importable

# Import them
POST /api/recordings/import
Body: { "recordingIds": ["id1", "id2"] }
```

### Reprocess Test
If a recording needs reprocessing:

```bash
POST /api/recordings/{id}/reprocess
```

## Webhook Events to Monitor

In Railway logs, look for:

```
[Recall Webhook] Received event: transcript.done for bot xxx
[Recall Webhook] Processing transcript for recording xxx
[Recall Webhook] Created meeting xxx from recording xxx
[Recall Webhook] Enqueueing meeting xxx to pipeline for processing
```

## Known Issues / Edge Cases

1. **Bot ID not saved for old recordings** - Use `/api/recordings/import` to process
2. **Transcript may take time** - Wait 2-5 minutes after meeting ends
3. **Mock mode** - If no RECALL_API_KEY, service runs in mock mode (for development)

## Files Changed

```
server/routes/recall-webhook.ts    # Pipeline trigger + import handling
server/routes/recordings.ts        # Import/reprocess endpoints
server/services/recall.ts          # Metadata parsing fix
server/services/microsoft-graph.ts # Timezone parsing fix
client/src/pages/settings.tsx      # Delete buttons, cache fix
```

## Environment Variables Required

```
RECALL_API_KEY=xxx
RECALL_REGION=eu-central-1
RECALL_WEBHOOK_SECRET=xxx (optional but recommended)
```
