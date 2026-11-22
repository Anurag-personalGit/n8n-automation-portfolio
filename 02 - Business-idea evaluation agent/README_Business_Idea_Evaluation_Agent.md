# Business Idea Evaluation Agent — n8n Multi-Agent System

## 1. Overview

The **Business Idea Evaluation Agent** is a multi-agent AI workflow built in n8n. It evaluates a business idea through five specialized AI roles coordinated by an Orchestrator:

1. Market Analyst
2. Feasibility Checker
3. Money Agent
4. Skeptic
5. Verdict Agent

The Orchestrator receives the user's business idea, coordinates the four specialist analyses, and passes the findings to the Verdict Agent. The Verdict Agent synthesizes the specialist outputs into scores and a final **GO / REWORK / KILL** recommendation.

The workflow is designed to give a founder a structured first-pass assessment of a business idea rather than relying on one general-purpose AI response.

### What it does

- Accepts a business idea through the n8n chat interface.
- Coordinates specialist AI agents through an Orchestrator.
- Evaluates market demand, competition, timing, and market potential.
- Assesses whether a first-time founder can build and operate the idea.
- Reviews the revenue model, unit economics, path to first revenue, and margins.
- Identifies the most significant reasons the idea could fail.
- Synthesizes the four specialist analyses into scores and a final recommendation.

---

## 2. Workflow Architecture

### High-level architecture

```text
                    User
                     |
                     v
        When chat message received
                     |
                     v
               Orchestrator
                     |
        +------------+-------------+----------------+----------------+
        |            |             |                |                |
        v            v             v                v                v
  Market Analyst  Feasibility   Money Agent      Skeptic       Verdict Agent
                   Checker
        |            |             |                |                ^
        +------------+-------------+----------------+----------------+
                                                                      |
                                                         Final synthesis and decision
                                                                      |
                                                                      v
                                                          GO / REWORK / KILL
```

**Important:** The five specialist nodes are connected to the Orchestrator through n8n's AI Tool connections. The Verdict Agent is also exposed as a tool to the Orchestrator; it is not a separate main-flow node. The Orchestrator's system instructions tell it to collect the four specialist outputs and then pass the idea and analyses to the Verdict Agent.

### Shared language model

A single **OpenRouter Chat Model** node is connected to the Orchestrator and all five specialist agents.

| Setting | Configuration in the exported workflow |
|---|---|
| Provider node | OpenRouter Chat Model |
| Model identifier | `nvidia/nemotron-3-super-120b-a12b:free` |
| Model display name | NVIDIA Nemotron 3 Super 120B A12B (Free) |
| Connections | Orchestrator, Market Analyst, Feasibility Checker, Money Agent, Skeptic, Verdict Agent |

The workflow therefore uses the same configured model node for all six AI agents.

---

## 3. Node Architecture and Responsibilities

| Node | n8n node type / role | Responsibility |
|---|---|---|
| **When chat message received** | Chat Trigger | Receives the business idea from the n8n chat interface and starts the workflow. |
| **Orchestrator** | AI Agent | Coordinates the specialist agents, ensures that each specialist is called, collects their responses, and passes the combined assessment to the Verdict Agent. |
| **Market Analyst** | AI Agent Tool | Evaluates customer demand, competition, timing, and rough market size. |
| **Feasibility Checker** | AI Agent Tool | Assesses the operational work, skills, team or licence requirements, constraints, and time to a working version. |
| **Money Agent** | AI Agent Tool | Evaluates the revenue model, unit economics, path to first revenue, and potential margins. |
| **Skeptic** | AI Agent Tool | Identifies the two or three strongest reasons the business idea may fail. |
| **Verdict Agent** | AI Agent Tool | Synthesizes the four specialist analyses into scores, a GO / REWORK / KILL verdict, and a short explanation. |
| **OpenRouter Chat Model** | Chat Model | Supplies the language model to the Orchestrator and all five specialist agents. |

### Connection design

- The Chat Trigger's main output connects to the Orchestrator.
- The Market Analyst, Feasibility Checker, Money Agent, Skeptic, and Verdict Agent connect to the Orchestrator through their `ai_tool` connections.
- The OpenRouter Chat Model connects to the `ai_languageModel` input of the Orchestrator and all five specialist agents.
- The exported workflow does not contain separate memory nodes, databases, web-search nodes, or external research integrations.

---

## 4. Detailed Agent Design and System Instructions

