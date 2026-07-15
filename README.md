# Elastic Agent Builder + Gemini Enterprise Hands-on Lab

Welcome to the **Talk to Your Data** hands-on lab. Today, we are breaking down AI silos by integrating Elastic Agent Builder with Google's Gemini Enterprise using the open A2A (Agent-to-Agent) protocol.

## The Objective
You will build a specialized "eCommerce Data Analyst" Agent in Elastic that has direct access to your data. Instead of moving petabytes of data to an LLM, you will expose this Agent via the A2A protocol and connect it directly to the Gemini Enterprise UI.

### Architecture Overview
```mermaid
flowchart LR
    U(("User<br/>Asks business questions"))
    G["Gemini Enterprise<br/>Master UI & Orchestrator"]
    EA["Elastic AI Agent<br/>Domain Expert & Tools"]
    ES[("Elasticsearch<br/>kibana_sample_data_ecommerce")]
    
    U -->|"Natural Language"| G
    G <-->|"A2A Protocol"| EA
    EA -->|"API Key & ES|QL"| ES
```

---

## Part 1: Elastic Environment Setup

**Facilitator:** Elastic Team

In this section, we will prepare the Elastic environment by spinning up an Elastic Cloud trial, loading our sample dataset, and ensuring our core AI integrations are configured.

