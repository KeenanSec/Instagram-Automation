# Instagram Automation (n8n)


## Overview

Manually posting to Instagram on a schedule does not scale and is easy to forget. This workflow turns a Google Sheet into a content queue: you fill in rows with an image URL, caption, and scheduled time, and the automation handles the publish and bookkeeping with no manual steps.

Stack: n8n (workflow engine), Google Sheets (content queue + state store), Meta Graph API (Instagram publishing).

## Architecture

![Instagram Automation Workflow](/workflow.png)

The pipeline runs end to end on every scheduled tick. State lives entirely in the sheet, so the workflow is stateless and can be re-run safely.

## How it works

1. **Schedule Trigger** fires on the configured interval (e.g. every 15 minutes).
2. **Get row(s) in sheet** reads the content queue and returns rows due to publish (rows where `status` is `pending` and `scheduled_time` is in the past).
3. **Facebook Graph API (1)** calls `POST /{ig-user-id}/media` with the row's `image_url` and `caption` to create a media container. The response returns a `creation_id`.
4. **Facebook Graph API (2)** calls `POST /{ig-user-id}/media_publish` with that `creation_id` to publish the post and returns the live `ig_media_id`.
5. **Update row in sheet** writes `status = published`, the `published_at` timestamp, and the returned `ig_media_id` back to the same row.

## Google Sheet schema

| Column | Purpose |
|---|---|
| `image_url` | Publicly reachable URL of the image to post |
| `caption` | Post caption text |
| `scheduled_time` | When the post should go out (ISO 8601) |
| `status` | `pending`, `published`, or `failed` |
| `published_at` | Timestamp written back on success |
| `ig_media_id` | Live media ID returned by the Graph API |

Adjust column names to match your actual sheet.

## Prerequisites

- A running n8n instance (self-hosted or cloud).
- An Instagram **Business** or **Creator** account linked to a Facebook Page. Personal IG accounts cannot use the Content Publishing API.
- A Meta (Facebook) Developer App with a long-lived access token carrying the `instagram_basic`, `instagram_content_publish`, and `pages_read_engagement` permissions.
- A Google Cloud service account (or OAuth credential) with access to the target Google Sheet.

## Setup

1. Import `workflow.json` into n8n (Workflows then Import from File).
2. Create the Google Sheets credential in n8n and point the two sheet nodes at your spreadsheet and tab.
3. Create the Meta Graph API credential in n8n and confirm the access token and IG user ID are set on both Graph API nodes.
4. Set the Schedule Trigger interval.
5. Run once manually with a single test row before enabling the schedule.

## Security notes

This handles real credentials and a public posting surface, so it is treated accordingly:

- **No secrets in the repo.** The Meta access token and Google service account key live only in the n8n credential store, never in the workflow or this repo.
- **Sanitized export.** `workflow.json` was exported with credentials stripped (n8n exports credential references, not secret values, but the file was reviewed by hand before commit to confirm nothing leaked).
- **Least privilege.** The Meta token requests only the permissions the publish flow needs, and the Google service account is shared on the single target sheet rather than the whole Drive.
- **Token lifecycle.** Long-lived Meta tokens expire on roughly a 60 day cycle and need rotation; this is documented rather than left to fail silently.

## Limitations and next steps

- No retry or dead-letter handling yet. A failed Graph API call leaves the row in `pending`; adding an error branch that writes `status = failed` with the error message is the next improvement.
- The media container can take time to process before publish is valid. A polling check on container `status_code` between the two Graph API calls would make this more robust.
- Single image posts only. Carousels and Reels use a different container flow.

## Repository structure

```
.
|- workflow.json        # n8n workflow export (credentials stripped)
|- architecture.png  # workflow screenshot
|- README.md
```