Each agent has a specific responsibility. The specialist prompts deliberately restrict each agent to its assigned evaluation dimension so that the final assessment can combine distinct perspectives.

### 4.1 Orchestrator

**Role:** Coordinates the business idea evaluation panel.

The Orchestrator receives the user's raw business idea and coordinates the four specialist agents before handing their findings to the Verdict Agent.

**System instruction configured in n8n:**

```text
You are the Orchestrator of a business idea evaluation panel. You receive a raw business idea and coordinate four specialist agents to judge it, then hand everything to the Verdict agent.

Your job:
1. Read the incoming idea. If it is one line, restate it in one clear sentence so all specialists judge the same thing.
2. Call each specialist tool exactly once, passing the restated idea: Market Analyst, Feasibility Checker, Money Agent, Skeptic.
3. Do not analyze the idea yourself. You coordinate, you do not judge.
4. Collect all four outputs. If any specialist returns something empty or off-topic, call it once more.
5. Pass the idea plus all four analyses to the Verdict agent and return its final output.

Never skip a specialist. Never invent an analysis a specialist did not return.
```

**Responsibilities:**

1. Receive the business idea from the chat trigger.
2. Restate a short or unclear idea as one clear sentence for consistent evaluation.
3. Call the Market Analyst, Feasibility Checker, Money Agent, and Skeptic.
4. Check that each specialist has returned a useful response.
5. Retry a specialist once if its response is empty or off-topic.
6. Pass the business idea and all four analyses to the Verdict Agent.
7. Return the final verdict.

The Orchestrator is instructed not to perform its own business evaluation or invent missing specialist findings.

### 4.2 Market Analyst

**Role:** Evaluates the market dimension only.

**System instruction configured in n8n:**

```text
You are a Market Analyst on a business idea evaluation panel. You judge ONE dimension only: market.

Given a business idea, assess:
- Real demand: is there a paying, underserved need?
- Competition: who already serves this, and how saturated is it?
- Timing: why now, or why not?
- Rough market size: small niche, or scalable.

Return 4 to 6 tight sentences. End with one line: "Market signal: strong / mixed / weak." Do not comment on money, feasibility, or risk. Stay in your lane.
```

**Evaluation areas:**

- Whether there is a real paying customer need.
- Whether the need is underserved.
- Existing competitors and market saturation.
- Whether the timing is favourable.
- Whether the opportunity is a small niche or potentially scalable.

**Expected response:** Four to six concise sentences, ending with one of these signals:

```text
Market signal: strong
Market signal: mixed
Market signal: weak
```

The Market Analyst is instructed not to evaluate economics, feasibility, or risk.

### 4.3 Feasibility Checker

**Role:** Evaluates whether a first-time founder can build and operate the idea.

**System instruction configured in n8n:**

```text
You are a Feasibility Checker on a business idea evaluation panel. You judge ONE dimension only: can a first-time founder actually build and run this?

Assess:
- Operational lift: what has to happen daily for this to work.
- Skills, team, or licenses required.
- Physical or supply constraints.
- Time to a working version.

Return 4 to 6 tight sentences. End with one line: "Buildable by a first-time founder: yes / with help / no." Do not comment on market, money, or risk.
```

**Evaluation areas:**

- Day-to-day operational requirements.
- Skills and team requirements.
- Licences or specialist capabilities, where applicable.
- Physical, supply, or operational constraints.
- Estimated time to a working version.

**Expected response:** Four to six concise sentences, ending with one of these assessments:

```text
Buildable by a first-time founder: yes
Buildable by a first-time founder: with help
Buildable by a first-time founder: no
```

The Feasibility Checker is instructed not to comment on market, money, or risk.

### 4.4 Money Agent

**Role:** Evaluates the economics of the business idea.

**System instruction configured in n8n:**

```text
You are the Money Agent on a business idea evaluation panel. You judge ONE dimension only: economics.

Assess:
- How this makes money (the actual model).
- Unit economics: rough cost to serve one customer vs what they pay.
- Realistic path to first revenue.
- Whether margins survive at scale.

Return 4 to 6 tight sentences with rough numbers where you can. End with one line: "Money model: healthy / thin / broken." Do not comment on market, feasibility, or risk.
```

**Evaluation areas:**

- How the business generates revenue.
- Approximate cost of serving one customer compared with customer revenue.
- A realistic path to the first sale or revenue.
- Whether the business can maintain margins as it scales.

