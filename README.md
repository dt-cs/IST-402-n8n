# AI Meeting Summarizer

This repository contains the n8n workflows for a sophisticated **AI Meeting Summarizer**. This agent-based system allows users to process, summarize, and query Zoom and YouTube meetings through a simple chat interface.

The system is designed to be the central hub for all meeting-related information, leveraging AI to extract structured data and a dual-vector-database strategy for both broad and detailed semantic search.

## Key Features

* **Chat-based Interface:** Users interact with a simple chat trigger.
* **Multi-Platform Support:** Processes both **YouTube** and **Zoom** meeting URLs.
* **Intelligent Orchestration:** A main AI agent analyzes user intent to decide which tool to use.
* **Automated Transcription & Processing:** Calls external services to scrape transcripts from provided URLs.
* **AI-Powered Insights:** Uses `gpt-4o-mini` to generate:
    * Executive Summaries
    * Key Action Items (with owners)
    * Structured Insights & Decisions
    * Attendee Lists
* **Duplicate Prevention:** Checks if a meeting URL has already been processed before starting a new transcription job.
* **Dual Vector Storage:** Creates two types of vector embeddings for nuanced search:
    1.  **Summary Embeddings:** For broad, fuzzy search ("*Which meeting was about Project X?*").
    2.  **Transcript Chunk Embeddings:** For detailed, granular Q&A ("*What did Alex say about the Q4 deadline?*").
* **Conversation Memory:** Remembers the context of the user's conversation using a Postgres database.

---

## System Architecture

The project consists of two main n8n workflows that work together:

1.  **`Chat agent.json` (The Main Orchestrator)**: This is the user-facing workflow that manages the conversation and orchestrates all tasks.
2.  **`Transcript_maker.json` (The Processing Tool)**: This is a sub-workflow, run as a "tool" by the main agent, that handles the heavy lifting of transcription and database population.

---

### 1. Main Workflow: `Chat agent.json`

This workflow acts as the "brain" of the operation. It's a stateful agent that maintains a conversation and intelligently calls one of its three available tools based on the user's request.


**Core Components:**

* ** Chat Trigger:** Provides the public-facing chat interface for the user.
* ** AI Agent (Meeting Orchestrator):** The central agent (`gpt-4o-mini`) that interprets the user's needs. Its system prompt instructs it to follow a specific, step-wise logic.
* ** Postgres Chat Memory:** Remembers the previous messages in the conversation.

**The Agent's Tools:**

1.  ** Meeting Existence Checker Tool (`HTTP Request`)**
    * **Purpose:** To prevent redundant processing.
    * **Trigger:** Called whenever a user provides a new URL.
    * **Action:** Performs a `GET` request to the Supabase `meetinglist` table to check if a record with the `zoom_url` already exists.
    * **Logic:**
        * **If Found:** The agent informs the user the meeting is already processed and proceeds to the *Retrieval Tool*.
        * **If Not Found:** The agent proceeds to the *Transcript Maker Tool*.

2.  ** Transcript Maker Tool (`Workflow Tool`)**
    * **Purpose:** To process a new meeting.
    * **Trigger:** Called when a user provides a new, unprocessed URL.
    * **Action:** Executes the entire `Transcript_maker.json` sub-workflow, passing the URL as a parameter.

3.  ** Meeting Retrieval Tool (`Supabase Vector Store`)**
    * **Purpose:** To answer questions about meetings.
    * **Trigger:** Called when a user asks a question *without* a URL (e.g., "*What were my action items last week?*") or when they ask a question about a meeting that is already known to exist.
    * **Action:** Performs a semantic search against the `transcript_chunks` vector database to find the most relevant information to answer the user's query directly.

![Alt text for the image](path/to/your/image.jpg)
---

### 2. Sub-Workflow: `Transcript_maker.json`

This workflow is the "factory." It is not triggered directly by the user but is called *by* the `Chat agent`. Its sole purpose is to take a URL, process it, and populate the databases.



**Step-by-Step Flow:**

1.  **Receive URL:** The workflow starts with the URL provided by the main agent.
2.  **Classify URL:** An AI model (`gpt-4o-mini`) analyzes the URL and outputs a JSON object identifying it as `{"task": "YouTube", ...}` or `{"task": "Zoom", ...}`.
3.  **Scrape Transcript:** A Switch node routes the URL to the correct scraper:
    * **Zoom:** Calls a custom Google Cloud Run service (`zoom-transcript-scraper...`) to get the transcript.
    * **YouTube:** Calls a custom Google Cloud Run service (`youtubecc...`) to get the transcript.
4.  **Generate AI Insights:** The raw transcript is fed to `gpt-4o-mini` with a specific prompt to extract structured JSON containing the **summary, action items, insights, attendees, and metadata**.
5.  **Populate Databases (Dual Storage Strategy):** This is the most critical step.
    * **Database 1: `meetinglist` (For Summaries)**
        1.  **Insert Row:** The AI-generated summary and metadata are saved to a primary table in Supabase (`HTTP Request5`).
        2.  **Create Summary Embedding:** A *new* embedding is generated *only* from the AI-generated summary and title (`LangChain Code1`).
        3.  **Update Row:** This "summary embedding" is added to the meeting's row in `meetinglist` (`HTTP Request3`). This embedding is used by the `Meeting Existence Checker Tool` for broad semantic matching.
    * **Database 2: `transcript_chunks` (For Detailed Q&A)**
        1.  **Chunk Transcript:** The *original, raw transcript* is split into small, overlapping chunks (`LangChain Code`).
        2.  **Create Chunk Embeddings:** An embedding is created for *each individual chunk*.
        3.  **Insert Chunks:** Each chunk, its embedding, and the `transcript_id` (linking it to the main `meetinglist` row) are saved to the `transcript_chunks` table (`HTTP Request1`). This database is used by the `Meeting Retrieval Tool` for detailed Q&A.

![Alt text for the image](path/to/your/image.jpg)
---

##  Setup & Prerequisites

To run this project, you will need:

* **n8n:** An active n8n instance (cloud or self-hosted).
* **OpenAI:** An API key for `gpt-4o-mini` and `text-embedding-3-small`.
* **PostgreSQL:** A database for the Chat Memory node.
* **Supabase Account:**
    * A project with two tables:
        1.  `meetinglist`: To store the main meeting record, metadata, and summary embedding.
        2.  `transcript_chunks`: To store the raw transcript chunks and their individual embeddings.
    * A database function (e.g., `match_meetings`) for vector search, as referenced by the `Meeting Retrieval Tool`.
* **Deployed Scraper Services:**
    * This workflow relies on two **external, custom-deployed services** (likely on Google Cloud Run) to handle the actual transcript scraping:
        * `https://youtubecc-423771082043.us-central1.run.app`
        * `https://zoom-transcript-scraper-423771082043.us-central1.run.app`
    * You will need to deploy your own versions of these scraper functions for the workflow to be operational.

##  How to Use

1.  Ensure all prerequisites, database tables, and credentials are correctly set up in your n8n instance.
2.  Activate the `Chat agent.json` workflow.
3.  Access the chat URL provided by the `When chat message received` trigger.
4.  Start the conversation. You can either:
    * **Provide a new URL:** "Here is the link to our last planning session: https://zoom.us/..."
    * **Ask a question:** "What were the action items from the 'Project X' meeting?"
