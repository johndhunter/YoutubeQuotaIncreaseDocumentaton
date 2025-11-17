# ContentFlow YouTube API Integration Design & Usage Document

Application: ContentFlow (Blazor Server CMS)  
Requested Daily Upload Target: 50 videos/day  
Primary Use Cases: Managed content publishing workflow + operational analytics

---

## 1. Purpose of YouTube API Usage
ContentFlow uses the YouTube Data API v3 to:
1. Upload user-generated, workflow-approved videos (automated background upload).
2. Maintain curated playlists per Topic and Category.
3. Retrieve lightweight video analytics (views, likes, comments) for performance tracking.
4. Minimize API usage via caching, throttling, quota monitoring, and batching.

No bulk scrapes or content replication. Only videos created and owned by the organization are uploaded.

---

## 2. High-Level Architecture


![ContentFlow Architecture Diagram](assets/diagrams/architecture.png)

---

## 3. Primary Components

| Component | Responsibility | API Calls |
|-----------|----------------|-----------|
| `YouTubeUploadService` | Validated video upload with quota enforcement | `videos.insert` |
| `YouTubeSyncService` | Playlist creation & maintenance + analytics sync | `playlists.insert`, `playlistItems.insert`, `videos.list` |
| `YouTubeQuotaMonitor` | In-memory tracking of estimated quota usage & daily upload cap | (internal only) |
| `InMemoryYouTubeAnalyticsCache` | Reduces repeat `videos.list` calls by caching stats for 6h | `videos.list` (batched) |
| `TokenManagementService` | Secure refresh/access token lifecycle | OAuth token refresh |
| `Blazor Admin UI` | Displays operational status, quota usage, analytics | Reads from DB + cache |

---

## 4. Authentication & Token Handling

Environment modes:
- Development: Google OAuth browser consent (single controlled developer account).
- Staging: Stored refresh token (encrypted at rest), refreshed server-side—no user impersonation.
- Production: Stored refresh token (encrypted at rest), refreshed server-side—no user impersonation.

Safeguards (per environment):
- Single service account / channel identity; no end-user OAuth.
- Refresh token validated before API use; failure blocks dependent operations.
- No token exposed to client-side UI; server-only handling.

---

## 5. Workflow Overview

### 5.1 Video Upload Lifecycle

![ContentFlow Architecture Diagram](assets/diagrams/youtubeintegration.png)

(Estimated cost units match published API documentation.)

### 5.2 Playlist Sync
- Category or Topic triggers creation of unlisted playlist.
- New uploads mapped to Topic → added to playlist if not present.
- Orphaned videos optionally removed (configurable).
- Operations throttled: per-item delay (default 100ms), batch delay (1000ms), category/topic stagger (500ms).

### 5.3 Analytics Refresh
- Batched `videos.list` requests (≤50 IDs per call).
- Smart scheduling: frequency based on last update + number of tasks referencing video.
- Cached for 6 hours to minimize quota expenditure.
- Only view/like/comment counts + publish date stored.

---

## 6. Data Stored From YouTube
| Data | Source | Purpose | Retention |
|------|--------|---------|-----------|
| Video ID | Upload response | Link & playlist management | Permanent |
| Publish Date | `videos.list` | Trend analysis | Permanent |
| View Count | `videos.list` | Performance KPIs | Periodically refreshed |
| Like Count | `videos.list` | Engagement signal | Periodically refreshed |
| Comment Count | `videos.list` | Engagement signal | Periodically refreshed |
| Playlist ID | `playlists.insert` | Organizational grouping | Permanent |

No bulk metadata replication (e.g., channel statistics, subscriber lists, comments bodies, or unrelated videos).

---

## 7. Quota & Usage Controls

