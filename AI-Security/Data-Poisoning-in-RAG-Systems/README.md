### Data Poisoning in RAG Systems - Learning Notes
***

## 1. What Is Data & Model Poisoning?

LLMs learn how to behave from the data they are trained on and continue to consume. Every pattern, association, and assumption originates from that data. **Data poisoning (model poisoning)** attacks the information the model learns from, rather than the prompt or the user.

- Categorised under **OWASP LLM04 – Data & Model Poisoning**
- Targets the system **before it is queried**

### Poisoning vs Prompt-Based Attacks

| | Prompt Injection / Excessive Agency | Data & Model Poisoning |
|---|---|---|
| **When** | Inference time | Upstream — training, ingestion, indexing |
| **What is manipulated** | Instructions in the prompt | What the model learns / retrieves |
| **Interaction needed** | Yes, per attack | No — one event can affect thousands of queries |

> The model is not being tricked. It is behaving exactly as it was trained to behave.

### Scenario: Poisoning Without Touching the Model

A company deploys an internal AI assistant for policies and procedures. It does not learn from users — it ingests and indexes a growing collection of internal documents (drafts, updated manuals, archived files, third-party reports). Nothing breaks. Weeks later, employees act on incorrect guidance: a security control is described inaccurately, a process is quietly altered.

- No one attacked the model directly
- No prompt was injected
- The attacker only needed to influence **what the system was allowed to read**

### Why Poisoning Is Dangerous

- Effects are **delayed and persistent**
- Removing the original source does not guarantee the behaviour disappears
- The system appears reliable, answers confidently, and passes basic testing
- The failure is invisible, but system **integrity** is already compromised

## 2. Training Data Poisoning

An attacker manipulates the data used to train or fine-tune the model — not the code or weights directly.

**Mechanism:** During training, the model updates parameters via **gradient descent** (gradually adjusting parameters in the direction that reduces prediction error). Each example slightly shifts weights based on prediction error. Over millions of updates, statistical patterns from the dataset become internalised. Poisoned data biases those updates.

### Training Data Across the AI Lifecycle

| Stage | Description | Poisoning Sensitivity |
|---|---|---|
| **Pre-training** | Large datasets scraped from public sources and licensed archives; general language patterns and baseline assumptions | Requires large volume |
| **Fine-tuning** | Smaller, targeted data adapting the model to a task/domain | **High** — even small amounts strongly affect behaviour |
| **Curated corpora / KBs** | Internal knowledge bases shaping outputs without retraining | Affects responses directly |

### How Attackers Poison Source Data

Poisoning usually begins with access to a **trusted** data source. Attackers may insert new documents, modify existing ones, or shift content gradually over time.

Effective poisoning is **subtle** rather than obviously false:

- A definition is slightly reframed
- A policy includes a quiet exception
- A specific phrasing appears repeatedly

**Repetition increases impact.** Models learn from patterns across data — if a poisoned idea appears often enough, it is more likely to be internalised. The model does not "know" which data was malicious; it optimises for **consistency across the dataset** (e.g. repeated samples teach it to associate Product X with negative sentiment).

### Intentional Poisoning vs Accidental Data Issues

| | Accidental | Intentional |
|---|---|---|
| **Cause** | Errors, outdated info, bias | Attacker-designed content |
| **Outcome** | Random error | Predictable, repeatable result |
| **Goal** | None | Consistent behaviour aligned with attacker objective |

### Why Poisoned Data Persists

The model stores **learned patterns, not individual files**. Removing the source documents may not remove the effect, and retraining is expensive and complex. Training data poisoning is a **long-term integrity risk**, not a temporary failure — every downstream system (embeddings, ingestion, retrieval) inherits the distortion.

## 3. Embedding & Corpus Poisoning

An **embedding** is a numerical representation of text capturing meaning rather than exact words; similar meanings sit closer together in semantic space. A **vector database** stores embeddings and retrieves documents by similarity. What the model sees depends entirely on this ranking process.

### How Similarity Search Controls Outputs

- Similarity measures **closeness in meaning — not correctness or authority**
- Most systems return only the **top-k** results; lower-ranked documents may never reach the model
- This creates **competition**: if an attacker moves poisoned documents closer to likely queries, legitimate documents can remain untouched but unused

> Controlling ranking means controlling influence.

