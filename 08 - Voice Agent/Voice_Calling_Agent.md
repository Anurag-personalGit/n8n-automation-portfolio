# Voice Calling Agent — n8n Workflow

## 1. Overview

The **Voice Calling Agent** workflow automates outbound AI-powered phone calls using contact details stored in Google Sheets. It selects records marked as `pending`, initiates calls through the Bolna API, checks the call execution status, retrieves the transcript, generates a structured call summary with an AI Agent, and writes the result back to the corresponding Google Sheets row.

The workflow is designed to process pending contacts and maintain call status, execution ID, and summary information in one central sheet.

## 2. Workflow Architecture

### High-level flow

1. Manually start the workflow.
2. Retrieve rows from Google Sheets where `status` is `pending`.
3. Pass the retrieved records through the Limit node.
4. Send an outbound call request to Bolna using the contact's phone number.
5. Store the returned Bolna execution ID in n8n workflow static data.
6. Retrieve the stored execution ID and request the call execution details from Bolna.
7. Check the returned status and transcript using the If node.
8. If the conditions are not met, wait for 30 seconds and check the execution again.
9. When the If node takes its true branch, send the transcript to the AI Agent.
10. Extract a structured call summary and update the matching Google Sheets row.
11. Return to Google Sheets and look for the next pending record.

### Workflow sequence

`Manual Trigger → Google Sheets (Get Rows) → Limit → Bolna Call API → Store Execution ID → Retrieve Execution ID → Bolna Execution Details → If`

- **True branch:** `AI Agent → Google Sheets (Update Row) → Get Rows`
- **False branch:** `Wait (30 seconds) → Retrieve Execution ID → Bolna Execution Details`

## 3. Key Features

- **Google Sheets-based contact queue:** Reads contact records whose status is `pending`.
- **Automated outbound calling:** Initiates calls through the Bolna API.
- **Execution tracking:** Stores and reuses the Bolna execution ID while checking call progress.
- **Status and transcript checking:** Uses an If node to decide whether to continue to AI processing or wait and check again.
- **Automatic transcript summarization:** Uses an AI Agent to extract a concise summary from the call transcript.
- **Structured AI response:** Uses a Structured Output Parser to enforce a JSON response containing the `summary` field.
- **Automatic sheet updates:** Writes the call status, execution ID, and generated summary to the original contact row.
- **Sequential record processing:** Returns to the Google Sheets read step after updating a row to check for further pending records.

## 4. Nodes and Responsibilities

| Node | Responsibility |
|---|---|
| When clicking ‘Execute workflow’ | Starts the workflow manually. |
| Get row(s) in sheet | Reads rows from the `Calls_24June` sheet where the `status` column equals `pending`. |
| Limit | Controls how many retrieved items continue through the calling flow, according to the node configuration. |
| HTTP Request | Sends a POST request to `https://api.bolna.ai/call` with the Bolna agent ID, recipient phone number, and callback URL. |
| Stored Execution Id | Saves the `execution_id` returned by Bolna in n8n global workflow static data. |
| Execution ID get stored | Reads the saved execution ID from workflow static data for the next API request. |
| Transcript | Calls the Bolna execution-details endpoint using the saved execution ID to retrieve call information, status, and transcript. |
| If | Checks the returned status and transcript against the configured conditions. |
| Wait | Pauses the workflow for 30 seconds before checking the execution again through the false branch. |
| AI Agent | Processes the call transcript and extracts a structured summary. |
| OpenRouter Chat Model | Provides the language model used by the AI Agent. The configured model is `nvidia/nemotron-3-ultra-550b-a55b:free`. |
| Structured Output Parser | Enforces the expected JSON structure containing a `summary` field. |
| Google Gemini Chat Model | Supplies the model connected to the Structured Output Parser for automatic output correction. The configured model is `models/gemini-2.5-flash`. |
| Upadte sheet | Updates the matching Google Sheets row with the call status, execution ID, and AI-generated summary. |

## 5. Google Sheets Structure

The workflow uses the Google spreadsheet **AI Agents Calling sheet sample**, in the `Calls_24June` worksheet.

The exported workflow references these columns:

| Column | Purpose |
|---|---|
| First name | Contact's first name. |
| phone number | Contact's phone number used to initiate the call. |
| status | Determines whether a record is eligible for processing; the initial lookup selects `pending` records. |
| execution ID | Stores the execution ID returned by Bolna. |
| Summary | Stores the summary generated from the call transcript. |

The update node matches the original record using `row_number`. This ensures that the result is written to the corresponding contact row.

## 6. Bolna API Configuration

The workflow uses the Bolna API for outbound calling and execution tracking.

### Initiate call

The HTTP Request node sends a `POST` request to:

`https://api.bolna.ai/call`

