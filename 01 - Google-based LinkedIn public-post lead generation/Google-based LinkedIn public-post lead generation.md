# Google-Based LinkedIn Public-Post Lead Generation

## 1. Overview

The **Google-Based LinkedIn Public-Post Lead Generation** workflow is an AI-assisted lead discovery and qualification automation built using n8n.

It accepts a user's search requirement through the n8n chat interface, generates Google search queries focused on public LinkedIn posts, retrieves matching search results through SerpAPI, and uses an AI agent to identify posts that indicate genuine customer intent for website and web-development services.

The workflow extracts available lead details, structures the information, and appends the results to a Google Sheets spreadsheet for review.

The primary objective is to identify **potential customers looking to purchase or outsource website development services**, while distinguishing them from job seekers, recruiters hiring internal employees, and developers or agencies promoting their own services.

### What the workflow does

- Accepts a lead-search request through the n8n chat trigger.
- Uses an AI agent to generate Google search queries based on the user's search intent.
- Searches Google for public LinkedIn posts using SerpAPI.
- Extracts LinkedIn post URLs, titles, and search-result snippets.
- Removes duplicate post URLs.
- Evaluates each result for customer/project buying intent using an AI agent.
- Produces structured lead information using an output parser.
- Appends the extracted information to Google Sheets.

---

## 2. Workflow Architecture

### High-level execution flow

```text
User Search Request
        |
        v
When Chat Message Received
        |
        v
AI Agent - Search Query Generation
        |
        |  OpenRouter Chat Model
        v
Code in JavaScript
(Process and deduplicate queries)
        |
        v
Loop Over Items
        |
        v
Google Search - SerpAPI
        |
        v
URL Converter
(Extract public LinkedIn posts
and remove duplicate URLs)
        |
        v
Loop Over Items
(Batch size: 10)
        |
        v
AI Agent 2 - Lead Qualification
        |
        |  OpenRouter Chat Model
        |  Structured Output Parser
        v
Code in JavaScript 2
(Map structured lead fields)
        |
        v
Append Row in Google Sheets
```

### AI model connections

The workflow uses OpenRouter Chat Model nodes for both AI agents.

| AI Agent | Model | Responsibility |
|---|---|---|
| AI Agent | NVIDIA Nemotron 3 Super 120B A12B (Free) through OpenRouter | Generates Google search queries from the user's request. |
| AI Agent 2 | NVIDIA Nemotron 3 Super 120B A12B (Free) through OpenRouter | Evaluates search results and classifies potential customer leads. |

The second AI agent is also connected to a **Structured Output Parser**, which enforces the expected JSON output structure.

---

## 3. Node Architecture and Responsibilities

| Node | Responsibility |
|---|---|
| **When chat message received** | Starts the workflow and receives the user's search request through the n8n chat interface. |
| **AI Agent** | Converts the user's search intent into a Google search query targeting public LinkedIn posts that indicate website-development buying intent. |
| **OpenRouter Chat Model** | Supplies the language model used by the first AI agent. |
| **Code in JavaScript** | Splits the generated output into individual non-empty queries and removes duplicate queries. |
| **Loop Over Items** | Processes the generated search queries one at a time. |
| **Google search** | Uses SerpAPI to execute Google searches. The configured request retrieves up to 100 organic results and restricts the returned fields to position, title, link, and snippet. |
| **URL Converter** | Extracts results whose URLs contain `linkedin.com/posts/`, removes query parameters and trailing slashes, and eliminates duplicate post URLs. |
| **Loop Over Items1** | Processes the extracted LinkedIn post results in batches of 10. |
| **AI Agent2** | Analyses each post's available title and snippet to determine whether it represents a potential customer lead. |
| **OpenRouter Chat Model1** | Supplies the language model used by the lead-qualification agent. |
| **Structured Output Parser** | Enforces a structured JSON response containing the lead classification and extracted fields. |
| **Code in JavaScript2** | Parses the AI output and maps the extracted fields to the Google Sheets column names. |
| **Append row in sheet** | Appends each processed result to the configured Google Sheets worksheet. |