**Expected response:** Four to six concise sentences, with rough numbers where possible, ending with one of these signals:

```text
Money model: healthy
Money model: thin
Money model: broken
```

The Money Agent is instructed not to evaluate market, feasibility, or risk.

### 4.5 Skeptic

**Role:** Identifies the strongest reasons the idea could fail.

**System instruction configured in n8n:**

```text
You are the Skeptic on a business idea evaluation panel. Your only job is to find the kill-shots.

Name the 2 or 3 reasons this idea most likely fails: the hidden cost, the wrong assumption, the reason 9 of 10 versions of this die. Be specific and blunt, not generic. No praise, no balance.

Return 3 to 5 tight sentences. End with one line: "Biggest kill-shot: <one phrase>."
```

**Evaluation areas:**

- Hidden costs and underestimated effort.
- Incorrect or untested assumptions.
- Reasons customers may not adopt or pay for the offering.
- Major execution or operational weaknesses.
- Other specific failure points relevant to the idea.

**Expected response:** Three to five direct sentences identifying the most serious failure points, ending with:

```text
Biggest kill-shot: <one phrase>.
```

Unlike the other specialists, the Skeptic is explicitly instructed to focus on failure points rather than provide a balanced assessment.

### 4.6 Verdict Agent

**Role:** Synthesizes the specialist analyses into a final decision.

**System instruction configured in n8n:**

```text
You are the Verdict agent. You receive a business idea plus four specialist analyses (Market, Feasibility, Money, Skeptic). You synthesize, you do not re-analyze.

Output exactly:
- Idea: <one line>
- Scores (1 to 10): Market _, Feasibility _, Money _, Risk-adjusted _
- Verdict: GO / REWORK / KILL
- One sentence on why.
- If REWORK: the single change that would most improve it.

Base scores only on what the specialists said. Be decisive.
```

The Verdict Agent receives the original/restated idea and the four specialist outputs. It is instructed to synthesize those findings rather than conduct a separate analysis.

**Required output format:**

```text
Idea: <one-line business idea>

Scores (1 to 10):
Market _
Feasibility _
Money _
Risk-adjusted _

Verdict: GO / REWORK / KILL

<One sentence explaining the decision.>

If REWORK: <The single change that would most improve the idea.>
```

The final recommendation must use one of three outcomes:

| Verdict | Meaning |
|---|---|
| **GO** | The specialist findings support moving forward. |
| **REWORK** | The idea needs a specific change before moving forward. |
| **KILL** | The assessment indicates that the idea should not proceed in its current form. |

The workflow prompt defines the verdict labels and scoring dimensions; it does not define fixed numerical thresholds for choosing between GO, REWORK, and KILL.

---

## 5. End-to-End Execution Flow

### Step 1: Submit a business idea

The **When chat message received** node starts the workflow when a user submits a message through the n8n chat interface.

Example input:

```text
I want to build an AI-powered WhatsApp assistant for real estate agents that automatically follows up with property leads.
```

The chat message is passed to the Orchestrator.

### Step 2: Normalize the idea

The Orchestrator reads the incoming idea. If it is expressed in one line or needs clarification for consistent evaluation, it restates it as one clear sentence.

The same restated idea is passed to each specialist.

### Step 3: Run the specialist evaluations

The Orchestrator calls each of the following tools:

| Specialist | Main question |
|---|---|
| Market Analyst | Is there a real market need, and what does the competitive landscape look like? |
| Feasibility Checker | Can a first-time founder realistically build and operate it? |
| Money Agent | Can the business generate revenue with workable unit economics and margins? |
| Skeptic | What are the strongest reasons this idea could fail? |

Each specialist returns its analysis according to its system instructions.

The Orchestrator is instructed to call each specialist exactly once initially. If a specialist returns an empty or off-topic response, it may call that specialist one more time.

### Step 4: Collect the specialist outputs

The Orchestrator collects the four analyses:

- Market assessment.
- Feasibility assessment.
- Money/economics assessment.
- Skeptical failure-point assessment.

It must not invent an analysis if a specialist has not returned one.

### Step 5: Generate the final verdict

The Orchestrator passes the idea and all four specialist analyses to the Verdict Agent.

The Verdict Agent produces:

- A one-line description of the idea.
- Market score.
- Feasibility score.
- Money score.
- Risk-adjusted score.
- GO / REWORK / KILL verdict.
- One-sentence explanation.
- A single improvement if the verdict is REWORK.

