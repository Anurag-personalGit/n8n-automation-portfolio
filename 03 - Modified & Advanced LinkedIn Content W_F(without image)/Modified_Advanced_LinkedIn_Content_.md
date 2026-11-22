# Modified & Advanced LinkedIn Content Workflow (Without Image)

## 1. Overview

This n8n workflow automates the preparation, review, and publishing of LinkedIn content based on a pre-planned AWS content calendar stored in Google Sheets.

The workflow runs daily at 12:00 PM. It selects the next eligible content idea, generates a LinkedIn post using an AI Agent, sends the draft for human approval through Gmail, and publishes the post to LinkedIn only when it is approved. It then updates the corresponding content-plan row to record the outcome.

The workflow is named **Modified & Advanced LinkedIn Content W/F (without image)**. The active execution path shown in the supplied workflow screenshot does not generate or attach an image.

## 2. Objectives

- Automate daily selection of content ideas from a Google Sheets content calendar.
- Generate structured, topic-aligned LinkedIn posts with an AI model.
- Keep a human approval step before publishing.
- Publish approved content through the LinkedIn node.
- Track approved and declined ideas in Google Sheets to prevent unintended reuse.

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
Human in Loop (Gmail approval)
        |
        v
       IF
      /  \
 Approved  Declined
    |        |
    v        v
Mark Idea   Update row
As Triggered in sheet
    |
    v
Create a post (LinkedIn)
```

### Main execution sequence

1. The Schedule Trigger starts the workflow every day at 12:00 PM.
2. Google Sheets returns the rows from the configured content-plan sheet.
3. The Filter node keeps rows whose `Status` is `Pending` and whose `Day` is less than or equal to `42`.
4. The Sort node orders the eligible ideas by `Day`.
5. The Limit node passes only the next idea to the AI Agent.
6. The AI Agent generates a LinkedIn post and a short image prompt in structured JSON.
7. Gmail sends the generated post to the configured reviewer and waits for an approval or rejection.
8. The IF node checks the approval result.
9. If approved, the workflow marks the idea as approved/triggered in Google Sheets and publishes the post to LinkedIn.
10. If declined, the workflow updates the sheet with `Status = Declined` and `Triggered = false`.

## 4. Node-by-Node Responsibilities

| Node | Responsibility |
|---|---|
| Schedule Trigger (Daily at 12 pm) | Starts the workflow every day at 12:00 PM using the cron expression `00 12 * * *`. |
| Get All Ideas | Reads content ideas from the `AWS 42-Day Content Plan` sheet in the `AWS LinkedIn Content Automation` Google spreadsheet. |
| Filter Untriggered (Day 1-30) | Keeps rows where `Status` equals `Pending` and `Day` is less than or equal to `42`. Both conditions must be true. |
| Sort By Day | Sorts eligible content ideas in ascending order by the `Day` field. |
| Take Next Idea Only | Limits the output to one item so that only the next eligible idea is processed in that run. |
| AI Agent | Uses the AWS service, category, topic/angle, and key point from the selected row to generate the LinkedIn post and image prompt. |
| OpenRouter Chat Model | Supplies the language model to the AI Agent. The exported workflow specifies `cohere/north-mini-code:free`. |
| Structured Output Parser | Enforces a JSON output structure containing `post` and `prompt` fields. |
| Human in Loop | Sends the generated post to the configured Gmail recipient and waits for a human decision. The configured wait limit is 15 minutes. |
| If | Checks whether the approval response has `data.approved = true`. |
| Mark Idea As Triggered | On the approved branch, updates the matching Google Sheets row: `Triggered = true`, `Triggered At = current ISO timestamp`, and `Status = Approved`. |
| Create a post | Publishes the generated post text to the configured LinkedIn account. |
| Update row in sheet | On the declined branch, updates the matching row to `Triggered = false` and `Status = Declined`. |

## 5. AI Agent Configuration

### Input supplied to the AI Agent

The user prompt is populated dynamically from the selected Google Sheets row:

```text
Write a LinkedIn post about the AWS service: {{ $json['AWS Service'] }}
Category: {{ $json['Category'] }}
Topic/Angle: {{ $json['Topic/Angle'] }}
Key Point: {{ $json['Key Point'] }}

