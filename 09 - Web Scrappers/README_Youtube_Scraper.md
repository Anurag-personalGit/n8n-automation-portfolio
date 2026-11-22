# YouTube Scraper — n8n + Apify + Google Sheets

## Overview

This workflow searches YouTube through an Apify actor and appends video and channel information to Google Sheets. The supplied workflow is currently configured to collect up to 10 results for the query **Retrieval Augmented Generation**.

## Workflow Flow

`Manual Trigger → Edit Fields → Youtube Scraper (Apify) → Append row in sheet (Google Sheets)`

1. **Manual Trigger** starts the workflow when you select **Execute workflow**.
2. **Edit Fields** contains a field named `search`, currently set to `RAG, retrival augmented generation`.
3. **Youtube Scraper** runs the Apify **YouTube Scraper (`streamers/youtube-scraper`)** actor. Its current `searchQueries` input is hardcoded to `Retrieval Augmented Generation`, and `maxResults` is set to `10`.
4. **Append row in sheet** maps the returned video and channel fields into the `RAG` worksheet.

## Technology Stack

- **n8n** — workflow orchestration
- **Apify** — YouTube search and scraping
- **Google Sheets** — storage for video and channel data

## Google Sheets Output

The workflow maps these fields to the **RAG** worksheet in the **YoutubeScraper Sheet** spreadsheet:

| Sheet column | Apify output field |
|---|---|
| Video Title | `title` |
| Link | `url` |
| Views | `viewCount` |
| Likes | `likes` |
| Comments | `commentsCount` |
| Channel Name | `channelName` |
| Channel URL | `channelUrl` |
| Subscribers | `numberOfSubscribers` |

## Configuration

The current Apify actor settings include:

- Search query: `Retrieval Augmented Generation`
- Maximum results: `10`
- Subtitle download and subtitle-related options: disabled
- Shorts results: `0`
- Stream results: `0`

**Important configuration note:** The `search` value in **Edit Fields** is not referenced by the current Apify `searchQueries` input. The actor currently uses the hardcoded query `Retrieval Augmented Generation`. To make the search configurable from **Edit Fields**, update the actor input to reference the field, for example `{{ $json.search }}`, and ensure the value is formatted as the actor expects.

## Prerequisites

1. An n8n instance.
2. An Apify account and a configured Apify credential in n8n.
3. A Google account, access to the destination spreadsheet, and a configured Google Sheets credential in n8n.
4. A destination worksheet named `RAG` with columns matching the output mapping above.

## Run the Workflow

1. Open the workflow in n8n.
2. Review the search query in the Apify node. If you want to change it, update `searchQueries` there; changing only the Edit Fields value will not currently change the actor query.
3. Confirm the Apify and Google Sheets credentials are connected.
4. Select **Execute workflow**.
5. Check the `RAG` worksheet for the appended video records.

## Notes

- The actor may return fewer than the configured maximum when fewer matching results are available.
- Metrics such as views, likes, comments, or subscriber counts may be unavailable for some results.
- Use collected information in accordance with YouTube, Apify, and applicable data-use requirements.
