# Modified & Advanced LinkedIn Content Workflow

## 1. Overview

This n8n workflow automates the creation, image generation, human review, and publishing of LinkedIn posts based on an AWS content calendar maintained in Google Sheets.

The workflow runs every day at 12:00 PM. It selects the next eligible content idea, generates a LinkedIn post and an image prompt using an AI Agent, generates an image through the Hugging Face Inference API, and sends the post draft to a reviewer through Gmail. If the post is approved, the workflow combines the generated image with the approval path and publishes the post to LinkedIn. If it is declined, the corresponding Google Sheets row is updated to record the rejection.

## 2. Objectives

- Select the next pending topic from a Google Sheets content calendar.
- Generate structured LinkedIn content using an AI Agent.
- Generate a supporting image using the Hugging Face image-generation API.
- Require human approval before publishing.
- Publish approved posts with an image to LinkedIn.
- Update the content calendar with the approval or rejection status.

## 3. Workflow Architecture

### High-level flow

```text
Schedule Trigger (Daily at 12 PM)
        |
        v
Get All Ideas (Google Sheets)
        |
        v
Filter Untriggered (Status = Pending, Day <= 42)
        |
        v
Sort By Day
        |
        v
Take Next Idea Only
        |
        v
AI Agent <--- OpenRouter Chat Model
    ^    <--- Structured Output Parser
    |
    v
Image generation by Hugging Face
        |
        v
Convert to File
        |
        +------------------------------+
        |                              |
        v                              v
Human in Loop (Gmail)             Merge Image + Approval
        |                              ^
        v                              |
       IF                              |
      /  \                             |
 Approved Declined                     |
    |        |                         |
    v        v                         |
Mark Idea   Update row                 |
As Triggered in sheet                  |
    |                                  |
    +----------------------------------+
                    |
                    v
              Create a post
                (LinkedIn)
```

### Execution sequence

1. The Schedule Trigger starts the workflow every day at 12:00 PM.
2. `Get All Ideas` reads the content calendar from Google Sheets.
3. `Filter Untriggered (Day 1-30)` keeps rows where `Status` is `Pending` and `Day` is less than or equal to `42`.
4. `Sort By Day` sorts the eligible rows by the `Day` field.
5. `Take Next Idea Only` limits processing to one content idea per execution.
6. The AI Agent generates the LinkedIn post text and a corresponding image-generation prompt.
7. The Hugging Face HTTP Request node submits the prompt to the `black-forest-labs/FLUX.1-schnell` model.
8. `Convert to File` converts the returned Base64 image data into a binary file.
9. `Human in Loop` sends the generated post text through Gmail and waits for an approval or rejection.
10. The IF node checks the approval response.
11. On approval, `Mark Idea As Triggered` updates the matching Google Sheets row. The generated image and approved path are then combined by `Merge Image + Approval`.
12. `Create a post` publishes the post text and image to LinkedIn.
13. On rejection, `Update row in sheet` records `Status = Declined` and `Triggered = false`.

## 4. Node-by-Node Responsibilities

| Node | Responsibility |
|---|---|
| Schedule Trigger (Daily at 12 pm) | Starts the workflow daily at 12:00 PM using the cron expression `00 12 * * *`. |
| Get All Ideas | Reads the content ideas from the `AWS 42-Day Content Plan` worksheet in the `AWS LinkedIn Content Automation` spreadsheet. |
| Filter Untriggered (Day 1-30) | Keeps rows where `Status` is `Pending` and `Day` is less than or equal to `42`. Both conditions must match. |
| Sort By Day | Sorts eligible content ideas by `Day`. |
| Take Next Idea Only | Limits the workflow to one content idea per execution. |
| AI Agent | Generates the LinkedIn post and image prompt using the selected row's AWS Service, Category, Topic/Angle, and Key Point. |
| OpenRouter Chat Model | Provides the language model to the AI Agent. The exported workflow specifies `cohere/north-mini-code:free`. |
| Structured Output Parser | Enforces a structured response containing `post` and `prompt` fields. |
| Image generation by Hugging Face | Sends the AI-generated image prompt to the Hugging Face Inference API using `black-forest-labs/FLUX.1-schnell`. |
| Convert to File | Converts the Base64 image response into a binary file for use by the LinkedIn publishing path. |
| Human in Loop | Sends the generated post text through Gmail and waits for a human approval decision. The configured wait limit is 15 minutes. |
| If | Checks whether `data.approved` is `true` or `false`. |
| Mark Idea As Triggered | On approval, updates the matching Google Sheets row with `Triggered = true`, the current timestamp in `Triggered At`, and `Status = Approved`. |
| Update row in sheet | On rejection, updates the matching row with `Triggered = false` and `Status = Declined`. |
| Merge Image + Approval | Combines the generated image data with the approved execution path before LinkedIn publishing. |
| Create a post | Publishes the generated post text and image through the LinkedIn node. |

## 5. AI Content Generation

### Input to the AI Agent

The AI Agent receives dynamic values from the selected Google Sheets row:

```text
AWS Service: {{ $json['AWS Service'] }}
Category: {{ $json['Category'] }}
Topic/Angle: {{ $json['Topic/Angle'] }}
Key Point: {{ $json['Key Point'] }}
Style: Informative
```

### Content-generation requirements

The configured system instructions direct the AI Agent to:

- Start with a strong, curiosity-building hook.
- Use a realistic problem, story, or observation to establish context.
- Explain a clear takeaway using logic and systems thinking.
- Use simple, natural language and avoid unnecessary jargon.
- Generate 3–4 paragraphs, with 2–3 sentences per paragraph.
- End with a relevant discussion question.
- Add 4–6 relevant hashtags.
- Keep the content informative and aligned with the selected topic.