Style: Informative.
```

### Content-generation instructions

The system message instructs the AI Agent to:

- Write crisp, informative, systems-driven LinkedIn content.
- Start with a strong, curiosity-building hook.
- Use a realistic story, problem, or insight to establish context.
- Explain a clear takeaway using logic, systems thinking, or first principles.
- Use simple, natural language and avoid unnecessary jargon, clichés, and generic motivational statements.
- Keep the post between three and four paragraphs, with two to three sentences per paragraph.
- End with a relevant open-ended question and add four to six relevant hashtags.
- Use one to three subtle emoticons where appropriate.
- Keep the post under 3,000 characters.
- Stay aligned with the AWS service, topic/angle, and key point provided by the content plan.

### Structured output

The AI Agent is expected to return only one JSON object:

```json
{
  "post": "Generated LinkedIn post text",
  "prompt": "Short visual prompt under 30 words"
}
```

The Structured Output Parser uses this structure to make the generated fields available to downstream nodes.

**Important:** Although the AI Agent still generates a `prompt` field, the supplied workflow screenshot represents the no-image publishing path. The image-generation, file-conversion, and merge nodes present in the exported JSON are not connected to the active execution path described above.

## 6. Human Approval and Publishing Logic

The Gmail Human in Loop node sends the generated post to the configured reviewer with the subject `Approval required`.

The node is configured to wait for a decision for up to 15 minutes.

### If approved

1. The IF node evaluates `{{ $json.data.approved }}`.
2. The approved branch runs `Mark Idea As Triggered`.
3. The matching row is identified using the `Day` value from `Take Next Idea Only`.
4. The row is updated with:
   - `Triggered = true`
   - `Triggered At = current timestamp`
   - `Status = Approved`
5. The LinkedIn `Create a post` node publishes the generated post.

### If declined

The false branch runs `Update row in sheet` and updates the matching row with:

- `Triggered = false`
- `Status = Declined`

A declined idea is therefore not marked as triggered by this branch. However, because the filter selects only rows with `Status = Pending`, a row marked `Declined` will not be selected again by the current filter.

## 7. Google Sheets Structure

The workflow reads from the spreadsheet **AWS LinkedIn Content Automation**, using the worksheet **AWS 42-Day Content Plan**.

The exported node configuration references these columns:

| Column | Purpose |
|---|---|
| Day | Sequence number used to order and identify the content idea. |
| AWS Service | AWS service that the post should discuss. |
| Category | Content category. |
| Topic/Angle | Main editorial angle for the post. |
| Key Point | Key message the post should communicate. |
| Triggered | Indicates whether the idea has been triggered/published. |
| Triggered At | Timestamp recorded on the approved branch. |
| Status | Used to filter pending ideas and record approval/decline outcome. |

The update nodes match rows using the `Day` column. Ensure that `Day` values are unique in the content plan so the intended row is updated.

## 8. Technology Stack and Integrations

- **n8n** — workflow orchestration and automation.
- **Google Sheets** — content calendar and workflow status tracking.
- **OpenRouter** — language-model access through the configured chat model.
- **Cohere North Mini Code (free model configuration)** — model selected in the exported workflow.
- **Structured Output Parser** — validates the expected JSON response structure.
- **Gmail** — sends the draft and collects the human approval decision.
- **LinkedIn** — publishes the approved post.

## 9. Setup and Configuration

Before importing or running the workflow:

1. Import the workflow JSON into your n8n instance.
2. Configure Google Sheets credentials and select the correct spreadsheet and worksheet.
3. Configure the OpenRouter credential for the Chat Model node.
4. Configure Gmail credentials and verify the approval recipient.
5. Configure LinkedIn credentials and verify the target LinkedIn account.
6. Check the Google Sheets column names and ensure the content-plan rows contain the required fields.
7. Review the schedule and activate the workflow only after testing.

Credentials and secret values are not included in this documentation. Reconnect or select your own credentials after importing the workflow.

## 10. Important Configuration Notes and Limitations

- The schedule is configured for **12:00 PM**. Confirm the n8n instance/workflow timezone to ensure it runs at the intended local time.
- The Filter node's displayed name says `Day 1-30`, but its actual condition is `Day <= 42`. The documentation reflects the configured condition, not the node name.
- The filter requires `Status = Pending`; rows with blank status or `Declined` status will not pass.
- The workflow processes only one eligible idea per scheduled execution.
- The Gmail approval wait is configured for 15 minutes. Test the timeout behaviour in your n8n version before relying on it operationally.
- The approved branch updates the sheet before the LinkedIn publishing node runs. If publishing fails after the sheet update, the row may show `Approved` even though the post was not successfully published. Verify the LinkedIn execution result and reconcile the sheet if needed.
- The workflow uses `Day` as the matching column for Google Sheets updates. Duplicate Day values can cause the wrong row or multiple matching rows to be updated, depending on node behaviour.
- The LinkedIn node publishes the `post` field generated by the AI Agent. Review the content before approving it.

### Inactive image-related nodes in the exported JSON

The exported JSON also contains these nodes:

- `Image generation by Hugging face`
- `Convert to File`
- `Merge Image + Approval`

They have no incoming connection from the active workflow path and are not part of the screenshot's no-image execution flow. They are therefore documented as inactive/orphaned nodes in this export, not as working image-generation functionality. The workflow should be checked in n8n before assuming these nodes are used.

## 11. Testing Checklist

- [ ] Confirm the schedule trigger runs at the intended time.
- [ ] Confirm Google Sheets returns the expected content-plan rows.
- [ ] Confirm only `Pending` rows with `Day <= 42` pass the filter.
- [ ] Confirm ideas are sorted by `Day` and only one item is selected.
- [ ] Confirm the AI Agent returns valid JSON with both `post` and `prompt`.
- [ ] Confirm Gmail sends the generated post and receives an approval decision.
- [ ] Test the approved branch and verify the Google Sheets status update.
- [ ] Verify the LinkedIn post is published successfully.
- [ ] Test the declined branch and verify the row is marked `Declined`.
- [ ] Verify the next scheduled execution selects the next eligible pending idea.

## 12. Possible Improvements

- Add explicit error handling for Google Sheets, Gmail, OpenRouter, and LinkedIn failures.
- Update the sheet only after successful LinkedIn publication, or add a separate `Publishing`/`Published` status to distinguish approval from successful publishing.
- Store the LinkedIn post URL and publication timestamp in Google Sheets.
- Add a retry or manual recovery path for failed LinkedIn publishing.
- Rename the filter node to reflect its actual `Day <= 42` condition.
- Consider adding a separate status for approval timeout so it is distinguishable from an explicit rejection.

## 13. Learning Outcomes

This workflow demonstrates how to combine scheduled automation, spreadsheet-driven task selection, AI-generated content, structured model output, human-in-the-loop approval, conditional branching, and social-media publishing in n8n.

It also demonstrates the importance of maintaining workflow state in an external data source so that scheduled executions can select and process content ideas in a controlled sequence.