The request body contains:

| Parameter | Value / Purpose |
|---|---|
| `agent_id` | Identifies the configured Bolna voice agent. |
| `recipient_phone_number` | Builds the recipient number using the `+91` country code and the phone number from Google Sheets. |
| `callback_url` | Uses the configured Bolna callback endpoint: `https://n8n.coachshwetagupta.com/webhook/bolna-callback`. |

The request uses an HTTP Header Auth credential configured in n8n. The API key itself should remain stored in n8n credentials and must not be committed to GitHub.

### Retrieve call execution details

The Transcript node sends a request to the execution-details endpoint:

`https://api.bolna.ai/executions/{execution_id}`

The execution ID is inserted dynamically from workflow static data.

## 7. AI Processing and Output

After the If node takes its true branch, the AI Agent receives the transcript and is instructed to extract a call summary.

The expected structured response is:

```json
{
  "summary": "Person is interested"
}
```

The example above represents the expected response shape; the actual summary is generated from each call transcript.

The AI Agent uses the OpenRouter Chat Model. Its Structured Output Parser is configured with automatic correction and is connected to a Google Gemini Chat Model.

## 8. End-to-End Execution

1. Start the workflow using **Execute workflow**.
2. The Google Sheets node retrieves records with `status = pending`.
3. The Limit node controls the records passed to the next step.
4. The Bolna API receives the contact's phone number and initiates an outbound call.
5. Bolna returns an execution ID.
6. The workflow stores that ID in global workflow static data.
7. The workflow retrieves the execution details from Bolna.
8. The If node evaluates the returned status and transcript.
9. If the conditions evaluate to false, the Wait node pauses for 30 seconds and the workflow checks the execution details again.
10. If the conditions evaluate to true, the AI Agent extracts the call summary.
11. The Google Sheets update node writes the status, execution ID, and summary to the original row.
12. The workflow returns to the Google Sheets read step to check for another pending record.

## 9. Setup and Configuration

Configure the following before running the workflow:

1. **Google Sheets:** Connect the Google account and select the correct spreadsheet and `Calls_24June` worksheet. Confirm that the expected column names are present.
2. **Bolna API:** Add the required API authentication through an n8n HTTP Header Auth credential.
3. **Bolna agent:** Verify that the configured `agent_id` belongs to the intended Bolna voice agent.
4. **Phone numbers:** Ensure the sheet contains valid phone numbers. The request currently prepends `+91`, so the input values should not already contain that country code.
5. **Callback endpoint:** Confirm that the configured callback URL is valid and that the corresponding callback handling is available.
6. **OpenRouter:** Configure the OpenRouter credential and verify access to the selected model.
7. **Google Gemini:** Configure the Google Gemini credential used by the Structured Output Parser.
8. **Google Sheets update mapping:** Verify that `row_number`, `status`, `execution ID`, and `Summary` map to the intended columns.

## 10. Important Considerations and Limitations

- **Review the If node conditions:** The exported workflow uses an `OR` combinator across the conditions `status = completed`, `status = busy`, `status = dropped`, and `transcript is not empty`. Therefore, any one matching condition sends the execution to the AI Agent. Confirm that this is the intended decision logic, especially for `busy` or `dropped` calls where a usable transcript may not yet be available.
- **Polling interval:** The Wait node is configured for 30 seconds. The workflow repeatedly checks the Bolna execution details through this wait-and-retry path when the If node evaluates to false.
- **Static data:** The execution ID is stored in global workflow static data. Be aware that static data is shared at the workflow level; concurrent executions or multiple calls running in parallel may overwrite the stored ID. This design should be reviewed before enabling concurrent processing.
- **Limit configuration:** Check the Limit node in the n8n editor to confirm the intended number of contacts processed in each pass.
- **Phone number format:** The request adds `+91` to the sheet's phone number. Adjust the expression if contacts from other country codes need to be supported.
- **API and model availability:** Bolna, OpenRouter, and Google Gemini availability, permissions, quotas, and pricing can affect execution.
- **Credential security:** Do not commit API keys, authorization headers, or other secrets to the repository. Keep them in n8n credentials and review exported JSON files before publishing.
- **Call costs and consent:** Outbound calls may incur charges. Only call contacts who have provided the appropriate consent, and follow applicable calling and privacy regulations.
- **Failure handling:** The current workflow does not show a separate error-handling path for failed API requests, authentication errors, or AI processing failures. Consider adding one before using it in production.

## 11. Technologies and Integrations

- **n8n** — workflow automation and orchestration
- **Google Sheets** — contact queue and call-result storage
- **Bolna AI** — outbound voice calls and call execution details
- **OpenRouter** — language model access for transcript summarization
- **Google Gemini** — structured output correction through the parser