### Step 6: Return the result in chat

The Orchestrator returns the Verdict Agent's final output through the n8n chat interaction.

The workflow does not write the evaluation to a database or spreadsheet in the exported implementation.

---

## 6. Key Features

### 1. Multi-agent evaluation

Uses specialized agents for separate evaluation dimensions instead of asking one agent to handle the entire assessment.

### 2. Orchestrated specialist collaboration

The Orchestrator coordinates the four specialist analyses and passes their findings to the final Verdict Agent.

### 3. Separation of responsibilities

Each specialist has a defined scope. For example, the Market Analyst is instructed not to assess economics, while the Money Agent is restricted to business economics.

### 4. Dedicated failure analysis

The Skeptic focuses on the strongest reasons an idea could fail, helping surface assumptions and weaknesses that may be missed by a purely positive assessment.

### 5. Scored final assessment

The Verdict Agent provides four scores on a 1-to-10 scale: Market, Feasibility, Money, and Risk-adjusted.

### 6. Action-oriented recommendation

The final result is expressed as GO, REWORK, or KILL. If the decision is REWORK, the Verdict Agent must identify the single change that would most improve the idea.

### 7. Conversational input

Users can submit business ideas directly through the n8n chat interface without preparing a fixed input form.

---

## 7. Technologies and Integrations Used

| Technology / Service | Usage |
|---|---|
| **n8n** | Workflow orchestration and AI-agent connections. |
| **n8n Chat Trigger** | Receives the business idea through the chat interface. |
| **n8n AI Agent** | Implements the Orchestrator and specialist agent roles. |
| **n8n AI Agent Tool connections** | Expose the specialist agents and Verdict Agent to the Orchestrator. |
| **OpenRouter** | Provides access to the configured language model. |
| **NVIDIA Nemotron 3 Super 120B A12B (Free)** | Language model configured in the OpenRouter Chat Model node. |

---

## 8. Setup and Configuration

### Prerequisites

Before importing the workflow, ensure that you have:

- An n8n instance with the required AI/LangChain nodes available.
- An OpenRouter account.
- An OpenRouter API key.
- Access to the configured model, or a compatible model available through OpenRouter.

### Step 1: Import the workflow

1. Download the workflow JSON file from this repository.
2. Open your n8n instance.
3. Create or open a workflow.
4. Use the workflow import option to import the JSON file.
5. Review the imported nodes and their connections.

### Step 2: Configure OpenRouter credentials

1. Open the **OpenRouter Chat Model** node.
2. Create or select your own OpenRouter API credentials.
3. Confirm that the configured model is available to your account.
4. Save the credential configuration.

The exported JSON references the original creator's credential record. Credential records are not transferred with the workflow export. Select your own credentials after importing.

### Step 3: Verify the model connections

Confirm that the OpenRouter Chat Model node is connected to:

- Orchestrator
- Market Analyst
- Feasibility Checker
- Money Agent
- Skeptic
- Verdict Agent

Also confirm that all five agent tools are connected to the Orchestrator.

### Step 4: Test the workflow

1. Open the workflow chat interface.
2. Submit a clearly described business idea.
3. Check that the Orchestrator invokes all four specialist agents.
4. Review each specialist's response and confirm that it stays within its assigned scope.
5. Verify that the Verdict Agent receives the four analyses.
6. Review the final scores and GO / REWORK / KILL recommendation.

---

## 9. Testing

The following are suggested test inputs for validating the workflow. They are examples for testing, not recorded execution results.

### Test Idea 1 — AI assistant for real estate agents

```text
I want to build an AI-powered WhatsApp assistant for real estate agents that automatically follows up with property leads.
```

Check that:

- The Market Analyst discusses demand and competition.
- The Feasibility Checker discusses the skills and operations required.
- The Money Agent discusses revenue and unit economics.
- The Skeptic identifies concrete reasons the idea may fail.
- The Verdict Agent returns all four scores and one of the three verdicts.

### Test Idea 2 — Office lunch subscription

```text
I want to start a subscription service delivering healthy office lunches to employees.
```

Use this to check whether the agents consider operational requirements, supply constraints, revenue, and potential failure points.

### Test Idea 3 — Long-video repurposing tool

```text
I want to build an AI tool that converts long YouTube videos into short-form social media content.
```

Use this to check whether the agents distinguish market demand, technical and operational feasibility, economics, and competitive or execution risks.

### Validation checklist

