# Schakel AI - API Reference

> **Version:** 1.0
> **Base URL:** `https://map-api.schakel.ai` (production) | `https://map-api-dev.schakel.ai` (staging)
> **Authentication:** Session-based with CSRF tokens

---

## Overview

This document describes the internal API endpoints for the Schakel AI Meeting Automation Platform. The API is primarily consumed by the React frontend application.

### Authentication

All authenticated endpoints require:
1. **Session cookie** (`mac.sid`) - Set after login
2. **CSRF token** (`X-CSRF-Token` header) - Synced via response headers

### Authorization Levels

| Level | Description |
|-------|-------------|
| **Public** | No authentication required |
| **Authenticated** | Logged in user with valid session |
| **Admin** | Tenant administrator role |
| **Superadmin** | Platform administrator (global role) |

### Multi-Tenancy

Most endpoints are tenant-scoped. The active tenant is determined by:
1. User's `activeTenantId` from session
2. Row Level Security (RLS) at database level

---

## Table of Contents

1. [Authentication](#1-authentication)
2. [Meetings](#2-meetings)
3. [Pipeline Processing](#3-pipeline-processing)
4. [Recordings (Recall.ai)](#4-recordings-recallai)
5. [Categories](#5-categories)
6. [People & Users](#6-people--users)
7. [Settings & Configuration](#7-settings--configuration)
8. [Billing (Stripe)](#8-billing-stripe)
9. [Chat / Insights Agent](#9-chat--insights-agent)
10. [Werkruimtes](#10-werkruimtes)
11. [Admin (Superadmin)](#11-admin-superadmin)
12. [Webhooks](#12-webhooks)
13. [GDPR Compliance](#13-gdpr-compliance)

---

## 1. Authentication

### Login
```
POST /api/auth/login
```
**Body:** `{ "username": string, "password": string }`
**Response:** `{ "user": SessionUser }` + Sets session cookie

### Logout
```
POST /api/auth/logout
```
**Response:** `{ "success": true }`

### Get Session
```
GET /api/auth/session
```
**Response:** `{ "authenticated": boolean, "user": SessionUser | null }`

### Switch Tenant
```
POST /api/auth/switch-tenant
```
**Body:** `{ "tenantId": string }`
**Response:** `{ "user": SessionUser }`

### Register (Self-Service)
```
POST /api/auth/register
```
**Body:** `{ "email": string, "password": string, "displayName": string, "companyName": string }`
**Response:** `{ "user": SessionUser }` + Starts trial

### Password Reset Request
```
POST /api/password-reset/request
```
**Body:** `{ "email": string }`
**Response:** `{ "success": true }`

### Password Reset Complete
```
POST /api/password-reset/:token/complete
```
**Body:** `{ "password": string }`
**Response:** `{ "success": true }`

### Accept Invitation
```
POST /api/invitations/:token/accept
```
**Body:** `{ "password": string, "displayName": string }`
**Response:** `{ "user": SessionUser }`

---

## 2. Meetings

### List Meetings
```
GET /api/meetings
```
**Auth:** Required
**Response:** `MeetingLight[]` (excludes transcript/notes for performance, includes `primaryWerkruimteName` and `primaryWerkruimteColor` via JOIN)

### Get Meeting Details
```
GET /api/meetings/:id
```
**Auth:** Required
**Response:** `Meeting` (full details)

### Get Meeting with Pipeline & Logs
```
GET /api/meetings/:id/full
```
**Auth:** Required
**Response:** `{ meeting: Meeting, pipelineSteps: PipelineStep[], logs: ProcessingLog[] }`

### Update Meeting Notes
```
PATCH /api/meetings/:id/notes
```
**Auth:** Required
**Body:** `{ "notesMarkdown": string }`
**Response:** `Meeting`

### Update Meeting Details
```
PATCH /api/meetings/:id/details
```
**Auth:** Required
**Body:** `{ "title"?: string, "clientName"?: string, "primaryWerkruimteId"?: string | null }`
**Response:** `Meeting`

> **Note:** `clientName` is deprecated — use `primaryWerkruimteId` to assign a meeting to a werkruimte.
> Setting `primaryWerkruimteId` replaces all existing `meeting_werkruimtes` entries with the new primary werkruimte (1:1 relationship). Setting to `null` removes all werkruimte assignments.

### Delete Meeting
```
DELETE /api/meetings/:id
```
**Auth:** Required
**Query:** `?deleteFromFireflies=true` (optional)
**Response:** `{ "success": true }`

### Download Notes (PDF/HTML)
```
GET /api/meetings/:id/download
```
**Auth:** Required
**Query:** `?format=pdf|html`
**Response:** File download

### Download Transcript
```
GET /api/meetings/:id/download-transcript
```
**Auth:** Required
**Response:** Text file download

### Manual Ingest
```
POST /api/meetings/ingest
```
**Auth:** Required
**Body:** `{ "title": string, "transcript": string, "date"?: string }`
**Response:** `Meeting`

### Re-upload to SharePoint
```
POST /api/meetings/:id/reupload-sharepoint
```
**Auth:** Required
**Response:** `{ "success": true, "documentUrl": string }`

### Create Outlook Draft
```
POST /api/meetings/:meetingId/create-draft
```
**Auth:** Required
**Body:** `{ "to": string[], "cc"?: string[], "subject": string, "body": string }`
**Response:** `{ "success": true, "webLink": string }`

---

## 3. Pipeline Processing

### Start/Requeue Processing
```
POST /api/meetings/:id/process
```
**Auth:** Required
**Response:** `{ "success": true, "message": string }`

### Retry Full Pipeline
```
POST /api/meetings/:id/retry
```
**Auth:** Required
**Response:** `{ "success": true }`

### Retry Specific Stage
```
POST /api/meetings/:id/retry-stage
```
**Auth:** Required
**Body:** `{ "stage": "categorize" | "summarize" | "sharepoint" | "outlook" | "tasks" }`
**Response:** `{ "success": true }`

### Override Category
```
PATCH /api/meetings/:id/category
```
**Auth:** Required
**Body:** `{ "categoryId": string, "reprocess"?: boolean }`
**Response:** `Meeting`

### Get Pipeline Steps (All Meetings)
```
GET /api/pipeline-steps
```
**Auth:** Required
**Response:** `Record<string, PipelineStep[]>`

### Preview Fireflies Imports
```
GET /api/fireflies/preview
```
**Auth:** Required
**Response:** `{ transcripts: FirefliesTranscript[], alreadyImported: string[] }`

### Import from Fireflies
```
POST /api/fireflies/import
```
**Auth:** Required
**Body:** `{ "transcriptIds": string[] }`
**Response:** `{ "imported": number, "results": ImportResult[] }`

---

## 4. Recordings (Recall.ai)

### List Recordings
```
GET /api/recordings
```
**Auth:** Required
**Response:** `ScheduledRecording[]`

### Get Upcoming Recordings
```
GET /api/recordings/upcoming
```
**Auth:** Required
**Query:** `?hours=48` (default: 48)
**Response:** `ScheduledRecording[]`

### Get Calendar Events with Bot Status
```
GET /api/recordings/calendar-events
```
**Auth:** Required
**Query:** `?days=14` (default: 14)
**Response:** `{ events: CalendarEvent[], withBotCount: number, scheduledButNoBotCount: number }`

### Get Recording History
```
GET /api/recordings/history
```
**Auth:** Required
**Query:** `?limit=50`
**Response:** `ScheduledRecording[]`

### Get Recording Status Summary
```
GET /api/recordings/status
```
**Auth:** Required
**Response:** `{ provider: string, totalRecordings: number, statusCounts: Record<string, number>, ... }`

### Create Recording
```
POST /api/recordings
```
**Auth:** Required
**Body:** `{ "meetingUrl": string, "title": string, "startTime": string }`
**Response:** `ScheduledRecording`

### Delete Recording
```
DELETE /api/recordings/:id
```
**Auth:** Required
**Query:** `?cancelRecallBot=true` (optional)
**Response:** `{ "success": true }`

### Toggle Bot for Calendar Event
```
POST /api/recordings/toggle-bot/:eventId
```
**Auth:** Required
**Response:** `{ "success": true, "action": "added" | "removed", "botId"?: string }`

### Refresh Calendar & Sync Bots
```
POST /api/recordings/refresh
```
**Auth:** Required
**Response:** `{ "success": true, "eventsFound": number, "botsScheduled": number }`

### Schedule Bot for Event
```
POST /api/recordings/schedule-event/:eventId
```
**Auth:** Required
**Response:** `ScheduledRecording`

### Get Importable Recordings
```
GET /api/recordings/importable
```
**Auth:** Required
**Response:** `ScheduledRecording[]` (recordings with transcripts ready to import)

### Import Recordings
```
POST /api/recordings/import
```
**Auth:** Required
**Body:** `{ "recordingIds": string[] }`
**Response:** `{ "imported": number, "results": ImportResult[] }`

### Reprocess Recording
```
POST /api/recordings/:id/reprocess
```
**Auth:** Required
**Response:** `{ "success": true, "meetingId": string }`

---

## 5. Categories

### List Categories
```
GET /api/categories
```
**Auth:** Required
**Response:** `Category[]`

### Create Category
```
POST /api/categories
```
**Auth:** Admin
**Body:** `{ "name": string, "folderPath": string, "description"?: string }`
**Response:** `Category`

### Update Category
```
PATCH /api/categories/:id
```
**Auth:** Admin
**Body:** `{ "name"?: string, "folderPath"?: string, "description"?: string, "isActive"?: boolean }`
**Response:** `Category`

### Delete Category
```
DELETE /api/categories/:id
```
**Auth:** Admin
**Response:** `204 No Content`

### Get Category Prompts
```
GET /api/category-prompts
```
**Auth:** Required
**Response:** `{ categoryId: string, instructions: string }[]`

### Set Category Prompt
```
PUT /api/category-prompts/:categoryId
```
**Auth:** Admin
**Body:** `{ "instructions": string }`
**Response:** `{ "success": true }`

---

## 6. People & Users

### List People (Users + Invitations)
```
GET /api/people
```
**Auth:** Required
**Response:** `PersonView[]`
**Note:** Each person has `id` (userTenant junction ID) and `userId` (actual user ID). When calling werkruimte leden endpoints, use `userId` — NOT `id`.

### Update Meeting Owner Status
```
PATCH /api/people/:userTenantId/meeting-owner
```
**Auth:** Admin
**Body:** `{ "isMeetingOwner": boolean, "ownerPriority"?: number }`
**Response:** `PersonView`

### Bulk Update Priorities
```
PATCH /api/people/priorities
```
**Auth:** Admin
**Body:** `{ "priorities": { userTenantId: string, ownerPriority: number }[] }`
**Response:** `{ "updated": number }`

### Invite User to Tenant
```
POST /api/tenant/users/invite
```
**Auth:** Admin
**Body:** `{ "email": string, "displayName": string, "role": "admin" | "member" }`
**Response:** `{ "success": true, "invitationId"?: string }`

### Get Tenant Users
```
GET /api/tenant/users
```
**Auth:** Admin
**Response:** `TenantUser[]`

### Change User Role
```
PATCH /api/tenant/users/:userId/role
```
**Auth:** Admin
**Body:** `{ "role": "admin" | "member" }`
**Response:** `TenantUser`

### Remove User from Tenant
```
DELETE /api/tenant/users/:userId
```
**Auth:** Admin
**Response:** `{ "success": true }`

---

## 7. Settings & Configuration

### Get Integration Status
```
GET /api/config/status
```
**Auth:** Required
**Response:** `{ integrations: { sharepoint: boolean, productive: boolean, outlook: boolean, ... } }`

### Get System Settings
```
GET /api/system-settings
```
**Auth:** Required
**Response:** `SystemSettings`

### Update System Settings
```
PATCH /api/system-settings
```
**Auth:** Admin
**Body:** `Partial<SystemSettings>`
**Response:** `SystemSettings`

### Get Branding
```
GET /api/branding
```
**Auth:** Required
**Response:** `BrandingConfig`

### Update Branding
```
PATCH /api/branding
```
**Auth:** Admin
**Body:** `Partial<BrandingConfig>`
**Response:** `BrandingConfig`

### Upload Logo
```
POST /api/branding/logo
```
**Auth:** Admin
**Body:** `FormData` with `logo` file
**Response:** `{ "logoUrl": string }`

### Upload Bot Avatar
```
POST /api/config/bot-avatar
```
**Auth:** Admin
**Body:** `FormData` with `avatar` file
**Response:** `{ "avatarUrl": string }`

### Get LLM Config
```
GET /api/llm/config
```
**Auth:** Required
**Response:** `LLMConfig`

### Update LLM Config
```
PATCH /api/llm/config
```
**Auth:** Admin
**Body:** `Partial<LLMConfig>`
**Response:** `LLMConfig`

### OAuth Status
```
GET /api/oauth/status
```
**Auth:** Required
**Response:** `{ microsoft: boolean, fireflies: boolean, productive: boolean }`

### OAuth Authorize
```
GET /api/oauth/authorize/:provider
```
**Auth:** Required
**Response:** `{ "authUrl": string }`

### Disconnect OAuth
```
DELETE /api/oauth/connections/:provider
```
**Auth:** Admin
**Response:** `{ "success": true }`

### Test SharePoint Connection
```
GET /api/config/test-sharepoint
```
**Auth:** Admin
**Response:** `{ "connected": boolean, "siteName"?: string, "error"?: string }`

### Get SharePoint Sites
```
GET /api/config/sharepoint-sites
```
**Auth:** Admin
**Response:** `{ "sites": SharePointSite[] }`

### Get Productive People
```
GET /api/productive/people
```
**Auth:** Required
**Response:** `{ "people": ProductivePerson[], "configured": boolean }`

### Get Productive Projects
```
GET /api/productive/projects
```
**Auth:** Required
**Response:** `ProductiveProject[]`

---

## 8. Billing (Stripe)

### Get Subscription Status
```
GET /api/billing/status
```
**Auth:** Required
**Response:** `{ "tier": string, "status": string, "trialEnd"?: string, "currentPeriodEnd"?: string, ... }`

### Create Checkout Session
```
POST /api/billing/checkout
```
**Auth:** Required
**Body:** `{ "tier": "starter" | "pro", "interval": "monthly" | "annual" }`
**Response:** `{ "url": string }` (Stripe checkout URL)

### Create Customer Portal
```
POST /api/billing/portal
```
**Auth:** Required
**Response:** `{ "url": string }` (Stripe portal URL)

### Get Billing Config (Public)
```
GET /api/billing/config
```
**Auth:** None
**Response:** `{ "trialDays": number, "prices": { starter: {...}, pro: {...} } }`

---

## 9. Chat / Insights Agent

### List Chat Threads
```
GET /api/chat/threads
```
**Auth:** Required
**Response:** `ChatThread[]`

### Create Thread
```
POST /api/chat/threads
```
**Auth:** Required
**Body:** `{ "title"?: string }`
**Response:** `ChatThread`

### Get Thread with Messages
```
GET /api/chat/threads/:threadId
```
**Auth:** Required
**Response:** `{ thread: ChatThread, messages: ChatMessage[] }`

### Send Message (SSE Stream)
```
POST /api/chat/threads/:threadId/messages
```
**Auth:** Required
**Body:** `{ "content": string }`
**Response:** Server-Sent Events stream with AI response

### Delete Thread
```
DELETE /api/chat/threads/:threadId
```
**Auth:** Required
**Response:** `{ "success": true }`

### Get Pending Actions
```
GET /api/chat/threads/:threadId/pending-actions
```
**Auth:** Required
**Response:** `PendingAction[]`

### Approve Action
```
POST /api/chat/pending-actions/:actionId/approve
```
**Auth:** Required
**Response:** `{ "success": true, "result": any }`

### Reject Action
```
POST /api/chat/pending-actions/:actionId/reject
```
**Auth:** Required
**Response:** `{ "success": true }`

---

## 10. Werkruimtes

Werkruimtes (workspaces) are containers for grouping meetings by client, project, or team. Meetings can have a **primary werkruimte** (single FK replacing `clientName`) and can belong to **multiple werkruimtes** via a many-to-many junction table for access control.

### List Werkruimtes
```
GET /api/werkruimtes
```
**Auth:** Required
**Response:** `Werkruimte[]`

### Get Werkruimte
```
GET /api/werkruimtes/:id
```
**Auth:** Required
**Response:** `Werkruimte`

### Create Werkruimte
```
POST /api/werkruimtes
```
**Auth:** Admin
**Body:** `{ "name": string, "type"?: "client" | "project" | "internal" | "general", "color"?: string, "description"?: string }`
**Response:** `Werkruimte` (201)
**Notes:** Subject to tier limits (Starter=0, Professional=10, Enterprise=unlimited). Auto-adds creating user as member.

### Update Werkruimte
```
PATCH /api/werkruimtes/:id
```
**Auth:** Admin
**Body:** `{ "name"?: string, "type"?: string, "color"?: string, "description"?: string }`
**Response:** `Werkruimte`

### Delete Werkruimte
```
DELETE /api/werkruimtes/:id
```
**Auth:** Admin
**Response:** `{ "success": true }`
**Notes:** Cannot delete default werkruimtes.

### List Werkruimte Members
```
GET /api/werkruimtes/:id/leden
```
**Auth:** Required
**Response:** `WerkruimteLid[]`

### Add Member to Werkruimte
```
POST /api/werkruimtes/:id/leden
```
**Auth:** Admin
**Body:** `{ "userId": string }`
**Response:** `WerkruimteLid` (201)

### Remove Member from Werkruimte
```
DELETE /api/werkruimtes/:id/leden/:userId
```
**Auth:** Admin
**Response:** `{ "success": true }`

### List Meetings in Werkruimte
```
GET /api/werkruimtes/:id/meetings
```
**Auth:** Required
**Response:** `MeetingLight[]`

### List Werkruimtes for Meeting
```
GET /api/meetings/:meetingId/werkruimtes
```
**Auth:** Required
**Response:** `Werkruimte[]`

### Assign Meeting to Werkruimte
```
POST /api/meetings/:meetingId/werkruimtes
```
**Auth:** Required
**Body:** `{ "werkruimteId": string }`
**Response:** `{ "success": true }` (201)

### Remove Meeting from Werkruimte
```
DELETE /api/meetings/:meetingId/werkruimtes/:werkruimteId
```
**Auth:** Required
**Response:** `{ "success": true }`

---

## 11. Admin (Superadmin)

### List All Tenants
```
GET /api/admin/tenants
```
**Auth:** Superadmin
**Response:** `TenantWithStats[]`

### Create Tenant
```
POST /api/admin/tenants
```
**Auth:** Superadmin
**Body:** `{ "name": string, "slug": string, "adminEmail": string, "adminDisplayName": string }`
**Response:** `Tenant`

### Update Tenant
```
PUT /api/admin/tenants/:id
```
**Auth:** Superadmin
**Body:** `{ "name"?: string, "slug"?: string }`
**Response:** `Tenant`

### Delete Tenant
```
DELETE /api/admin/tenants/:id
```
**Auth:** Superadmin
**Response:** `{ "success": true }`

### List All Users
```
GET /api/admin/users
```
**Auth:** Superadmin
**Response:** `PlatformUser[]`

### Get LLM Usage Stats
```
GET /api/admin/llm-usage/stats
```
**Auth:** Required
**Response:** `{ totalCost: number, totalTokens: number, byModel: {...} }`

### Get Global Secrets
```
GET /api/admin/global-secrets
```
**Auth:** Superadmin
**Response:** `{ secrets: { key: string, masked: string }[] }`

### Update Global Secrets
```
PATCH /api/admin/global-secrets
```
**Auth:** Superadmin
**Body:** `{ [key: string]: string }`
**Response:** `{ "success": true }`

---

## 12. Webhooks

All webhooks are **public** (no session auth) but verified via signatures.

### Fireflies Webhook
```
POST /api/webhook/fireflies
```
**Headers:** Fireflies signature
**Body:** Fireflies transcription notification
**Response:** `200 OK`

### Recall.ai Webhook
```
POST /api/webhook/recall
```
**Headers:** `X-Recall-Signature`
**Body:** Recall bot status/transcript events
**Response:** `200 OK`

### Stripe Webhook
```
POST /api/webhook/stripe
```
**Headers:** `Stripe-Signature`
**Body:** Stripe subscription events
**Response:** `200 OK`

### Microsoft Calendar Webhook
```
POST /api/webhook/calendar
```
**Body:** Microsoft Graph change notifications
**Response:** `200 OK`

---

## 13. GDPR Compliance

### Export Personal Data
```
GET /api/user/data-export
```
**Auth:** Required
**Response:** JSON file download with all personal data

### Get Data Summary
```
GET /api/user/data-summary
```
**Auth:** Required
**Response:** `{ meetings: number, chatThreads: number, ... }`

### Request Account Deletion
```
POST /api/user/delete-account
```
**Auth:** Required
**Response:** `{ "success": true }` (sends confirmation email)

### Confirm Account Deletion
```
DELETE /api/user/delete-account/confirm
```
**Auth:** Token-based (from email)
**Query:** `?token=...`
**Response:** `{ "success": true }`

---

## Error Responses

All errors follow this format:

```json
{
  "error": "Error message",
  "code": "ERROR_CODE",
  "details": { ... }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `AUTH_UNAUTHENTICATED` | 401 | Not logged in |
| `AUTH_SESSION_EXPIRED` | 401 | Session expired |
| `AUTHZ_FORBIDDEN` | 403 | Not authorized for this action |
| `AUTHZ_NOT_TENANT_MEMBER` | 403 | Not a member of this tenant |
| `VAL_INVALID_INPUT` | 400 | Invalid request body |
| `RES_NOT_FOUND` | 404 | Resource not found |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `SRV_INTERNAL_ERROR` | 500 | Server error |

---

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| `/api/auth/login` | 5 requests / minute |
| `/api/password-reset/request` | 3 requests / hour |
| `/api/chat/*/messages` | 20 requests / minute |
| All other endpoints | 100 requests / minute |

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.3 | 2026-02-15 | Added `userId` vs `id` note on People endpoint; POST leden expects user ID not userTenant ID |
| 1.2 | 2026-02-15 | `setPrimaryWerkruimte` now replaces all junction entries (1:1 enforcement) |
| 1.1 | 2026-02-15 | Added Werkruimtes section (#10), `primaryWerkruimteId` on PATCH meetings/details |
| 1.0 | 2026-02-03 | Initial documentation |