### Structured output

The Structured Output Parser expects the AI Agent to return:

```json
{
  "post": "Generated LinkedIn post text",
  "prompt": "Prompt for generating the supporting image"
}
```

The `post` field is used in the Gmail approval message and LinkedIn post. The `prompt` field is sent to the Hugging Face image-generation API.

## 6. Image Generation

The workflow uses an HTTP Request node to call the Hugging Face router endpoint:

- **API endpoint:** `https://router.huggingface.co/nscale/v1/images/generations`
- **Model:** `black-forest-labs/FLUX.1-schnell`
- **Prompt source:** `{{ $('AI Agent').item.json.output.prompt }}`
- **Output handling:** The response's `data[0].b64_json` value is converted into a binary file by the Convert to File node.

A valid Hugging Face access token is required in the HTTP Authorization header. Configure the token securely in n8n; do not commit a real token to GitHub.

## 7. Human Approval and Publishing Logic

### Approval path

When the reviewer approves the post:

1. The IF node routes the execution through the approved branch.
2. Google Sheets is updated with:
   - `Triggered = true`
   - `Triggered At = current ISO timestamp`
   - `Status = Approved`
3. The generated image and approved path are combined.
4. The LinkedIn node publishes the post with the generated image.

### Rejection path

When the reviewer rejects the post:

- `Triggered` is set to `false`.
- `Status` is set to `Declined`.
- The workflow does not proceed to LinkedIn publishing.

The Gmail Human in Loop node is configured with a 15-minute wait limit. Confirm the intended behavior for executions that reach this timeout.

## 8. Google Sheets Configuration

The workflow uses the following spreadsheet and worksheet:

- **Spreadsheet:** `AWS LinkedIn Content Automation`
- **Worksheet:** `AWS 42-Day Content Plan`

The workflow reads these content fields:

| Field | Purpose |
|---|---|
| Day | Sequence number used to order content ideas. |
| AWS Service | AWS service covered by the post. |
| Category | Content category. |
| Topic/Angle | Main editorial direction. |
| Key Point | Key message to communicate. |
| Status | Determines whether an idea is eligible for processing. |
| Triggered | Tracks whether the idea has been triggered. |
| Triggered At | Records the timestamp when an idea is approved. |

The approved and declined paths update the row by matching the `Day` value.

## 9. Prerequisites and Credentials

Configure the following integrations in n8n before running the workflow:

| Service | Requirement |
|---|---|
| Google Sheets | OAuth2 credentials with access to the content calendar spreadsheet. |
| OpenRouter | API credentials for the configured chat model. |
| Hugging Face | Access token with permission to use the image-generation endpoint/model. |
| Gmail | OAuth2 credentials to send the approval email and wait for the reviewer's response. |
| LinkedIn | Credentials authorized to publish posts for the configured LinkedIn account. |

Also verify that the Google Sheets worksheet contains the expected column names and values, especially `Day` and `Status`.

## 10. Setup and Configuration

1. Import the workflow JSON into n8n.
2. Open each Google Sheets node and select the correct spreadsheet and worksheet.
3. Configure the Google Sheets credentials.
4. Configure the OpenRouter Chat Model credentials and verify model availability.
5. Add the Hugging Face access token securely to the HTTP Request node's authentication/header configuration.
6. Configure the Gmail credentials and review recipient.
7. Configure the LinkedIn credentials and confirm the target account.
8. Verify the schedule, filter conditions, and Google Sheets column mappings.
9. Test the workflow with a non-production or test content row before enabling the daily schedule.

## 11. Important Configuration Notes

- The filter node's displayed name says `Day 1-30`, but its actual condition is `Day <= 42`. The node name should be updated if the 42-day condition is intentional.
- The workflow updates the Google Sheets row to `Approved` before the LinkedIn publishing node runs. If LinkedIn publishing fails after that update, the sheet may show the idea as approved even though the post was not published. Consider adding a separate `Published` status update after successful LinkedIn publishing.
- The Hugging Face HTTP Request node in the exported JSON contains the placeholder `YOUR_HUGGINGFACE_TOKEN`. Replace it with a securely configured credential before execution. Never store a real token in the exported workflow or repository.
- The workflow uses a 15-minute wait limit for human approval. Test timeout behavior so pending content is not unintentionally left in an unclear state.
- The workflow's daily schedule is configured for 12:00 PM. Confirm the n8n instance timezone to ensure it runs at the intended local time.

## 12. Testing Checklist

- [ ] Confirm the Schedule Trigger runs at the intended local time.
- [ ] Confirm Google Sheets returns the expected content rows.
- [ ] Confirm only `Pending` rows with `Day <= 42` pass the filter.
- [ ] Confirm the earliest eligible idea is selected.
- [ ] Confirm the AI Agent returns both `post` and `prompt`.
- [ ] Confirm Hugging Face returns image data successfully.
- [ ] Confirm Convert to File creates a valid binary image.
- [ ] Confirm the Gmail approval email is received.
- [ ] Test the approval path and verify LinkedIn publishing.
- [ ] Test the rejection path and verify the Google Sheets update.
- [ ] Test the approval timeout behavior.
- [ ] Confirm no credentials or access tokens are exposed in the workflow export.

## 13. Technology Stack

- n8n — workflow automation and orchestration
- Google Sheets — content calendar and status tracking
- OpenRouter — LLM access
- Hugging Face Inference API — AI image generation
- Gmail — human approval workflow
- LinkedIn — post publishing
- Structured Output Parser — structured AI response validation