A poisoned document with a higher cosine similarity score outranks a legitimate one even when both are close in meaning. In real systems embeddings have hundreds or thousands of dimensions — small shifts in semantic phrasing can move a document closer in vector space and raise its chance of landing in the top-k.

### Corpus Poisoning Techniques

| Technique | Description |
|---|---|
| **Keyword stuffing** | Repeat common search phrases |
| **Semantic mimicry** | Imitate the tone and structure of trusted documents |
| **Duplication / corpus flooding** | Upload multiple slightly modified copies of the same idea |

Clustering many near-duplicates increases the **local density** around a topic. During nearest-neighbour search, a dense cluster raises the probability that at least one poisoned vector appears in the top-k — even if each individual document is only slightly similar to the query. The ranking algorithm does not understand intent; it selects what is closest and most frequent in that region of vector space.

### Why Legitimate Data Can Remain Untouched

Trusted documents are not deleted or modified. The attack **shifts retrieval outcomes**. The system still contains correct information — it simply does not surface it. This makes embedding poisoning hard to spot through simple audits.

### Comparing Poisoning Layers

| Layer | What It Manipulates | Model Weights Changed? |
|---|---|---|
| **Training Data Poisoning** | Internal weights during training/fine-tuning; distortion becomes part of learned parameters | ✅ Yes |
| **Embedding-Level Poisoning** | How documents are represented in vector space; similarity relationships | ❌ No |
| **Corpus Flooding** | Density of attacker-controlled documents in a semantic region, raising retrieval probability | ❌ No |

Training data poisoning changes **what the model learns**; embedding/corpus poisoning changes **what the model sees at inference time**. Because retrieval is dynamic, embedding poisoning impact can be **immediate and selective**.

## 4. Ingestion Pipeline Attacks

Modern LLM systems rarely use static datasets — they continuously collect, process, and index new documents through an **ingestion pipeline**: collection → parsing → chunking → embedding → indexing → storage. Once processed, content becomes part of the searchable knowledge base.

### Where Trust Assumptions Exist

Pipelines assume incoming data is safe. Files from internal drives, shared folders, APIs, or web sources are treated as valid input. **Automation increases scale but reduces scrutiny** — documents may be parsed and embedded without human review. Access to any trusted ingestion source grants indirect influence over the model.

### How Attackers Exploit Ingestion

Ingestion is often triggered by scheduled jobs or filesystem events. The pipeline checks **file permissions but not semantic intent** — a malicious instruction embedded in otherwise legitimate text is processed identically to trusted content.

Attackers may:

- Upload a malicious document into a shared directory
- Modify an existing file that is automatically re-indexed
- Inject poisoned content into a third-party feed
- Exploit weak validation rules in file parsers

### Automation as an Attack Multiplier

If ingestion runs hourly or daily, poisoned content spreads quickly with no clear signal that anything changed. The infrastructure keeps operating normally. In many deployments ingestion is treated as an **engineering problem rather than a security boundary**.

### Why Ingestion Is a Security Boundary

- Ingestion determines what becomes **persistent system knowledge**
- Unlike prompt attacks, ingestion abuse **modifies stored state** — the attack does not need to be repeated
- Every scheduled re-indexing job effectively **redefines what the model is allowed to know**
- If validation is weak, the trust boundary collapses **at scale**

> Training data poisoning shapes what the model **learns**.
> Embedding poisoning shapes what the model **retrieves**.
> Ingestion pipeline attacks determine what **enters the system** in the first place.

## 5. Behavioural Impact

Poisoning does not usually cause crashes or visible errors. The model keeps producing fluent, confident responses; the change occurs in **assumptions, framing, or recommendations**.

### Obvious vs Subtle Effects

| Obvious | Subtle (more dangerous) |
|---|---|
| Backdoor triggers activating specific behaviour | Slightly favouring one product over another |
| Persona shifts or tone changes | Adjusting regulatory thresholds by small margins |
| Clearly incorrect or extreme responses | Reframing a security recommendation |
| | Omitting critical warnings |

Obvious failures stand out and attract attention — they are easier to detect and investigate. Subtle distortions each look reasonable in isolation but **influence decisions at scale** over time.

### Why Subtle Effects Are Hard to Notice

- LLMs are **probabilistic** — outputs vary naturally, blurring malicious drift and normal variation
- No system errors → infrastructure logs stay clean, no alerts trigger, responses stay fast
- The only difference is **behavioural drift**, which can persist for a long period

### Case Study: Waze Traffic Poisoning

