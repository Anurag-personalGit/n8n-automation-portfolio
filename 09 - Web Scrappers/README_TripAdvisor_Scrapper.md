# TripAdvisor Scraper — n8n + Apify + Google Sheets

## Overview

This workflow retrieves hotel listing information from TripAdvisor using an Apify actor and appends the returned records to Google Sheets. The current configuration searches for **JW Marriot Hotel, Pune** and requests hotel results only.

## Workflow Flow

`Manual Trigger → Edit Fields → Tripadvisor scrapper (Apify) → Append row in sheet (Google Sheets)`

1. **Manual Trigger** starts the workflow when you select **Execute workflow**.
2. **Edit Fields** defines the input field `Location`, currently set to `JW Marriot Hotel, Pune`.
3. **Tripadvisor scrapper** runs the Apify **Tripadvisor Scraper (`maxcopell/tripadvisor`)** actor using the `Location` value as its query. It is configured for INR, English, and up to 10 items per query. Hotel results are enabled; restaurants, attractions, vacation rentals, nearby results, price offers, tags, and AI review summaries are disabled.
4. **Append row in sheet** maps the returned fields into the `Trip_advisor` worksheet.

## Technology Stack

- **n8n** — workflow orchestration
- **Apify** — TripAdvisor data extraction
- **Google Sheets** — storage for the scraped records

## Google Sheets Output

The workflow appends these fields to the **Trip_advisor** worksheet in the **WebScraper Sheets** spreadsheet:

| Sheet column | Apify output field |
|---|---|
| Type | `type` |
| Category | `category` |
| Name | `name` |
| Rating | `rating` |
| Phone Number | `phone` |
| Price Range | `priceRange` |
| No. of Reviews | `numberOfReviews` |
| Email | `email` |
| Website | `website` |

The Address field exists in the node's column schema but is marked as removed and is not currently written to the sheet.

## Configuration

Change the `Location` value in **Edit Fields** to search for a different hotel, place, or location supported by the Apify actor. The current actor configuration uses:

- Currency: `INR`
- Language: `en`
- Maximum items per query: `10`
- Hotels: enabled
- Restaurants, attractions, and vacation rentals: disabled

## Prerequisites

1. An n8n instance.
2. An Apify account and a configured Apify credential in n8n.
3. A Google account, access to the destination spreadsheet, and a configured Google Sheets credential in n8n.
4. A destination worksheet named `Trip_advisor` with columns matching the output mapping above.

## Run the Workflow

1. Open the workflow in n8n.
2. Update `Location` in **Edit Fields** if required.
3. Confirm the Apify and Google Sheets credentials are connected.
4. Select **Execute workflow**.
5. Check the `Trip_advisor` worksheet for the appended hotel records.

## Notes

- Results depend on the query, actor behavior, and information available from TripAdvisor at execution time.
- Contact details, ratings, and price information may be missing for some listings.
- The current workflow does not map Address to the Google Sheets output.
- Review applicable platform terms, privacy requirements, and local regulations before using collected data for outreach or other purposes.
