# LinkedIn Job Search Workflow --- n8n

## 1. Overview

This n8n workflow automates the collection and summarization of LinkedIn
job listings based on search criteria submitted through an n8n form. It
retrieves matching job links, extracts job details, uses an AI Agent to
organize each description, and stores the results in Google Sheets.

The workflow is designed to reduce manual job-search effort and make job
details easier to review in one place.

## 2. Objectives

-   Collect job-search criteria through a form.
-   Build a LinkedIn job-search URL using the submitted criteria.
-   Retrieve job listing links from the search results page.
-   Extract key information from individual job postings.
-   Use an AI Agent to summarize each job description in a consistent
    structure.
-   Save the collected job information and AI-generated summary to
    Google Sheets.

## 3. Workflow Architecture

``` text
n8n Form Trigger
      |
      v
Code in JavaScript
      |
      v
HTTP Request (LinkedIn search results)
      |
      v
HTML (extract job listing URLs)
      |
      v
Split Out
      |
      v
Loop Over Items (batch size: 10)
      |
      v
Wait (2 seconds)
      |
      v
HTTP Request1 (retrieve individual job page)
      |
      v
HTML1 (extract job details)
      |
      v
Edit Fields (normalize fields and build apply link)
      |
      v
AI Agent <---- OpenRouter Chat Model
      |
      v
Edit Fields1 (prepare final output)
      |
      v
Google Sheets (append job record)
      |
      +------> Loop Over Items (next job)
```

## 4. Workflow Components

  -----------------------------------------------------------------------
  Node                                Responsibility
  ----------------------------------- -----------------------------------
  On form submission                  Collects the job-search inputs:
                                      keyword, location, experience
                                      level, remote preference, and a PDF
                                      resume upload field.

  Code in JavaScript                  Builds the LinkedIn job-search URL
                                      and converts experience and
                                      workplace preferences into LinkedIn
                                      filter values.

  HTTP Request                        Requests the LinkedIn job-search
                                      results page.

  HTML                                Extracts job-posting URLs from the
                                      search-results HTML.

  Split Out                           Converts the extracted job URL
                                      array into individual items.

  Loop Over Items                     Processes the job listings in
                                      batches of 10.

  Wait                                Adds a 2-second delay before
                                      requesting each individual job
                                      page.

  HTTP Request1                       Retrieves the individual
                                      job-posting page.

  HTML1                               Extracts the job title, company,
                                      location, description, and job
                                      identifier.

  Edit Fields                         Normalizes the extracted
                                      information and constructs the
                                      LinkedIn application URL.

  AI Agent                            Analyzes the job description and
                                      produces a structured summary.

  OpenRouter Chat Model               Provides the language model used by
                                      the AI Agent.

  Edit Fields1                        Prepares the AI output and job
                                      metadata for storage.

  Append row in sheet                 Appends the job information and
                                      AI-generated details to Google
                                      Sheets.
  -----------------------------------------------------------------------

## 5. Input Parameters

The form is configured with these fields:

-   **Key word** --- job title, skill, or search term.
-   **Location** --- target job location.
-   **Experience Level** --- experience category used to filter search
    results.
-   **Remote** --- workplace type (On-Site, Remote, or Hybrid).
-   **Resume** --- PDF file upload field.

The JavaScript node also checks for an `Easy Apply` value. However, that
field is not present in the form configuration shown in the exported
workflow. Add the field to the form if this filter is intended to be
used.

## 6. Job Data Extraction

For each listing, the workflow attempts to extract:

-   Job title
-   Company
-   Location
-   Job description
-   Job ID
-   LinkedIn application link

The application link is constructed from the extracted LinkedIn job
identifier.

## 7. AI-Powered Job Description Analysis

The AI Agent uses the OpenRouter Chat Model to organize the supplied job
description into three sections:

**Role Summary** - A short summary of the role.

**Key Responsibilities** - The main responsibilities described in the
posting.

**Requirements and Skills** - Required technical skills, experience, and
qualifications.

The agent is instructed to use only information in the supplied job
description, avoid inventing details, preserve the original meaning, and
mark missing information as "Not specified".

## 8. Google Sheets Output

The workflow appends one row per processed job to the configured Google
Sheets document.

  Column        Stored information
  ------------- ----------------------------------
  Title         Job title
  Company       Hiring company
  Location      Job location
  Job ID        LinkedIn job identifier
  Apply Link    Constructed LinkedIn job URL
  Job Details   AI-generated structured analysis

This creates a centralized list of collected opportunities for later
review.

## 9. Technologies and Integrations

-   **n8n** --- workflow orchestration and automation.
-   **LinkedIn public job-search pages** --- source of job listing and
    posting information.
-   **HTTP Request and HTML nodes** --- retrieve and extract page
    content.
-   **JavaScript** --- construct search URLs and transform filter
    values.
-   **OpenRouter** --- provides the language model connection.
-   **AI Agent** --- structures and summarizes job descriptions.
-   **Google Sheets** --- stores the collected job records.

## 10. Setup and Configuration

1.  Import the workflow JSON into n8n.
2.  Configure the OpenRouter credential for the OpenRouter Chat Model
    node.
3.  Configure the Google Sheets OAuth credential.
4.  Select or update the destination Google Sheets document and
    worksheet.
5.  Verify that the form field names match the properties read by the
    JavaScript node.
6.  Test the workflow with a small search and confirm that job details
    and AI summaries are written correctly.

## 11. Important Notes and Limitations

-   The exported workflow retrieves LinkedIn pages through HTTP requests
    and extracts information from their HTML. LinkedIn may change its
    page structure, which can cause the CSS selectors to stop working.
-   LinkedIn may restrict automated requests. Use this workflow
    responsibly and ensure your use complies with LinkedIn's applicable
    terms and policies.
-   The workflow includes a 2-second wait and processes listings in
    batches of 10; this does not guarantee that requests will always
    succeed or avoid rate limits.
-   The resume field is present in the form, but the exported workflow
    does not show any downstream node using the uploaded resume. It is
    not currently used for resume matching or job ranking.
-   The form field is labelled `Key word`, while the JavaScript reads
    `Keyword`. Confirm the actual form output property and make the
    names consistent; otherwise, the keyword may not be applied.
-   The JavaScript reads an `Easy Apply` value, but the exported form
    does not define that field. Add it if Easy Apply filtering is
    required.
-   The workflow export has `active: false`; activate it in n8n if it is
    intended to run as an active workflow.

## 12. Expected Outcome

After a successful execution, the destination Google Sheet contains one
record per processed job, including its title, company, location, job
ID, application link, and AI-generated job description analysis.
