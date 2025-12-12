# Control-whatsapp Workflow

This n8n workflow operates as an intelligent WhatsApp bot control center. It receives incoming messages via Twilio, uses an AI Agent to classify user intent, and executes specific sub-workflows based on that intent. The system handles calendar events, research tasks, database queries, and RAG (Retrieval-Augmented Generation) searches, providing feedback to the user via WhatsApp.

## Workflow Overview

The core logic revolves around an **AI Agent** (powered by OpenAI) that acts as a router. Depending on the user's message, it categorizes the request into one of four distinct paths:

1.  **set_event**: Schedules meetings or reminders in Google Calendar.
2.  **do_research**: Adds a research task to a queue in Supabase for a separate agent to process.
3.  **exec_action**: Converts natural language into SQL queries to retrieve and summarize data from a Postgres database.
4.  **rag_search**: Performs a file-based search using Google's Gemini 2.5 Pro model to answer complex queries.


## Detailed Logic

### 1. Classification (The "Brain")
*   **Node**: AI Agent (`@n8n/n8n-nodes-langchain.agent`)
*   **Model**: GPT-5 Mini (configured as `gpt-5-mini` in the node, acting as the decision maker).
*   **Function**: Analyzes the message body and outputs a JSON object containing the `intent` and `details`.

### 2. Paths

#### i) set_event
*   **Trigger**: User asks to set a reminder, call, or meeting (e.g., "Set a meeting with Mani tomorrow at 10am").
*   **Action**:
    *   Extracts `event_name`, `start_time`, and `end_time`.
    *   Creates an event in **Google Calendar**.
*   **Output**: Sends a WhatsApp confirmation: "PAPA has successfully created the event on your calendar."

#### ii) do_research
*   **Trigger**: User asks to research a topic (e.g., "Research latest developments in quantum physics").
*   **Action**:
    *   Fetches the current `research_agent` configuration from **Supabase**.
    *   Appends the new research task to the task list.
    *   Updates the `agent_config` table in Supabase.
*   **Note**: This path updates the backend state for a separate research agent to pick up; it does not send a final result message in this specific flow.

#### iii) exec_action
*   **Trigger**: User requests a summary or data lookup (e.g., "Summarize my day today").
*   **Action**:
    *   Generates a SQL query based on the user request and the `agent_output` table schema.
    *   Executes the query against a **Postgres** database.
    *   Uses a secondary **AI Agent** to summarize the SQL result payload.
*   **Output**: Sends the summarized insight back to the user via WhatsApp.

#### iv) rag_search
*   **Trigger**: User explicitly asks for a RAG search (e.g., "Control use RAG to find out which frameworks support RAG natively").
*   **Action**:
    *   Retrieves `control_agent` configuration from **Supabase** to determine file search categories.
    *   Calls **Google Gemini 1.5 Pro** API (`generativelanguage.googleapis.com`) with a `file_search` tool configuration.
    *   Rephrases the raw LLM output into clear, plain-English paragraphs using a **Categorizer** agent.
*   **Output**: Sends the detailed answer back to the user via WhatsApp.

## Prerequisites

To run this workflow, you need the following credentials and services configured in n8n:

1.  **Twilio Account**: For receiving messages and sending replies (WhatsApp Sandbox or Business API).
2.  **OpenAI API**: For the primary classification agent and summarizer.
3.  **Google Calendar**: OAuth2 credentials to manage events.
4.  **Supabase**:
    *   Table `agent_config`: Stores agent tasks and configurations.
    *   Table `agent_output`: Stores logs/data for the `exec_action` path.
5.  **Postgres**: Database connection for executing generated SQL queries.
6.  **Google Gemini API Key**: For the `rag_search` HTTP request node.

## Setup

1.  **Import**: Import the `Control-whatsapp.json` file into your n8n instance.
2.  **Credentials**: Update all credential nodes (Twilio, OpenAI, Google Calendar, Supabase, Postgres) with your own keys.
3.  **Webhook**: Configure your Twilio WhatsApp sender to point to the `whatsapp-in` webhook URL exposed by n8n.
4.  **Database**: Ensure your Supabase/Postgres tables (`agent_config`, `agent_output`) match the schema expected by the AI Agent prompts.