| Control | Description |
|---------|-------------|
| Daily Upload Cap (75 hard-coded; target 50 operational) | Enforced before each upload. |
| Quota Pre-Check | `CanPerformOperationAsync("videos.insert", cost)` blocks if projected usage exceeds limit. |
| Upload Cost Configuration | Default 1600 units (configurable via `Contentflow.json`). |
| Degradation Strategy | If quota usage >90%, non-essential syncs (playlist cleanup, analytics refresh) skipped. |
| Near-Reset Guard | Avoid non-critical sync operations within 2 hours of reset. |
| Caching | Prevents re-fetching analytics within 6h window. |
| Batching | Groups video stats into ≤50 ID calls to minimize overhead. |

---

## 8. Privacy & Compliance

- Only organization-owned videos are uploaded.
- No user personal data sent to YouTube.
- API responses limited to necessary operational fields.
- All tokens encrypted and never logged.
- Uploads default to `unlisted` unless policy changes.
- No redistribution of YouTube data externally.

---

## 9. Failure Handling & Resilience

| Scenario | Handling |
|----------|----------|
| Quota exhaustion | Immediate block, log warning, retry next cycle. |
| Token refresh failure | Abort operation, surface admin alert. |
| Upload failure (network) | Automatic retry (max 3 attempts, exponential backoff). |
| Playlist conflicts | Idempotent add/remove with conflict handling. |
| Analytics missing video | Marks task; removes stale cache; no retry spam. |

---

## 10. Administrative Interfaces

1. Quota Dashboard (`YouTubeQuotaDashboard.razor`)
   - Current usage, percentage, remaining units, time until reset.
   - Daily upload count vs cap.

2. Task Upload Progress
   - Real-time SignalR updates (hub: `/youtubeuploads`).

3. Category & Topic Temporal Analytics
   - Does not increase API usage; relies on stored data.

---

## 11. Configuration Summary

ContentFlow uses a multi-layered configuration approach for flexibility across environments:

### Environment-Specific Settings

- appsettings.json: Base configuration
- appsettings.{Environment}.json: Environment overrides (Development, Staging, Production)
- Contentflow.json: External file for production secrets (connection strings, API keys, certificates)
### Key Configuration Sections
 - ConnectionStrings: Database connection (DefaultConnection)
 - ContentFlow: Core options including YouTube background service settings
   - YoutubeUploadBackgroundService: Upload intervals, privacy status, quota limits
   - YoutubeSyncBackgroundService: Sync intervals, retry policies
   - CertificateConfig: Thumbprint for data protection
 - YouTube: API credentials and options
 - Serilog: Logging configuration with MSSqlServer sink
### Production Configuration
 - External config path specified in appsettings.Production.json
 - Certificate-based data protection for key encryption
 - IIS deployment with SSL certificates
 - Network share paths for video uploads and secrets
### Development vs Production
 - Development: Browser OAuth, local file storage for tokens
 - Production: Encrypted token storage, external config files, certificate protection---

---
## 12. Logging & Audit
- Structured logs include task ID & operation type.
- No sensitive tokens or raw API responses logged.
- Upload lifecycle + playlist actions traceable.
- Temporal DB tables preserve change history.

---

## 13. Request Justification for Increased Quota
- Target of 50 uploads/day supports scaled editorial production.
- Each upload is authenticated, organization-originated content.
- Strict quota-aware gating prevents accidental overuse.
- Efficient analytics retrieval (batched + cached) minimizes incremental quota impact.
- No non-essential or exploratory API traffic.

Estimated Daily Units:
- Uploads: 50 × 1600 = 80,000 (requires approved increase beyond default 10,000).
- Analytics & playlist ops (optimized): typically < 2,000 units.
- Total requested quota aligns with planned controlled publishing throughput.

---

## 14. Safeguards Summary
- Hard-coded daily upload ceiling (< requested target).
- Pre-flight quota projection before each high-cost operation.
- Batched reads + 6h caching window.
- Reduced activity near reset window.
- Idempotent playlist synchronization.
- Non-destructive fallback paths on transient failures.

---

## 15. Non-Usage Areas
- No comment threads ingestion.
- No subscriber or channel-wide competitive metrics.
- No third-party redistribution of YouTube data.

---

## 16. Contact & Ownership
- Single channel administrator responsible for token management.
- Internal technical team maintains codebase & monitoring.

---

End of document.
