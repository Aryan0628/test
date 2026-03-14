# UrbanConnect Civic Analysis LangGraph Workflow

This diagram illustrates the multi-modal RAG pipeline for post classification, clustering, and fact-checking.

```mermaid
graph TD
    %% Entry Point
    START((New Post)) --> N1[Node 1: Triage & Sentiment]
    
    %% Node 1
    subgraph "Classification Layer"
    N1 -->|Analysis| P1[Analyze Title + Description]
    P1 -->|Vision| V1[Analyze Images]
    V1 -->|Result| T1{Post Type?}
    end

    %% Transition to Node 2
    N1 --> N2[Node 2: Vectorize & Cluster]

    %% Node 2
    subgraph "Clustering Layer"
    N2 --> E1[Generate 768d Embedding]
    E1 --> C1[POST /api/cluster-check]
    C1 --> |Similarity >= 0.75| S1[Assing Cluster ID]
    S1 --> |Size >= 5| SM[Fire-and-Forget Summary Agent]
    end

    %% Routing Logic
    N2 --> ROUTE{Post Type?}
    
    ROUTE -->|POLICY_RUMOR| N3[Node 3: Multi-Modal RAG Fact-Check]
    ROUTE -->|CIVIC_REPORT / GENERAL| END((END: Save AI Analysis))

    %% Node 3
    subgraph "Fact-Checking Layer"
    N3 --> R1[POST /api/announcements/search]
    R1 --> R2{Official Context Found?}
    
    R2 -->|No / Low Results| WS[Web Search Fallback: Tavily API]
    R2 -->|Results Found| MC[Combine RAG Context]
    WS --> MC
    
    MC --> FC[Gemini 2.0 Flash Fact-Check]
    FC --> |Verdict| MN[Generate Misinformation Note]
    end

    MN --> END
```

## Node Descriptions

### **Node 1: Triage & Sentiment**
- **Inputs:** `title`, `description`, `image_urls`.
- **Purpose:** Classifies the post into `CIVIC_REPORT` (raw onsite issues), `POLICY_RUMOR` (claims requiring verification), or `GENERAL`.
- **Logic:** Uses Gemini 2.0 Flash for structured output.

### **Node 2: Vectorize & Cluster**
- **Inputs:** Combined text from Node 1.
- **Purpose:** Vectorizes the post and checks for emerging issues.
- **Workflow:** 
  1. Generates 768-dim vector.
  2. Hits Node.js `/cluster-check` to find siblings within a 12h window.
  3. If cluster size reaches 5, triggers the **Cluster Summary Agent**.

### **Node 3: Multi-Modal RAG Fact-Check**
- **Trigger:** Only runs if Post Type is `POLICY_RUMOR`.
- **Workflow:**
  1. Performs a vector search on the `announcements` collection.
  2. **Fallback:** If internal search finds < 2 releases, hits the **Tavily Web Search** for live ground truth.
  3. **Visual Verification:** Analyzes post images for forgery or contradictions.
  4. **Final Verdict:** Flags `is_misinformation` and generates a `context_note`.