Researchers and local residents demonstrated that Waze could be manipulated by injecting **false traffic data** — repeatedly reporting fake incidents or simulating slow "ghost cars" to create artificial congestion hotspots.

- The routing model was **not modified**; it simply trusted poisoned GPS and incident data
- Small amounts of fake data → subtle changes (slightly longer ETAs, marginal route shifts)
- Larger attacks → obvious effects (red traffic jams, forced detours around clear roads)
- Infrastructure remained fully operational; only the system's **learned view of traffic** changed

➡️ Core principle: **same model, same code, same system — different behaviour due to corrupted input data.**

### System-Level Consequences

- Trust in the AI system degrades
- Decisions may be influenced in unintended ways
- Compliance and safety risks increase
- The source of the problem becomes difficult to trace (impact surfaces long after the upstream cause)

## 6. Detection & Mitigation

### Why Poisoning Is Difficult to Detect

Poisoning rarely triggers technical failures — logs stay clean, pipelines operate normally, outputs stay coherent. The problem is **behavioural drift**, so detection requires monitoring **trends over time** rather than isolated responses.

### No Single Control Is Enough

Keyword blocking is insufficient; malicious content can be subtle and context-aware. Poisoning may occur at multiple layers, each needing different controls:

- Training data
- Ingestion pipelines
- Vector databases
- Retrieval ranking

### Validation at Ingestion

Treat incoming data as **untrusted until validated**. Automated sources, shared drives, and third-party feeds should never be blindly embedded or indexed.

- Source verification
- Access control restrictions
- Structured content review
- Logging and change tracking

### Monitoring Behavioural Drift

- Tracking shifts in tone or persona
- Detecting consistent recommendation bias
- Comparing outputs **before and after data updates**

Behavioural monitoring does not guarantee detection, but it increases visibility into subtle changes.

### Case Study: Amazon Fake Reviews & Layered Detection

Organised brokers recruited users to post coordinated 5-star "verified purchase" reviews (or fake negatives against competitors), poisoning signals that drove search rankings, "Amazon's Choice" badges, and personalised recommendations.

Detection was hard: fake reviews looked legitimate, varied in wording, spread across many accounts, and had no ground-truth label. As poisoned products gained visibility, **genuine buyers added genuine reviews**, blending malicious and legitimate data.

Amazon's layered response — ML models blocking suspicious reviews pre-publication, behavioural anomaly detection, identity restrictions, human investigation, legal action against brokers, and downstream ranking corrections — reportedly blocked **over 250 million suspected fake reviews** in 2023–2024.

➡️ Key principle: **detection is probabilistic, and no single control is sufficient.**

### Review and Governance

Poisoning is ultimately a **data integrity** issue. Treat training data and retrieval corpora as sensitive assets:

- Change management
- Access auditing
- Periodic review of indexed content

Governance controls matter as much as technical ones. Security for AI systems is not only about models — it is about controlling **what the model learns from and what it retrieves**.

## 7. Framework Alignment

### OWASP Top 10 for LLM Applications

| ID | Risk | Relevance |
|---|---|---|
| LLM04 | Data & Model Poisoning | Attackers manipulate training data, embeddings, or corpora to influence behaviour |
| LLM07 | Insecure Model Monitoring | Behavioural drift remains undetected without proper monitoring |
| LLM05 | Supply Chain Vulnerabilities | External data sources and ingestion pipelines expand the attack surface |

### NIST AI Risk Management Framework

- **Map:** Identify all data sources that influence model behaviour
- **Measure:** Monitor behavioural drift and ranking anomalies
- **Manage:** Apply layered controls across ingestion, storage, and monitoring

### EU AI Act

- **Article 9:** Continuous risk management for system behaviour
- **Article 10:** Data governance, quality, and lifecycle integrity

Across these frameworks, poisoning is treated as a **system-level integrity failure**, not a model defect.

## 8. Key Takeaways

- **Control over data can equal control over behaviour**
- Embedding and ranking manipulation can influence outputs **without retraining**
- **Automation amplifies** poisoning risk at scale
- **Subtle behavioural drift is often more dangerous** than obvious failure
- **No single detection mechanism is sufficient** — defence must be layered
- Poisoning attacks are powerful because they target **trust, not code**, exploiting assumptions about data, automation, and relevance

***

**Related notes:** [RAG Security Fundamentals](../RAG-Security-Fundamentals/README.md)
