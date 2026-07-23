### RAG Security Fundamentals - Learning Notes
***

## 1. What is RAG?

**Retrieval-Augmented Generation (RAG)** allows a language model to use external documents when answering questions. Instead of relying solely on training data, the system retrieves relevant information at **inference time** and injects it into the model's context before generating a response.

- **Benefit:** Improves accuracy and freshness of answers.
- **Security implication:** It changes how **trust** works in the system. The model follows the structure of its input, but it never verifies whether retrieved content is correct, safe, or appropriate.

Because retrieval happens at inference time, malicious documents can influence responses **without retraining the model** — this is called **inference-time data poisoning**. Retrieved documents may even contain instruction-like text that overrides system intent, without the prompt itself ever being modified.

## 2. Core Components of a RAG System

| Component | Role |
|---|---|
| **Embedding Model** | Converts text (queries and documents) into vectors |
| **Vector Store** | Stores document embeddings; enables similarity comparison |
| **Retriever** | Finds relevant documents via similarity matching (not exact keywords) |
| **LLM** | Generates the final response using retrieved documents as context |

### Data Flow

1. User submits a query
2. Query is converted into an embedding
3. Vector store searches for similar document embeddings
4. Top matching documents are retrieved
5. Retrieved content is injected into the LLM's context
6. LLM generates a response

> ⚠️ At no point does the model verify whether the retrieved data is correct or safe.

## 3. RAG-Specific Attack Surfaces

Risk concentrates in three areas: **ingestion**, **retrieval**, and **context injection**.

1. **Document Ingestion** — RAG systems ingest data from shared drives, wikis, or automated feeds. Weak validation lets untrusted or malicious documents enter the knowledge base and become "trusted" information.
2. **Embedding Generation** — Converting documents to vectors strips context like authorship or approval status, so malicious and legitimate content look equally valid.
3. **Similarity-Based Retrieval** — Documents are selected by **semantic relevance, not trust or correctness**. An attacker's content only needs to "sound relevant" to be retrieved.
4. **Context Injection** — Retrieved documents go directly into the model's prompt. The model **cannot distinguish instructions from data**, so all retrieved content is treated as trusted context.

### Why Retrieval Is the Highest-Risk Component

Retrieval happens automatically and invisibly to the user. The LLM:

- Cannot see where documents came from
- Cannot verify document intent
- Cannot distinguish instructions from data

Once retrieved, content is treated as authoritative **because of its placement in the context window, not because it was verified**. This is a design limitation, not a configuration mistake.

## 4. Retrieval Abuse & Context Manipulation

**Retrieval abuse** = influencing model output by controlling which documents the retriever selects, instead of touching the prompt directly (unlike classic prompt injection).

### Active vs Passive

| Type | Description |
|---|---|
| **Passive poisoning** | Malicious content is ingested once and left in the knowledge base; the attacker waits for normal queries to retrieve it |
| **Active manipulation** | Content is deliberately crafted to rank highly for common or sensitive queries |

In both cases the attacker does **not** need continuous access to the system.

### How Context Manipulation Works

Problems arise when a retrieved document:

- Contains misleading or false information
- Includes hidden instructions framed as documentation

If a document ranks highly on similarity, it gets included in generation context — intent and safety are never checked.

### Why It's Hard to Detect

- Outputs appear logical and well-structured
- No visible prompt injection exists
- Logs only show "relevant documents retrieved" — the system behaves exactly as designed

### Security Impact

- Influence responses without modifying prompts
- Indirectly override system intent
- Introduce subtle misinformation or unsafe guidance

➡️ **Retrieval must be treated as a security boundary, not just a performance feature.**

## 5. Detection & Defence

There is no single reliable signal of abuse — poisoned content is semantically similar to queries and blends with clean content. Detection relies on observing **behaviour over time**.

### Guardrails on Retrieved Content

- Limit how retrieved text is inserted into prompts
- Separate retrieved data from system instructions
- Apply heuristics to flag instruction-like patterns

Guardrails are imperfect: instruction-like language is ambiguous, and attackers can rephrase/obfuscate to bypass simple checks. They **reduce** risk, not eliminate it.

### Validation During Ingestion

- Review document sources
- Enforce approval workflows
- Track ownership and update history

Once untrusted data is in the vector store, detection becomes much harder and more expensive — **prevent at ingestion**.

### Monitoring and Output Review

Focus on behavioural signals:

- Unusual retrieval patterns
- Repeated retrieval of the same documents
- **Output drift** — a gradual shift in response tone/behaviour over time; a key warning sign of poisoning

Behavioural monitoring is often the most effective way to catch RAG poisoning, since it captures subtle long-term deviations that automated controls miss.

### Defence-in-Depth

No single control fully protects a RAG system. Layered safeguards must:

- Reduce the likelihood of successful abuse
- Limit the impact of failures
- Detect problems early

## 6. Framework Mapping

### OWASP Top 10 for LLM Applications

| ID | Risk | RAG Relevance |
|---|---|---|
| LLM01 | Indirect Prompt Injection | Retrieved content influences behaviour without direct prompt access |
| LLM04 | Data & Model Poisoning | Inference-time poisoning via untrusted/stale retrieved data |
| LLM07 | Insecure Model Monitoring | RAG failures stay undetected without retrieval/output monitoring |

### NIST AI Risk Management Framework

- **Map:** Identify dependencies on internal and external knowledge sources
- **Measure:** Evaluate how retrieved data affects outputs
- **Manage:** Apply controls across ingestion, retrieval, and monitoring

## 7. Key Takeaways

- RAG expands the attack surface beyond traditional inputs
- Retrieval can amplify risk even when the system behaves "as designed"
- Security failures often occur **silently**, without obvious errors
- Retrieval is a **critical trust boundary** enabling indirect prompt injection, retrieval poisoning, and subtle manipulation — all without touching the user prompt