---

## 4. Detailed Workflow Execution

### Step 1: Receive the search request

The workflow begins with the **When chat message received** trigger.

The user enters a search request describing the type of leads they want to discover.

Example:

```text
Find LinkedIn posts from people or businesses looking for website development services.
```

The chat input is passed to the first AI agent.

### Step 2: Generate Google search queries

The first **AI Agent** acts as a search-query generation agent.

It converts the user's request into a Google search query focused on public LinkedIn posts.

The configured system instructions include buying-intent phrases and variations such as:

- Website developer looking for
- Need a website
- Website required
- Website redesign
- Web development project
- Looking for a web-development agency
- WordPress developer required
- React developer required
- Shopify website development

The instructions also exclude employment-related phrases, including job searches, internships, resumes, and open-to-work posts.

The agent is instructed to return only the final Google search query.

The agent uses the **OpenRouter Chat Model** with the NVIDIA Nemotron 3 Super 120B A12B (Free) model.

### Step 3: Process the generated queries

The **Code in JavaScript** node:

1. Reads the AI agent's output.
2. Splits the output into separate lines.
3. Removes empty lines.
4. Excludes lines containing safety-classification text.
5. Removes duplicate queries.
6. Creates one n8n item for each remaining query.

The **Loop Over Items** node processes these queries individually.

### Step 4: Search Google using SerpAPI

The **Google search** node uses SerpAPI to execute the generated query.

Configured search settings:

| Setting | Configuration |
|---|---|
| Search provider | SerpAPI |
| Search engine | Google |
| Results requested | Up to 100 |
| Search start position | 0 |
| Execution mode | Synchronous (`async: false`) |
| Returned fields | Position, title, link, snippet |

The search is intended to discover public LinkedIn posts rather than LinkedIn profiles.

### Step 5: Extract LinkedIn post URLs and remove duplicates

The **URL Converter** JavaScript node processes the organic search results.

Its responsibilities are:

- Reads the `organic_results` returned by SerpAPI.
- Keeps results whose URLs contain `linkedin.com/posts/`.
- Removes URL query parameters.
- Removes trailing slashes.
- Tracks previously encountered URLs.
- Skips duplicate LinkedIn posts.
- Extracts the post URL, title, snippet, search position, and source.

Each unique post is passed to the next loop.

### Step 6: Process posts in batches

The second **Loop Over Items1** node processes the extracted posts.

Its configured batch size is **10 items**.

Each batch is passed to the lead-qualification AI agent for analysis.

### Step 7: Qualify potential customer leads

The second **AI Agent**, named `AI Agent2`, acts as a B2B lead-qualification agent for a web-development company.

It receives three inputs for each search result:

- `post_url`
- `title`
- `snippet`

The agent determines whether the result demonstrates genuine buying intent for website or web-development services.

#### Customer leads that can qualify

Examples include posts indicating a requirement for:

- A new business website.
- Website development.
- Website redesign or repair.
- Website maintenance or improvement.
- WordPress, React, PHP, or Shopify development.
- E-commerce website development.
- An external web-development agency.
- A developer for a specific website project.

#### Results that should not qualify

The agent is instructed not to classify the following as customer leads:

- People looking for developer jobs or internships.
- Companies recruiting developers for internal teams.
- Developers looking for clients.
- Freelancers advertising their services.
- Agencies promoting their own services.
- Posts without reasonable evidence of customer/project buying intent.

The agent must use only information supported by the supplied title and snippet. It must not invent names, companies, technologies, or requirements.

The second agent also uses the OpenRouter Chat Model.

### Step 8: Structure the AI response

The **Structured Output Parser** is connected to the lead-qualification agent.

It enforces the following JSON structure:

```json
{
  "is_lead": true,
  "lead_type": "",
  "person_name": "",
  "company_name": "",
  "requirement": "",
  "technology": "",
  "post_url": "",
  "relevance": "High",
  "reason": ""
}
```

