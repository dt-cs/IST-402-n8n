# 1. Project Title
AI Meeting Summarizer

---

# 2. Overview

This project delivers a fully functional AI Meeting Summarizer capable of processing Zoom and YouTube meeting URLs, automatically retrieving transcripts, and generating structured summaries, action items, insights, and searchable embeddings. Users interact through an n8n chat interface powered by a tool-orchestrating agent using OpenAI’s gpt-4o-mini model.

The system supports:

- Real-time transcript processing  
- Automatic meeting classification (Zoom or YouTube)  
- Semantic search across previously processed meetings  
- Context-aware natural-language questioning  
- Structured extraction of action items, decisions, insights, and attendees  
- Persistent conversation memory (we have limited this to 25)

The full application is implemented through two n8n workflows, supported by Supabase and Google Cloud Run microservices.

---

# 3. Input Definition 

This project **does not use any predefined dataset**.

Instead, **the user provides a single input**:

- A **Zoom meeting URL**, or  
- A **YouTube video URL**

Transcripts are fetched dynamically from these sources.  
No static corpus, dataset, or collection of transcripts is required or used.

All processed transcripts come **directly from the user’s provided link**, and the system only processes data explicitly provided or authorized by the user.

Stored data consists solely of:

- The user’s processed meeting summaries  
- Raw transcript chunks (from that same URL)  
- Embeddings used for retrieval  

All stored data originates **only from the user’s submitted meeting/video URLs**.

---

# 4. Techniques Used

## 4.1 Large Language Model  
Model used: gpt-4o-mini  
Used for:  
- URL classification  
- Transcript analysis and summarization  
- Extracting action items, insights, attendees  
- Orchestrator decision-making  
- Retrieval-augmented Q&A  

## 4.2 Tool Orchestration (LangChain + n8n)
- LangChain code node inside n8n is used for transcript chunking and embedding creation  
- n8n handles:
  - Chat interface  
  - Agent instructions  
  - Live transcript retrieval - custom function served in Google Cloud 
  - Database storage using Supabase 
  - Vector search  
  - Tool selection and pipeline execution  

## 4.3 Retrieval-Augmented Generation (RAG)
Two separate vector stores:

- Summary embeddings → broad semantic search  
- Transcript chunk embeddings → detailed Q&A retrieval  

---

# 5. System Architecture

User → n8n Chat Interface → AI Orchestrator Agent (gpt-4o-mini)  
→ Tool 1: Meeting Existence Checker (Supabase)  
→ Tool 2: Transcript Maker Workflow  
  → URL Classifier  
  → Transcript Scraper (Cloud Run)  
  → AI Summary + Action Item Extraction  
  → Insert meetinglist record  
  → LangChain Summary Embedding  
  → Transcript Chunking + Embeddings  
  → Insert transcript_chunks  
→ Tool 3: Vector Retrieval (Supabase)  
→ Chat Response

Components:

- n8n Chat Trigger (frontend)  
- n8n agent workflows (backend)  
- Supabase (PostgreSQL + pgvector)  
- GCP Cloud Run scrapers  
- OpenAI models (LLM and embeddings)

---

# 6. Technical Decisions

- **Model:** gpt-4o-mini  (Cheap and Efficient)
- **Backend:** n8n  
- **Frontend:** n8n Chat Interface  
- **Action Item Extraction:** Prompt-engineered structured JSON adhering to table schema created in Supabase project database 
- **Speaker Identification:** Derived from transcript metadata  
- **Output Format:** Displayed chat messages formatted in markdown syntax
- **Deployment:** n8n self hosted

---

# 7. Prompt Engineering Examples

- Transcript classification  
- Structured summary + action items extraction   
- URL classification into Zoom/YouTube  

---

# 8. Workflow Design

## 8.1 Chat Orchestrator  
- Conversational interface  
- Maintains memory  
- Selects tools (Transcript maker, Meeting existence Checker, Meeting Retriever)  
- Executes the chosen workflow behaviour 

