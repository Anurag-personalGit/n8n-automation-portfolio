# Modified LinkedIn Content Workflow

## 1. Overview

This n8n workflow creates a LinkedIn post from a user-provided idea and writing style, generates a supporting image using Hugging Face, requests human approval through Gmail, and publishes the post on LinkedIn.

The workflow is designed to reduce the manual effort involved in drafting and preparing LinkedIn content while keeping a human approval step before publication.

## 2. Workflow Architecture

**Flow:**

1. User submits a post idea and selects a writing style through an n8n form.
2. An AI Agent generates the LinkedIn post and a matching image prompt.
3. OpenRouter provides the language model, while a Structured Output Parser enforces the expected JSON structure.
4. The image prompt is sent to the Hugging Face image-generation API.
5. The generated image is converted from Base64 data into a binary file.
6. Gmail sends the generated post for human approval.
7. After approval, the workflow publishes the post with its image to LinkedIn.

## 3. Key Features

- **Form-based input:** Collects the post idea and preferred writing style.
- **Multiple writing styles:** Supports Funny, Contrary, and Informative.
- **AI-assisted content generation:** Produces a structured post and image prompt.
- **Image generation:** Uses Hugging Face with the `black-forest-labs/FLUX.1-schnell` model.
- **Human-in-the-loop approval:** Sends the generated content through Gmail for review before publishing.
- **LinkedIn publishing:** Publishes the approved post with image media.

## 4. Nodes and Responsibilities

| Node | Responsibility |
|---|---|
| On form submission | Captures the post idea and selected style. |
| AI Agent | Generates the LinkedIn post and image prompt according to the configured writing instructions. |
| OpenRouter Chat Model | Supplies the language model used by the AI Agent. |
| Structured Output Parser | Structures the AI response with `post` and `prompt` fields. |
| Image generation by Hugging Face | Sends the image prompt to the Hugging Face image-generation endpoint. |
| Convert to File | Converts the returned Base64 image data into a binary file. |
| Human in Loop | Sends the generated post to Gmail for approval. |
| Create a post | Publishes the post on LinkedIn with image media. |

## 5. Input Fields

The n8n form collects:

| Field | Purpose |
|---|---|
| Post Idea | Topic or idea to be developed into a LinkedIn post. |
| Style | Desired writing style: Funny, Contrary, or Informative. |

## 6. AI Output Structure

The AI Agent is configured to return a JSON object containing:

- `post` — the generated LinkedIn post.
- `prompt` — a short prompt used to generate the supporting image.

The configured writing instructions specify a clear hook, a short narrative, a practical insight, a closing question, and relevant hashtags.

## 7. Technologies and Integrations

- n8n — workflow orchestration
- OpenRouter — LLM access
- Hugging Face Inference API — image generation
- Gmail — human approval
- LinkedIn — post publishing

## 8. Setup and Configuration

Before running the workflow, configure the required credentials and account access in n8n:

1. **OpenRouter:** Configure the OpenRouter credential and ensure the selected model is available.
2. **Hugging Face:** Add a valid Hugging Face access token to the HTTP Request node's Authorization header. Do not commit the token or export it into a public workflow file.
3. **Gmail:** Configure Gmail credentials and verify the approval recipient.
4. **LinkedIn:** Configure LinkedIn credentials and verify the connected account has permission to publish posts.
5. **Form:** Submit a post idea and select one of the available styles to start the workflow.

## 9. Important Notes

- The workflow uses external services, so their API availability, usage limits, and account permissions can affect execution.
- Keep API tokens and credentials out of the GitHub repository. Review the exported workflow JSON before publishing it and remove any sensitive values.
- Test the complete approval and publishing path with a non-sensitive sample post before using it for regular publishing.
- The approval step is intended to keep a person in control before LinkedIn publication.