The `is_lead` field is a Boolean value, and `relevance` must be one of:

- High
- Medium
- Low

### Step 9: Map the extracted information

The **Code in JavaScript2** node reads the structured AI output and maps its fields to the column names expected by Google Sheets.

It handles the AI output whether it arrives as a JSON string or an object.

It also provides empty-string or default values for missing fields.

### Step 10: Store the results in Google Sheets

The final **Append row in sheet** node appends the mapped information to the configured Google Sheets document.

The configured worksheet is named **Extracted Data**.

The workflow maps the following information into the spreadsheet:

| Google Sheets Column | Description |
|---|---|
| `is_lead` | Whether the result is classified as a potential customer lead (`true` or `false`). |
| `Person` | Person's name, when available. |
| `Company` | Company name, when available. |
| `Lead Type` | Type of website-development requirement. |
| `Requirement` | Extracted customer requirement. |
| `Technology` | Technology mentioned in the search result, when available. |
| `LinkedIn Post` | Public LinkedIn post URL. |
| `Relevance` | Lead relevance: High, Medium, or Low. |
| `Reason` | Explanation supporting the lead classification. |

**Important:** The current workflow appends the processed results to Google Sheets regardless of whether `is_lead` is `true` or `false`. The classification is stored in the spreadsheet so that results can be reviewed and filtered later.

---

## 5. Key Features

### 1. AI-powered search query generation

Uses an AI agent to convert a natural-language search request into a Google search query containing relevant website-development buying-intent phrases.

### 2. Public LinkedIn post discovery

Uses Google search through SerpAPI to discover publicly indexed LinkedIn posts.

### 3. Duplicate post elimination

Uses JavaScript to normalize LinkedIn post URLs and remove duplicate results before qualification.

### 4. AI-based lead qualification

Uses a separate AI agent to distinguish potential customers from job seekers, recruiters, developers advertising services, and other irrelevant results.

### 5. Structured AI output

Uses the Structured Output Parser to produce consistent JSON fields for downstream processing.

### 6. Batch processing

Processes extracted posts in batches of 10 through the second Loop Over Items node.

### 7. Automated Google Sheets integration

Stores extracted lead details and classification results in Google Sheets for further review and follow-up.

---

## 6. Technologies and Integrations Used

| Technology / Service | Usage |
|---|---|
| **n8n** | Workflow orchestration and automation. |
| **AI Agent** | Search-query generation and lead qualification. |
| **OpenRouter** | Access to the NVIDIA Nemotron 3 Super 120B A12B (Free) language model. |
| **SerpAPI** | Google search results retrieval. |
| **Google Search** | Discovery of publicly indexed LinkedIn posts. |
| **JavaScript** | Query processing, URL extraction, duplicate removal, JSON parsing, and field mapping. |
| **Structured Output Parser** | Enforces the lead-qualification JSON structure. |
| **Google Sheets** | Stores extracted lead information and qualification results. |

---

## 7. Setup and Configuration

### Prerequisites

Before importing the workflow, ensure that you have:

- An n8n instance.
- An OpenRouter API account and API key.
- A SerpAPI account and API key.
- A Google account.
- A Google Sheets document with a worksheet named `Extracted Data`.

### Step 1: Import the workflow

1. Download the workflow JSON file from this repository.
2. Open your n8n instance.
3. Create or open a workflow.
4. Select the workflow import option.
5. Import the downloaded JSON file.

### Step 2: Configure OpenRouter

1. Open both OpenRouter Chat Model nodes.
2. Create or select your OpenRouter credentials.
3. Configure the NVIDIA Nemotron 3 Super 120B A12B (Free) model, or select another compatible model if required.
4. Verify that both AI agents can access the configured model.

### Step 3: Configure SerpAPI

1. Open the Google search node.
2. Create or select your SerpAPI credentials.
3. Verify that the API connection is working.

### Step 4: Configure Google Sheets

