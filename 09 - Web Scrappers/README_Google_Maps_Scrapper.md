# Google Maps Scraper — n8n + Apify + Google Sheets

## Overview

This workflow collects local business listings from Google Maps using an Apify actor and appends the returned records to a Google Sheets worksheet. It can be used for local business research, prospecting, and lead-list preparation.

## Workflow Flow

`Manual Trigger → Edit Fields → Google Maps Scrapper (Apify) → Append row in sheet (Google Sheets)`

1. **Manual Trigger** starts the workflow when you select **Execute workflow**.
2. **Edit Fields** defines the search inputs:
   - `Search Term`: `Interior Design Company`
   - `Location`: `Pune`
3. **Google Maps Scrapper** runs the Apify **Google Maps Scraper (`compass/crawler-google-places`)** actor with the configured search term and location. The current configuration requests up to 10 places per search.
4. **Append row in sheet** writes each returned item to the `Google_Maps` worksheet.

## Technology Stack

- **n8n** — workflow orchestration
- **Apify** — Google Maps data extraction
- **Google Sheets** — storage for the scraped records

## Google Sheets Output

The workflow maps the following fields:

| Sheet column | Apify output field |
|---|---|
| Title | `title` |
| Address | `address` |
| City | `city` |
| Website | `website` |
| Email | Currently configured as a static/blank value; it is not mapped from the actor output |
| Phone | `phone` |

The destination worksheet is named **Google_Maps** in the **WebScraper Sheets** spreadsheet.

## Configuration

Update the values in the **Edit Fields** node to change the search:

- **Search Term:** the type of business to find (for example, `Interior Design Company`).
- **Location:** the target city or area (for example, `Pune`).

The Apify actor is currently configured with English as the language and a maximum of 10 crawled places per search. Contact scraping and several optional enrichment features are disabled in the actor input.

## Prerequisites

1. An n8n instance.
2. An Apify account and a configured Apify credential in n8n.
3. A Google account, access to the destination spreadsheet, and a configured Google Sheets credential in n8n.
4. A destination worksheet with columns matching the output mapping above.

## Run the Workflow

1. Open the workflow in n8n.
2. Check or update `Search Term` and `Location` in **Edit Fields**.
3. Confirm the Apify and Google Sheets credentials are connected.
4. Select **Execute workflow**.
5. Check the `Google_Maps` worksheet for the appended business records.

## Notes

- Results depend on the search phrase, location, actor behavior, and data available from Google Maps at execution time.
- Some business fields may be missing in the source data.
- The Email column is not currently populated from the Apify result by this workflow.
- Review applicable platform terms, privacy requirements, and local regulations before using collected data for outreach.
