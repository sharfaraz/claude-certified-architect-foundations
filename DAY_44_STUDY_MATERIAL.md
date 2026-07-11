# Day 44: Context Compression, Human Review Workflows, and Information Provenance

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 5: Context Management & Reliability (15%) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Building with Claude API Pod 5 |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Context Compression Techniques](#1-context-compression-techniques)
2. [When to Compress](#2-when-to-compress)
3. [Human Review Workflows](#3-human-review-workflows)
4. [Designing Review Triggers](#4-designing-review-triggers)
5. [Information Provenance](#5-information-provenance)
6. [Why Provenance Matters](#6-why-provenance-matters)
7. [Connecting to the Multi-Agent Research Scenario](#7-connecting-to-the-multi-agent-research-scenario)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Docs — Long Context | https://docs.anthropic.com/en/docs/build-with-claude/long-context |
| Claude Agent SDK — Human Review | https://code.claude.com/docs/en/agent-sdk/human-review |
| Anthropic Safety Practices | https://docs.anthropic.com/en/docs/build-with-claude/safety |
| Claude API Token Counting | https://docs.anthropic.com/en/api/messages-count-tokens |

---

## 1. Context Compression Techniques

### The Three Core Techniques

| Technique | How It Works | Token Reduction | Information Loss |
|-----------|-------------|-----------------|-----------------|
| **Summarization** | Replace verbose messages with concise summaries | 70-90% | Moderate (detail lost) |
| **Key-Fact Extraction** | Keep only assertions/conclusions, drop reasoning | 60-80% | Low (facts preserved) |
| **Dropping Low-Relevance** | Remove messages that don't contribute to current task | 100% per message | Variable |

### Technique 1: Summarization

Replace a block of conversation with a single summary message.

```python
# Before compression (5 messages, ~3000 tokens):
messages = [
    {"role": "user", "content": "Analyze the authentication module..."},
    {"role": "assistant", "content": "I'll look at the auth module. First, let me read the files... [long reasoning]"},
    {"role": "assistant", "content": "[tool_use: read_file auth.ts]"},
    {"role": "user", "content": "[tool_result: 500 lines of code]"},
    {"role": "assistant", "content": "After analyzing auth.ts, I found three issues: 1) No rate limiting... 2) Token expiry not checked... 3) Missing CSRF protection..."}
]

# After compression (1 message, ~200 tokens):
compressed = [
    {"role": "assistant", "content": "[Summary of auth analysis: Found 3 security issues in auth.ts: (1) no rate limiting on login endpoint, (2) JWT token expiry not validated, (3) missing CSRF protection on state-changing routes. Source: auth.ts]"}
]
```

### Technique 2: Key-Fact Extraction

Strip reasoning, keep only conclusions and data.

```python
def extract_key_facts(messages):
    """Extract only factual claims from conversation."""
    facts = []
    for msg in messages:
        if msg["role"] == "assistant":
            # Extract structured findings
            findings = extract_findings(msg["content"])
            for finding in findings:
                facts.append({
                    "claim": finding.claim,
                    "confidence": finding.confidence,
                    "source": finding.source_reference
                })
    
    return {"key_facts": facts}

# Result: Just the facts, no "let me think about this..." reasoning
# Original: 5000 tokens → Extracted facts: 800 tokens
```

### Technique 3: Dropping Low-Relevance Messages

Identify and remove messages that don't contribute to the current task.

```python
def score_relevance(message, current_task):
    """Score how relevant a message is to the current task."""
    # Factors:
    # - Recency (newer = more relevant)
    # - Semantic similarity to current task
    # - Contains a decision or conclusion vs. exploration
    # - Referenced by later messages
    pass

def drop_low_relevance(messages, current_task, threshold=0.3):
    """Remove messages below relevance threshold."""
    scored = [(msg, score_relevance(msg, current_task)) for msg in messages]
    return [msg for msg, score in scored if score >= threshold]
```

### Combining Techniques (Tiered Approach)

```python
def tiered_compression(messages, utilization):
    """Apply different compression levels based on urgency."""
    
    if utilization < 0.8:
        # Light: just summarize tool results
        return summarize_tool_results(messages)
    
    elif utilization < 0.9:
        # Medium: summarize older turns + extract key facts
        old = messages[:-5]
        recent = messages[-5:]
        summary = summarize_to_key_facts(old)
        return [summary_message(summary)] + recent
    
    else:
        # Heavy: aggressive extraction + drop low-relevance
        essential = drop_low_relevance(messages, current_task)
        facts = extract_key_facts(essential)
        recent = messages[-3:]
        return [facts_message(facts)] + recent
```

---

## 2. When to Compress

### Compression Triggers

| Trigger | Action | Rationale |
|---------|--------|-----------|
| **Approaching token limits** (>80%) | Compress older context | Prevent request failure |
| **Between pipeline phases** | Compress phase output before passing | Each phase gets clean context |
| **Before passing to subagents** | Compress coordinator context for subagent | Subagent needs focused, not full context |
| **After tool-heavy sequences** | Summarize tool results | Tool results are verbose |
| **Before human review** | Compress for readability | Humans can't review 50K tokens |
| **Fan-in aggregation** | Compress multiple outputs before merging | Combined outputs may exceed limits |

### Compression Decision Matrix

```
Should I compress?
  │
  ├── Is utilization > 80%? → YES (mandatory)
  │
  ├── Am I crossing a phase boundary? → YES (best practice)
  │
  ├── Am I passing context to a subagent? → YES (focused context)
  │
  ├── Are there > 10 tool results in history? → YES (verbose)
  │
  ├── Is a human going to review this? → YES (readability)
  │
  └── None of the above → NO (preserve full context)
```

### What to Preserve During Compression

| ALWAYS Preserve | CAN Compress | SAFE to Drop |
|----------------|--------------|--------------|
| Key findings/conclusions | Detailed reasoning | Exploration dead-ends |
| Decisions made | Full tool results | Repeated attempts |
| Source attribution | Step-by-step logic | Format/style discussions |
| Confidence scores | Intermediate summaries | Tool call metadata |
| Pending tasks | Old user clarifications | Superseded information |
| Error context (for retries) | Verbose examples | Redundant confirmations |

---

## 3. Human Review Workflows

### The Approval Gate Pattern

Human review workflows insert a human decision point at critical junctures in an agentic system.

```
Agent Working → [Critical Decision Point] → PAUSE
                                              │
                                    ┌─────────┼─────────┐
                                    ▼         ▼         ▼
                                 APPROVE    MODIFY    REJECT
                                    │         │         │
                                    ▼         ▼         ▼
                                Continue   Re-do    Halt/Escalate
                                            with
                                          feedback
```

### Types of Review Gates

| Gate Type | When It Activates | Human Action |
|-----------|-------------------|--------------|
| **Approval Gate** | Before high-impact action | Approve/Reject |
| **Correction Gate** | After generation, before publishing | Edit/Accept |
| **Escalation Gate** | When agent can't resolve | Human takes over |
| **Validation Gate** | After synthesis, before report | Verify accuracy |

### Implementation Pattern

```python
class HumanReviewWorkflow:
    def __init__(self, review_triggers):
        self.triggers = review_triggers
    
    async def process_with_review(self, agent_output, context):
        """Check if human review is needed and handle it."""
        
        # Check all triggers
        triggered = [
            trigger for trigger in self.triggers
            if trigger.should_activate(agent_output, context)
        ]
        
        if not triggered:
            return agent_output, "auto_approved"
        
        # Prepare review package
        review_package = ReviewPackage(
            agent_output=agent_output,
            triggers_activated=[t.name for t in triggered],
            context_summary=compress_for_human(context),
            suggested_action=agent_output.recommended_action,
            confidence=agent_output.confidence,
            provenance=agent_output.sources
        )
        
        # Submit and await human decision
        decision = await self.submit_for_review(review_package)
        
        return self.handle_decision(decision, agent_output)
    
    def handle_decision(self, decision, agent_output):
        if decision.action == "approve":
            return agent_output, "human_approved"
        elif decision.action == "modify":
            return self.reprocess_with_feedback(
                agent_output, decision.modifications
            )
        elif decision.action == "reject":
            return None, "human_rejected"
```

### Where to Place Review Gates

```
Multi-Agent Research System:
  Search → Analyze → Synthesize → [REVIEW GATE] → Report
                                         │
                                    Best placement:
                                    After synthesis (all info gathered)
                                    Before report (last chance to correct)

CI Pipeline:
  Generate → [REVIEW GATE for high-risk] → Deploy
                    │
               Only for: destructive ops, prod changes, security-sensitive

Code Generation:
  Plan → [REVIEW GATE] → Execute → Validate
              │
         Review the plan before expensive execution
```

---

## 4. Designing Review Triggers

### Trigger Categories

| Category | Signal | Example |
|----------|--------|---------|
| **High-Stakes** | Action has irreversible consequences | Database migration, production deploy |
| **Low-Confidence** | Agent's confidence below threshold | confidence < 0.6 |
| **Policy-Sensitive** | Action touches regulated domain | Healthcare data, financial transactions |
| **Anomaly** | Unusual or unexpected output | Result contradicts expectations |
| **Scope Creep** | Agent wants to do more than asked | Modifying files outside the original scope |

### Trigger Implementation

```python
class ReviewTrigger:
    """Base class for human review triggers."""
    
    def __init__(self, name, description):
        self.name = name
        self.description = description
    
    def should_activate(self, output, context) -> bool:
        raise NotImplementedError

class ConfidenceTrigger(ReviewTrigger):
    def __init__(self, threshold=0.6):
        super().__init__("low_confidence", "Output confidence below threshold")
        self.threshold = threshold
    
    def should_activate(self, output, context):
        return output.confidence < self.threshold

class DestructiveActionTrigger(ReviewTrigger):
    def __init__(self):
        super().__init__("destructive_action", "Potentially destructive operation")
        self.patterns = ["DROP", "DELETE", "TRUNCATE", "rm -rf", "force push"]
    
    def should_activate(self, output, context):
        return any(p in str(output) for p in self.patterns)

class ConflictTrigger(ReviewTrigger):
    def __init__(self, max_conflicts=2):
        super().__init__("unresolved_conflicts", "Multiple unresolved conflicts")
        self.max_conflicts = max_conflicts
    
    def should_activate(self, output, context):
        return len(output.conflicts) > self.max_conflicts

class DomainTrigger(ReviewTrigger):
    def __init__(self, sensitive_domains):
        super().__init__("sensitive_domain", "Sensitive domain detected")
        self.sensitive_domains = sensitive_domains  # ["medical", "legal", "financial"]
    
    def should_activate(self, output, context):
        return context.domain in self.sensitive_domains
```

### Trigger Composition

```python
# Configure triggers for the research system
review_workflow = HumanReviewWorkflow(
    review_triggers=[
        ConfidenceTrigger(threshold=0.6),
        ConflictTrigger(max_conflicts=2),
        DomainTrigger(sensitive_domains=["medical", "legal", "financial"]),
    ]
)

# Any ONE trigger activating is enough to request review
```

---

## 5. Information Provenance

### What is Provenance?

The complete record of where information came from and how it was transformed through the pipeline.

### Provenance Data Model

```python
@dataclass
class ProvenanceRecord:
    # ORIGIN
    source_url: str                    # Where the data came from
    source_title: str                  # Human-readable source name
    source_type: str                   # "web", "document", "database", "user_input"
    retrieval_timestamp: datetime      # When it was retrieved
    
    # EXTRACTION
    extracted_by: str                  # Which subagent extracted it
    extraction_method: str             # "full_read", "summary", "key_fact_extraction"
    extraction_confidence: float       # How confident the extraction was
    
    # TRANSFORMATION CHAIN
    transformations: list[Transformation]  # Each step that modified the data
    
    # CURRENT STATE
    current_form: str                  # "raw", "compressed", "synthesized", "cited"
    last_modified_by: str              # Last agent/process to touch it
    last_modified_at: datetime

@dataclass
class Transformation:
    step: str                # "extracted", "compressed", "synthesized", "formatted"
    agent: str               # Which agent performed it
    timestamp: datetime
    description: str         # What was done
    tokens_before: int       # Size before transformation
    tokens_after: int        # Size after transformation
```

### Provenance Through the Pipeline

```
┌──────────────────────────────────────────────────────────────────┐
│  PROVENANCE CHAIN                                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Stage 1: RETRIEVAL                                              │
│  ┌────────────────────────────────────────────┐                  │
│  │ source: "https://journal.org/study-2024"   │                  │
│  │ type: "academic_paper"                     │                  │
│  │ retrieved_at: "2024-01-15T10:30:00Z"       │                  │
│  │ retrieved_by: "web_search_subagent"        │                  │
│  └────────────────────┬───────────────────────┘                  │
│                       ▼                                           │
│  Stage 2: EXTRACTION                                             │
│  ┌────────────────────────────────────────────┐                  │
│  │ extracted_by: "doc_analysis_subagent"      │                  │
│  │ method: "key_fact_extraction"              │                  │
│  │ confidence: 0.92                           │                  │
│  │ finding: "Treatment X shows 40% improvement" │                │
│  └────────────────────┬───────────────────────┘                  │
│                       ▼                                           │
│  Stage 3: COMPRESSION                                            │
│  ┌────────────────────────────────────────────┐                  │
│  │ compressed_by: "coordinator"               │                  │
│  │ from: 3000 tokens → to: 500 tokens         │                  │
│  │ method: "structured_summary"               │                  │
│  │ preserved: [key_findings, confidence, source]│                │
│  └────────────────────┬───────────────────────┘                  │
│                       ▼                                           │
│  Stage 4: SYNTHESIS                                              │
│  ┌────────────────────────────────────────────┐                  │
│  │ synthesized_by: "synthesis_subagent"       │                  │
│  │ combined_with: [source_2, source_3]        │                  │
│  │ synthesis_confidence: 0.85                 │                  │
│  │ note: "consistent with 2 other sources"    │                  │
│  └────────────────────┬───────────────────────┘                  │
│                       ▼                                           │
│  Stage 5: REPORT                                                 │
│  ┌────────────────────────────────────────────┐                  │
│  │ formatted_by: "report_generation_subagent" │                  │
│  │ citation: "[1] Journal Study, 2024"        │                  │
│  │ final_confidence: 0.85                     │                  │
│  └────────────────────────────────────────────┘                  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. Why Provenance Matters

### Four Reasons (Exam-Relevant)

| Reason | Explanation | Example |
|--------|-------------|---------|
| **Auditability** | Trace any claim to its source | "This 40% figure came from Source A, extracted by doc_analysis, confirmed by Source B" |
| **Debugging** | Find where errors entered the pipeline | "The incorrect date was introduced during compression of Source C" |
| **Trust** | Users can verify claims independently | Report includes clickable citations to original sources |
| **Conflict Resolution** | Understand which sources disagree | "Source A says 40%, Source D says 15% — different methodologies" |

### Provenance Enables

```
Without Provenance:
  "Treatment X shows 40% improvement" — says who? When? How confident?

With Provenance:
  "Treatment X shows 40% improvement"
  [Source: journal.org/study-2024, extracted 2024-01-15, 
   confidence: 0.92, confirmed by 2 additional sources,
   synthesis confidence: 0.85]
```

---

## 7. Connecting to the Multi-Agent Research Scenario

### How All Three Concepts Work Together

```
RESEARCH PIPELINE WITH ALL THREE CONCEPTS:

1. Web Search (retrieval)
   └── Provenance: tag each result with source URL + retrieval time

2. Document Analysis (extraction)
   └── Provenance: tag findings with source + extraction confidence
   └── Compression: structure findings into standardized format

3. Coordinator (orchestration)
   └── Compression: compress analyses before passing to synthesis
   └── Provenance: maintain provenance metadata through compression
   └── Human Review: check if conflicts/low-confidence triggers review

4. Synthesis (integration)
   └── Provenance: track which sources support each conclusion
   └── Compression: produce concise synthesis from many inputs

5. [HUMAN REVIEW GATE]
   └── Trigger: conflicts > 2 OR confidence < 0.6 OR sensitive domain
   └── Package: compressed summary + provenance + conflicts

6. Report Generation (output)
   └── Provenance: inline citations with full attribution
   └── Compression: final output is already compressed (concise report)
```

### The Interplay

| Concept | Supports | How |
|---------|----------|-----|
| Compression → Human Review | Makes review feasible | Humans can't review 50K tokens; compress to ~2K |
| Provenance → Human Review | Informed decisions | Reviewer sees sources, confidence, transformation history |
| Provenance → Compression | Guides what to keep | High-provenance data preserved during compression |
| Human Review → Provenance | Records review decision | "Human approved synthesis at 2024-01-15T14:30Z" |
| Compression → Provenance | Creates transformation record | Each compression step logged in provenance chain |

---

## 8. Anti-Patterns

| Anti-Pattern | Why It Fails | Correct Pattern |
|-------------|--------------|-----------------|
| Compression without provenance | Lose track of sources | Always maintain source attribution through compression |
| Human review of full uncompressed context | Reviewer overwhelmed | Compress to review package with summary + highlights |
| No review triggers (review everything) | Pipeline too slow | Define specific triggers, auto-approve low-risk |
| No review triggers (review nothing) | High-stakes errors pass through | At minimum: confidence, conflicts, domain triggers |
| Provenance only at final stage | Can't debug intermediate steps | Track provenance at EVERY transformation |
| Dropping provenance during compression | Can't trace claims in report | Provenance metadata always travels with the data |
| Symmetric review gates | Equal scrutiny on low and high risk | Risk-proportional review intensity |
| Compressing recent context | Loses current working state | Always preserve last 3-5 turns uncompressed |

---

## 9. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│    COMPRESSION + HUMAN REVIEW + PROVENANCE                   │
├─────────────────────────────────────────────────────────────┤
│ COMPRESSION TECHNIQUES:                                      │
│   1. Summarization (verbose → concise, 70-90% reduction)    │
│   2. Key-fact extraction (keep claims, drop reasoning)       │
│   3. Drop low-relevance (remove non-contributing messages)   │
│                                                              │
│ COMPRESSION PRESERVES:                                       │
│   ✓ Key findings  ✓ Confidence  ✓ Source (provenance)       │
│   ✓ Decisions  ✓ Pending tasks                              │
│   ✗ Reasoning chains  ✗ Dead-end exploration                │
│                                                              │
│ HUMAN REVIEW TRIGGERS:                                       │
│   • Low confidence (< 0.6)                                   │
│   • Unresolved conflicts (> 2)                               │
│   • Sensitive domain (medical, legal, financial)             │
│   • Destructive actions (DROP, DELETE, deploy)               │
│   • Anomalous output (unexpected results)                    │
│                                                              │
│ HUMAN REVIEW ACTIONS:                                        │
│   Approve → continue pipeline                               │
│   Modify  → re-process with feedback                        │
│   Reject  → halt or escalate                                │
│                                                              │
│ PROVENANCE TRACKS:                                           │
│   source_url + extracted_by + confidence +                   │
│   transformations[] + current_form                           │
│                                                              │
│ PROVENANCE PURPOSE:                                          │
│   Auditability / Debugging / Trust / Conflict Resolution     │
│                                                              │
│ EXAM KEY:                                                    │
│   Provenance survives compression (metadata always kept)     │
│   Review gates go AFTER synthesis, BEFORE report             │
│   Triggers are risk-proportional (not one-size-fits-all)     │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge

### Practice Question

> In the Multi-Agent Research System, the coordinator needs to pass 12 document analyses to the synthesis subagent. Each analysis is ~3000 tokens. The synthesis subagent's effective input budget is 25,000 tokens. What is the correct approach?
>
> **A)** Pass all 12 analyses in full — 36,000 tokens will fit within the model's 200K context window
>
> **B)** Compress each analysis to a structured summary (~600 tokens each) preserving key findings, confidence, and source attribution, then pass all 12 compressed summaries
>
> **C)** Pass only the 8 most relevant analyses in full and drop the other 4
>
> **D)** Have the synthesis subagent request analyses one at a time in separate API calls

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: B**

**Explanation:**
- **B is correct** — Compressing to structured summaries (600 × 12 = 7,200 tokens) fits well within the 25K budget. Preserving key findings, confidence, and source attribution maintains the information needed for synthesis while respecting the budget. This is the standard context compression pattern.
- **A is wrong** — While 36K fits in the 200K context window technically, the question specifies the synthesis subagent's effective INPUT budget is 25K. Token budgeting accounts for system prompt, reasoning space, and output — not just the raw context window size.
- **C is wrong** — Dropping 4 analyses loses information without attempting compression. Always compress before discarding.
- **D is wrong** — Separate API calls lose the ability to cross-reference and synthesize across all findings in a single reasoning pass. Synthesis requires seeing all inputs together.

**Key Concepts Tested:**
- Context compression technique selection (Domain 5)
- Token budgeting vs. context window (Domain 5)
- Provenance preservation through compression (Domain 5)
- Compression > dropping (principle)

</details>

### Bonus Practice Question

> A human reviewer receives a research synthesis for approval. The synthesis includes a claim: "Drug X reduces symptoms by 60%." The reviewer wants to verify this claim. Which provenance information is most important?
>
> **A)** The timestamp when the synthesis was generated
>
> **B)** The source URL where the finding was extracted, the extraction confidence score, and whether other sources corroborate the claim
>
> **C)** The name of the synthesis subagent that produced the output
>
> **D)** The total token count of the original document

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: B**

**Explanation:**
- **B is correct** — To verify a specific claim, the reviewer needs: (1) the original source to check directly, (2) extraction confidence to assess reliability, and (3) corroboration status to understand consensus. This is exactly what provenance tracking provides.
- **A is wrong** — Timestamp tells when synthesis happened but doesn't help verify the claim's accuracy.
- **C is wrong** — The subagent's name is a system detail, not useful for claim verification.
- **D is wrong** — Document token count is irrelevant to verifying a specific factual claim.

**Key Concept:** Provenance enables human reviewers to make informed decisions by providing source + confidence + corroboration data.

</details>

---

## Key Takeaways for Exam Day

1. **Three compression techniques** — summarization, key-fact extraction, dropping low-relevance
2. **Compression preserves provenance** — source attribution ALWAYS survives compression
3. **Human review triggers are risk-proportional** — not everything needs review
4. **Review gates go after synthesis, before report** — maximum value, minimum disruption
5. **Provenance enables auditability + debugging + trust + conflict resolution**
6. **All three concepts interlock** — compression makes review feasible, provenance makes review informed