1. Open the Append row in sheet node.
2. Connect your Google account using Google Sheets OAuth2 credentials.
3. Select your own spreadsheet.
4. Select the `Extracted Data` worksheet.
5. Ensure that the spreadsheet contains columns matching the field mappings in the workflow.

### Step 5: Test the workflow

1. Open the workflow chat interface.
2. Enter a search request.
3. Execute the workflow.
4. Check the generated search query.
5. Verify the SerpAPI search results.
6. Inspect the extracted LinkedIn post URLs.
7. Review the AI lead classifications.
8. Verify that the results are appended to Google Sheets.

**Note:** Imported workflows may contain references to credentials and spreadsheet resources from the original n8n instance. These references are not portable credentials. Configure your own credentials and spreadsheet after importing.

---

## 8. Screenshots

Add screenshots of the actual n8n workflow and its execution to this section.

Suggested repository structure:

```text
01 - Google-based LinkedIn public-post lead generation/
│
├── workflow.json
├── README.md
│
└── screenshots/
    ├── workflow-overview.png
    ├── lead-qualification.png
    └── google-sheets-output.png
```

Embed the screenshots in this README using relative paths:

### Workflow Overview

![Workflow Overview](screenshots/workflow-overview.png)

### Lead Qualification

![Lead Qualification](screenshots/lead-qualification.png)

### Google Sheets Output

![Google Sheets Output](screenshots/google-sheets-output.png)

Replace these example image files with your actual screenshots. If a screenshot is not available, you can leave that image out until it is added.

---

## 9. Important Considerations and Limitations

- **Publicly indexed content:** The workflow searches Google for publicly indexed LinkedIn posts. It does not guarantee discovery of every LinkedIn post.
- **Search-result information:** The qualification agent receives the post title and search-result snippet, along with the post URL. The current workflow does not contain a separate node that opens and retrieves the complete LinkedIn post.
- **AI classification:** Lead classification depends on the information available in the search result and the AI model's interpretation. Results should be reviewed before contacting potential customers.
- **Data availability:** Person name, company, technology, and requirement may be empty when the search-result information does not provide sufficient evidence.
- **Duplicate handling:** Duplicate URLs are removed within the URL Converter execution using a JavaScript Set.
- **Batch size:** The lead-qualification loop is configured to process 10 items per batch.
- **Spreadsheet storage:** Both qualified and unqualified results are appended. The `is_lead` field can be used to filter potential customer leads.
- **API usage:** SerpAPI and OpenRouter usage may be subject to account limits, pricing, and model availability. Check the applicable plans before running large searches.

---

## 10. Future Enhancements

The current workflow can be extended with additional functionality:

1. **Full post retrieval:** Add a suitable data-retrieval step to obtain more post context where permitted and technically supported.
2. **Lead filtering:** Add an IF or Filter node to separate qualified leads from unqualified results before storage.
3. **Lead scoring:** Introduce a scoring mechanism based on buying intent, requirement clarity, and relevance.
4. **Contact enrichment:** Add an enrichment step using appropriate data sources to obtain additional business information.
5. **CRM integration:** Send qualified leads to a CRM for tracking and follow-up.
6. **Notifications:** Notify the sales team when high-relevance leads are discovered.
7. **Scheduled execution:** Add a schedule trigger to run lead discovery periodically without requiring a manual chat request.
8. **Improved search coverage:** Refine search queries and result-handling logic to improve discovery and reduce irrelevant results.

These are potential enhancements and are **not part of the current workflow implementation**.

---

## 11. Learning Outcomes

This workflow demonstrates practical implementation of:

- AI agent integration in n8n.
- Natural-language-to-search-query generation.
- Google search automation using SerpAPI.
- Processing and deduplication of search results using JavaScript.
- AI-based B2B lead qualification.
- Structured JSON generation and parsing.
- Loop-based and batch processing.
- Google Sheets integration for automated data storage.
- Multi-step automation combining AI models, external APIs, and business applications.

The workflow demonstrates how AI-assisted search and qualification can reduce manual effort in discovering and organizing potential website-development customers.
