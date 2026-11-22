# n8n Automation Portfolio

A collection of practical workflow automations built with **n8n**, APIs, AI-powered steps, webhooks, and external services. This repository is intended to showcase workflow design, integration patterns, data handling, and automation projects in a format that others can review and learn from.

> **Portfolio status:** The repository contains nine numbered project folders with exported workflow files and project-specific documentation. The contents and setup requirements vary by workflow.

## Workflow Projects

| # | Project | Overview |
|---|---|---|
| 01 | [Google-Based LinkedIn Public-Post Lead Generation](./01%20-%20Google-based%20LinkedIn%20public-post%20lead%20generation/) | Explores finding relevant public LinkedIn posts through Google-based search to support lead-generation research. |
| 02 | [Business-Idea Evaluation Agent](./02%20-%20Business-idea%20evaluation%20agent/) | An AI-assisted workflow for evaluating and organizing business ideas. |
| 03 | [Advanced LinkedIn Content Workflow — Without Image Generation](./03%20-%20Modified%20%26%20Advanced%20LinkedIn%20Content%20W_F%28without%20image%29/) | A content-creation workflow version that excludes the image-generation step. |
| 04 | [Webhook and Supabase Integration](./04%20-%20Webhook%20-%20Supabase%20Integration/) | Demonstrates receiving webhook requests and integrating workflow steps with Supabase. |
| 05 | [Advanced LinkedIn Content Workflow](./05%20-%20Modified%20%26%20Advanced%20LinkedIn%20Content%20W_F/) | An advanced version of the LinkedIn content workflow that includes AI-assisted content and image-generation steps. |
| 06 | [LinkedIn Job Search](./06%20-%20LinkedIn%20Job%20Search%20w_f/) | A workflow project focused on automating parts of job-search research. |
| 07 | [Modified LinkedIn Content Workflow](./07%20-%20Modified%20LinkedIn%20Content%20W_F/) | A modified LinkedIn content workflow that generates content from a form submission and routes it for review and publishing. |
| 08 | [Voice Calling Agent](./08%20-%20Voice%20Agent/) | Automates the processing of voice-call execution data, retrieves call transcripts, uses AI to evaluate the conversation, and updates the corresponding Google Sheets record. |
| 09 | [Web Scrapers](./09%20-%20Web%20Scrappers/) | A collection of scraping workflows for Google Maps, YouTube, and TripAdvisor, using Apify actors and saving the resulting data to Google Sheets. |

## Skills and Concepts Demonstrated

Across these projects, the repository aims to demonstrate practical experience with concepts such as:

- Designing and connecting nodes in n8n.
- Working with webhooks and API requests.
- Integrating external services and data sources.
- Building AI-assisted workflow steps and content-generation processes.
- Processing, validating, and routing workflow data.
- Integrating services such as Supabase, Google Sheets, Gmail, LinkedIn, OpenRouter, and external scraping platforms where applicable.
- Using loops, conditional logic, data transformation, and structured output parsing.
- Testing workflows and documenting configuration requirements.

The exact integrations and requirements differ by project; check the individual workflow and its documentation before setting it up.

## Repository Structure

Each numbered folder contains a separate workflow project. A project folder may include:

```text
workflow-project/
├── workflow.json       # Exported n8n workflow
└── README.md           # Purpose, workflow overview, configuration, and setup notes
```

The actual files may differ between project folders depending on the workflow.

## Using a Workflow

1. Open the project folder and review its README and any setup notes.
2. Download the workflow JSON file from that folder.
3. Import the JSON into your own n8n instance.
4. Configure the required credentials, API keys, environment variables, and integrations in your own environment.
5. Test with sample data before enabling the workflow for real use.

Importing a workflow does **not** provide the credentials or external accounts it depends on. Some node settings, API plans, URLs, database tables, and permissions may also need to be configured for your environment.

## Security and Responsible Use

- Never commit API keys, access tokens, passwords, private webhook URLs, or other secrets.
- Use n8n's credential features or environment variables for sensitive values instead of hardcoding them in workflow nodes.
- Replace personal or confidential data with safe sample data before sharing a workflow.
- Review every exported JSON file before pushing it to a public repository. A workflow export can contain configuration details even when credentials themselves are not exported.
- Use public data and external services in accordance with their terms and applicable privacy requirements.

These workflows are shared for portfolio, learning, and demonstration purposes. Review and test each workflow before using it in a production environment.

## Feedback and Improvements

This portfolio will evolve as the workflows are tested and documented. Future updates may include architecture diagrams, setup instructions, example inputs and outputs, limitations, and testing notes for individual workflows.