### Step 1: Start an Elastic Cloud Free Trial
1. Navigate to the [Elastic Cloud Trial Page](https://www.elastic.co/cloud/cloud-trial-overview) and click **Start free trial**.
2. Sign up with your business email or Google/Microsoft account.
3. Once inside the Cloud Console, click **Create deployment**.
4. Configure your environment details:
   - **Cloud Provider:** Select **Google Cloud (GCP)**.
   - **Deployment Name:** e.g., `A2A-Lab-Environment`.
5. Click **Create Deployment** and wait a few minutes. Save the generated `elastic` username and password.

### Step 2: Load Sample eCommerce Data
1. From the Kibana home page, search for **Sample Data** in the top search bar.
2. Find the **eCommerce orders** dataset and click **Add data**.
3. Verify the data is loaded by checking the pre-built dashboards.

---

## Part 2: Building the Elastic Agent

**Facilitator:** Elastic Team

### Step 1: Generate an API Key
Agent and platform external (Gemini) need access to read the index and execute MCP.
Open Kibana **Dev Tools** and run:

```json
POST /_security/api_key
{
  "name": "a2a-ecommerce-api-key",
  "expiration": "30d",
  "role_descriptors": {
    "mcp-access": {
      "cluster": ["monitor_inference"],
      "indices": [
        {
          "names": ["kibana_sample_data_ecommerce", "*"],
          "privileges": ["read", "view_index_metadata"]
        }
      ],
      "applications": [
        {
          "application": "kibana-.kibana",
          "privileges": ["feature_agentBuilder.read", "feature_actions.read"],
          "resources": ["space:default"]
        }
      ]
    }
  }
}
```
> **Note:** Copy the `encoded` value from the response. This is your API Key!

### Step 2: Create a Custom ES|QL Tool
Let's build an ES|QL tool to query our eCommerce data.

<details>
<summary><b>Option A: Using Kibana UI</b></summary>

1. Navigate to **Agent Builder > Tools**.
2. Click **Create tool** and select **ES|QL query**.
3. ID: `esql-revenue-by-region`.
4. Paste the following query:
   ```esql
   FROM kibana_sample_data_ecommerce 
   | WHERE order_date >= ?startTime 
   | STATS total_revenue = SUM(taxful_total_price), total_orders = COUNT(*) BY geoip.continent_name 
   | SORT total_revenue DESC 
   | LIMIT ?limit
   ```
5. Configure variables: `startTime` (Date) and `limit` (Integer). Save.
</details>

<details>
<summary><b>Option B: Using Dev Tools API</b></summary>

```json
POST kbn:/api/agent_builder/tools
{
  "id": "esql-revenue-by-region",
  "type": "esql",
  "description": "An ES|QL tool to analyze eCommerce total revenue and order counts grouped by continent, filtered by a start time.",
  "tags": ["ecommerce", "sales", "esql"],
  "configuration": {
    "query": "FROM kibana_sample_data_ecommerce | WHERE order_date >= ?startTime | STATS total_revenue = SUM(taxful_total_price), total_orders = COUNT(*) BY geoip.continent_name | SORT total_revenue DESC | LIMIT ?limit",
    "params": {
      "startTime": {
        "type": "date",
        "description": "Start time for the analysis in ISO format (e.g., 2026-01-01T00:00:00Z)"
      },
      "limit": {
        "type": "integer",
        "description": "Maximum number of results to return"
      }
    }
  }
}
```
</details>

### Step 3: Create a Custom Agent Skill
Skills define behavioral guidelines for presenting financial reports.

<details>
<summary><b>Option A: Using Kibana UI</b></summary>

1. Go to **Agent Builder > Skills**.
2. Click **Create Skill**.
3. Name: `eCommerce Reporting Guidelines`.
4. Add formatting instructions (see API payload below for the text) and Save.
</details>

<details>
<summary><b>Option B: Using Dev Tools API</b></summary>

```json
POST kbn:/api/agent_builder/skills
{
  "id": "ecommerce-presentation-skill",
  "name": "eCommerce Reporting Guidelines",
  "description": "Rules for presenting eCommerce financial and operational data.",
  "content": "When answering questions about eCommerce sales:\n1. Always format 'total_revenue' in USD currency (e.g., $1,500.25).\n2. Mention the 'total_orders' to give context on volume.\n3. Present the regional breakdown in a clean bulleted list or markdown table.\n4. Provide a brief 1-2 sentence business insight summarizing the top-performing region."
}
```
</details>

### Step 4: Create the Agent & Assign Tools

<details>
<summary><b>Option A: Using Kibana UI</b></summary>

1. Go to **Agent Builder > Agents** and click **Create Agent**.
2. Name: `eCommerce Analyst`.
3. Provide system instructions defining its role.
4. Add Tools: Check `esql-revenue-by-region` and all `platform.core.*` utilities. Save.
</details>

<details>
<summary><b>Option B: Using Dev Tools API</b></summary>

```json
POST kbn:/api/agent_builder/agents
{
  "id": "elastic-ecommerce-agent",
  "name": "eCommerce Analyst",
  "description": "Hi! I am your eCommerce data analyst. Ask me about our sales revenue and regional performance.",
  "labels": ["ecommerce", "data-analyst", "esql"],
  "avatar_color": "#F04E98",
  "avatar_symbol": "EA",
  "configuration": {
    "instructions": "You are an expert eCommerce data analyst. You help users understand sales performance from the kibana_sample_data_ecommerce dataset. Use your ES|QL tools to fetch accurate data and apply your reporting skills to present it.",
    "tools": [
      {
        "tool_ids": [
          "esql-revenue-by-region",
          "platform.core.search",
          "platform.core.get_document_by_id",
          "platform.core.execute_esql",
          "platform.core.generate_esql",
          "platform.core.get_index_mapping",
          "platform.core.list_indices",
          "platform.core.generate_workflow",
          "platform.core.execute_workflow",
          "platform.core.list_workflow_executions",
          "platform.core.get_workflow_execution_status",
          "platform.core.resume_workflow_execution"
        ]
      }
    ]
  }
}
```
</details>

### Step 5: Assign the Skill via Kibana UI
1. Navigate to the **Agent Builder > Agents** tab.
2. Select and open the newly created **"eCommerce Analyst"** agent card.
3. Scroll down to the **Skills** panel.
4. Click **Add Skill**.
5. Pick the **"eCommerce Reporting Guidelines"** skill created in Step 3.
6. Save the changes.

### Step 6: Verify & Export A2A Configuration
In **Dev Tools**, run this query to fetch your fully assembled A2A Agent Card:

```json
GET kbn:/api/agent_builder/a2a/elastic-ecommerce-agent.json
```
Copy the JSON response payload and save it locally as `elastic-ecommerce-agent.json`.

### Step 7: Test the Agent in Kibana UI
Before integrating with Gemini, let's test it internally.
1. Open the **eCommerce Analyst** agent page.
2. In the **Test your agent** chat panel, ask:
   > *"What is our total revenue and order count broken down by continent since January 1st, 2026? Please present it according to your reporting guidelines."*

---

## Part 3: Gemini Enterprise Integration

**Facilitator:** Google Team

### Step 1: Google Cloud Marketplace
1. Log in to your **Google Cloud Console**.
2. Navigate to the Google Cloud Marketplace.
3. Search for **Elastic AI Agent** and click **Enable / Configure**.

### Step 2: Configure A2A Connection
1. In the configuration form, input the **Agent Card URL** / upload the `elastic-ecommerce-agent.json` file.
2. Input the **Kibana API Key** for authentication.
3. Save and Sync the configuration.

### Step 3: Verification
1. Open your **Gemini Enterprise** chat interface.
2. Check the available agents/tools list. You should now see **eCommerce Analyst** available to be invoked.

---

## Part 4: Test Your Work

### Scenario 1: Natural Language Analytics
In Gemini, ask:
> *"Ask the eCommerce Analyst Agent: What is the top-selling clothing category from last week's data?"*

### Scenario 2: Multi-Agent Handoff
Follow up with a creative task:
> *"Based on the top-selling items you just found, please draft a promotional marketing email to send to our customers."*

---

## Part 5: Actionable Workflows (HITL)

**Facilitator:** Joint

Take your Agent from a "Data Reader" to an "Action Taker" by triggering a Human-in-the-Loop (HITL) workflow directly from Gemini.

### Step 1: Create a Marketing Approval Workflow in Kibana

<details>
<summary><b>Option A: Using Kibana UI</b></summary>

1. Navigate to **Management > Workflows**.
2. Click **Create Workflow** and switch to the **YAML** tab.
3. Paste the following definition and click **Save** and **Enable**:
```yaml
version: "1"
name: Marketing Campaign Approval
description: A HITL workflow that pauses for approval before sending a marketing email.
triggers:
  - type: manual
inputs:
  email_draft:
    type: string
    description: The generated email draft to be reviewed
  audience_email:
    type: string
    description: The email address to send the campaign to
steps:
  - id: ask_approval
    name: Wait for Approval
    type: waitForInput
    with:
      message: "Do you approve this marketing email draft to be sent to {{ inputs.audience_email }}?"
      schema:
        type: object
        properties:
          approved:
            type: boolean
  - id: send_email
    name: Send Email
    type: email
    connector-id: "elastic-cloud-email"
    with:
      to: ["{{ inputs.audience_email }}"]
      subject: "Exclusive Marketing Offer"
      message: "{{ inputs.email_draft }}"
    if: "${{ steps.ask_approval.output.approved == true }}"
  - id: reject_log
    name: Log Rejection
    type: console
    with:
      message: "REJECTED! Email draft was not approved."
    if: "${{ steps.ask_approval.output.approved == false }}"
```
</details>

<details>
<summary><b>Option B: Using Dev Tools API</b></summary>

```json
POST kbn:/api/workflows
{
  "name": "Marketing Campaign Approval",
  "description": "A HITL workflow that pauses for approval before sending a marketing email.",
  "enabled": true,
  "workflow": "version: \"1\"\nname: Marketing Campaign Approval\ndescription: A HITL workflow that pauses for approval before sending a marketing email.\ntriggers:\n  - type: manual\ninputs:\n  email_draft:\n    type: string\n    description: The generated email draft to be reviewed\n  audience_email:\n    type: string\n    description: The email address to send the campaign to\nsteps:\n  - id: ask_approval\n    name: Wait for Approval\n    type: waitForInput\n    with:\n      message: \"Do you approve this marketing email draft to be sent to {{ inputs.audience_email }}?\"\n      schema:\n        type: object\n        properties:\n          approved:\n            type: boolean\n  - id: send_email\n    name: Send Email\n    type: email\n    connector-id: \"elastic-cloud-email\"\n    with:\n      to: [\"{{ inputs.audience_email }}\"]\n      subject: \"Exclusive Marketing Offer\"\n      message: \"{{ inputs.email_draft }}\"\n    if: \"${{ steps.ask_approval.output.approved == true }}\"\n  - id: reject_log\n    name: Log Rejection\n    type: console\n    with:\n      message: \"REJECTED! Email draft was not approved.\"\n    if: \"${{ steps.ask_approval.output.approved == false }}\""
}
```
</details>

### Step 2: Expose Workflow as a Tool
1. Navigate to **Agent Builder > Tools** and click **Create Tool**.
2. Name it `Send Marketing Campaign` and choose **Workflow** as the tool type.
3. Select the **Marketing Campaign Approval** workflow and click **Save**.
4. Go to **Agent Builder > Agents** and open your **eCommerce Analyst** agent.
5. In the **Tools** section, click **Add Tool** and select **Send Marketing Campaign**. *(Ensure `platform.core.get_workflow_execution_status` and `platform.core.resume_workflow_execution` are also attached).*
6. Click **Save**.

### Step 3: Test the Workflow in Kibana Agent Builder
1. In the **Test your agent** chat panel, type:
   > *"Draft a promotional email for men's shoes and execute the Marketing Campaign Approval workflow to send it to my_email@example.com."*
2. The agent will call `execute_workflow` and pause for approval.
3. Reply: *"Yes, I approve the draft."*
4. The agent will resume the workflow and send the email!

### Step 4: Trigger the Action from Gemini
Now, run the exact same process from our external Gemini Enterprise interface:
> *"Based on the men's shoe sales data rising, please create a promotional email draft. Then, execute the marketing workflow to send it to my_email@example.com, but wait for my approval first."*

---
*Powered by Elastic & Google Cloud*