## 8.2 Transcript Maker  
- Classifies URL  
- Retrieves transcript via Cloud Run scrapers  
- Generates structured summary + metadata  
- Stores meeting list entry  
- Creates summary and chunk embeddings  
- Inserts transcript chunks

## 8.3 Meeting Existence Checker  
This workflow makes sure a meeting or video URL is **not processed more than once**.

### What it Does:
- Receives a URL from the Orchestrator Agent  
- Checks the Supabase `meetinglist` table to see if the meeting already exists  
- Returns either:
  - **EXISTS** → Use stored summary and insights  
  - **NOT_FOUND** → Run the Transcript Maker workflow  

### Why it Matters:
- Saves processing time  
- Avoids duplicate transcripts and embeddings  
- Reduces API cost  
- Keeps the database clean  

## 8.4 Meeting Retriever (Vector Search Tool)  
This workflow handles all **semantic search** and **Q&A** after a meeting has been processed.

### What it Does:
- Receives the user’s question  
- Turns the question into an embedding  
- Searches:
  - Summary embeddings (for general topics)  
  - Transcript chunk embeddings (for detailed answers)  
- Returns the most relevant pieces of information to the agent  

### Why it Matters:
- Lets users ask natural questions about any processed meeting  
- Supports detailed queries like:
  - “What did they say about deadlines?”  
  - “Show all action items from last week.”  
- Enables accurate RAG responses  
- Provides long-term searchable memory across all meetings  

---

# 9. Success Metrics for evaluation

- Summary accuracy (Expected -  90%+)
- Processing time (Expected - under 1 min for new URL and less than 30s for querying existing meeting )
- Duplicate detection (Implemented mechanism in the supabase database table to avoid repeating the creation of database for previously provided links)
- Retrieval accuracy (Expected - 90%+ )

---

# 10. Error Handling

- Invalid URLs → Message to user  
- Duplicate meetings → Skip processing and load stored results  
- No transcript returned → Process stops  
- No action items → Return empty list  

---

# 11. Ethical & Privacy Considerations

## 11.1 Transcript Access Limitations  
Zoom and YouTube **only permit access to transcripts for the owner of the meeting or video** (or users explicitly granted access).  
However, when scraping is used:

- Zoom transcripts behind authentication **cannot be ethically or legally obtained through scraping**  
- YouTube may hide transcripts or require authentication  
- Using proxies to bypass restrictions violates:
  - Terms of Service  
  - User consent principles  
  - Platform copyright rules  

Thus, scraping transcripts without explicit permission is **not ethically acceptable**, even if the URL is publicly accessible.

This system is intended to be used **only when the user owns the meeting/video or has legitimate access**.

## 11.2 User Consent and Ownership  
The system assumes the user:

- Owns the Zoom meeting or YouTube video  
- Has rights to its transcript  
- Intentionally inputs the meeting URL for processing  

## 11.3 Data Security  
- All processed data is stored only in the user’s Supabase instance  
- No external logging or unauthorized data retention  
- No third-party redistribution of transcripts  
- Embeddings contain semantic information but not full transcript text  

## 11.4 Model Bias and Accuracy  
- AI-generated summaries may contain errors or omissions  
- Users are encouraged to verify action items and critical decisions  
- The model does not generate or infer sensitive personal data  

---

# 12. Creativity & Innovation

- Dual-vector RAG architecture  
- Automatic URL classification (Zoom or Youtube - This can be scaled for Google meets, Teams,...)
- Cloud Run scrapers  (Custom function for transcript creation served in Google Cloud)

---

# 13. Demo Plan

1. User submits a Zoom/YouTube link  
2. System checks if meeting exists  
3. If new → Transcript Maker workflow runs  
4. Outputs summary, action items, insights  
5. User asks follow-up questions  
6. System uses vector search to answer  
7. Show workflows and embeddings  
8. Display total processing time  

---

# 14. GitHub Repository  
[(Project Link)](https://github.com/dt-cs/IST-402-n8n)