- [ ] Chat Trigger receives the user's idea.
- [ ] Orchestrator receives the chat message.
- [ ] All four specialist tools are invoked.
- [ ] Each specialist follows its assigned evaluation scope.
- [ ] Empty or off-topic specialist responses are handled according to the Orchestrator instructions.
- [ ] Verdict Agent receives the idea and four specialist analyses.
- [ ] All four scores are present and fall within the 1-to-10 scale.
- [ ] The final verdict is GO, REWORK, or KILL.
- [ ] A REWORK verdict includes one recommended change.

---

## 10. Important Considerations and Limitations

- **AI-generated assessment:** The output is an AI-assisted first-pass evaluation, not proof that a business will succeed or fail.
- **Market evidence:** The exported workflow does not include web-search or market-data tools. The Market Analyst's assessment is therefore based on the idea and the model's available knowledge, not live market research performed by this workflow.
- **Financial assumptions:** The Money Agent may provide rough numbers, as instructed, but the workflow does not connect to accounting systems, financial databases, or live pricing sources to validate them.
- **Feasibility assumptions:** The Feasibility Checker provides an assessment based on the submitted idea; it does not inspect the founder's actual skills, team, budget, or operating environment.
- **Risk assessment:** The Skeptic identifies likely failure points but does not perform a formal legal, security, regulatory, or technical risk review.
- **Model dependency:** Output quality, response time, availability, and usage limits depend on the selected OpenRouter model and account.
- **No persistent storage in this workflow:** The exported implementation returns the verdict through chat. It does not save evaluations to Google Sheets, a database, or a CRM.
- **No fixed scoring thresholds:** The prompts request scores from 1 to 10 and a GO / REWORK / KILL verdict, but do not specify numerical cut-offs for each verdict.

Treat the result as decision support. Validate important market, legal, technical, and financial assumptions independently before investing money or committing to a launch.

---

## 11. Future Enhancements

The following are possible extensions. They are **not included in the current exported workflow**.

### 1. Live market research

Connect appropriate web-search or research tools to the Market Analyst so it can examine current competitors, pricing, customer discussions, and market evidence.

### 2. Financial modelling

Add a structured financial model to estimate startup costs, operating expenses, customer acquisition cost, revenue, break-even point, and cash requirements.

### 3. Founder-specific feasibility

Collect information about the founder's budget, skills, available time, team, and location so the Feasibility Checker can make a more context-aware assessment.

### 4. Evidence and confidence

Require agents to distinguish verified information from assumptions and include a confidence level for their conclusions.

### 5. Persistent evaluation history

Store each submitted idea, specialist analysis, score, and verdict in a database or spreadsheet so users can compare evaluations over time.

### 6. Report generation

Generate a downloadable business evaluation report containing the specialist findings, scores, verdict, assumptions, and recommended next steps.

### 7. Follow-up questions

Add a clarification stage so the Orchestrator can ask the user for missing details before sending an incomplete idea to the specialist agents.

### 8. Human review

Add a review or approval step before treating a verdict as a final business decision.

---

## 12. Learning Outcomes

This workflow demonstrates practical implementation of:

- Multi-agent AI architecture in n8n.
- Orchestrator and specialist-agent design.
- Connecting AI agents through n8n AI Tool connections.
- Sharing a language model across multiple AI agents.
- Prompt design and role-based task separation.
- Multi-perspective business idea evaluation.
- AI-assisted scoring and decision synthesis.
- Conversational workflow initiation using the n8n Chat Trigger.
- Designing an extensible AI workflow that can later incorporate research, data storage, and report generation.

The project demonstrates how multiple focused AI agents can collaborate through an orchestrator to produce a structured business assessment from a natural-language idea.

---

## 13. Screenshots

Store screenshots of the actual n8n workflow and its execution in a `screenshots` folder alongside this README.

Suggested folder structure:

```text
02 - Business-idea evaluation agent/
│
├── workflow.json
├── README.md
└── screenshots/
    ├── workflow-overview.png
    └── sample-evaluation.png
```

Add the actual workflow screenshot to the folder and embed it below. Add a sample evaluation screenshot after capturing a successful execution.

### Workflow Overview

![Business Idea Evaluation Agent — Workflow Overview](screenshots/workflow-overview.png)

### Sample Evaluation

![Business Idea Evaluation Agent — Sample Evaluation](screenshots/sample-evaluation.png)

The image paths above are placeholders until the corresponding screenshots are added to the repository.
