# Day 40: Scenario Deep-Dive — Multi-Agent Research System (Advanced Patterns)

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 1: Agentic Architecture & Orchestration (27%) + Domain 5: Context Management & Reliability (15%) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Building with Claude API Pod 5 |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Context Management Across Subagents](#1-context-management-across-subagents)
2. [Token Budgeting Strategies](#2-token-budgeting-strategies)
3. [Context Compression Techniques](#3-context-compression-techniques)
4. [Information Provenance Tracking](#4-information-provenance-tracking)
5. [Human Review Workflows](#5-human-review-workflows)
6. [Handling Conflicting Findings](#6-handling-conflicting-findings)
7. [Coordinator Decision Logic](#7-coordinator-decision-logic)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Agent SDK — Context Management | https://code.claude.com/docs/en/agent-sdk/context |
| Anthropic Docs — Long Context | https://docs.anthropic.com/en/docs/build-with-claude/long-context |
| Claude Agent SDK — Human-in-the-Loop | https://code.claude.com/docs/en/agent-sdk/human-review |
| Claude API Token Counting | https://docs.anthropic.com/en/api/messages-count-tokens |

---

## 1. Context Management Across Subagents

### The Core Challenge

As research progresses, context grows: search results + document analyses + synthesis = potentially exceeds context window limits.

### Context Flow Through the Pipeline

```
Query (small)
  ↓
Search Results (medium: ~2K tokens per source × 10 sources = 20K)
  ↓
Document Analyses (large: ~3K tokens per analysis × 10 docs = 30K)
  ↓
Combined for Synthesis (potentially 50K+ tokens)
  ↓
Synthesis Output (medium: ~5K tokens)
  ↓
Report (final output: ~3K tokens)
```

### The Problem

The synthesis subagent receives ALL document analyses. With 10-15 documents, this easily exceeds practical context limits — even if technically within the model's window, quality degrades with very long inputs.

### The Solution: Managed Context Boundaries

```python
# Each subagent interaction is a FRESH context
# The coordinator decides what context each subagent receives

# BAD: Pass everything
synthesis_input = {
    "full_analyses": all_15_analyses  # 45K+ tokens → quality issues
}

# GOOD: Compressed, structured context
synthesis_input = {
    "compressed_analyses": [
        compress_to_summary(analysis) for analysis in all_15_analyses
    ],  # ~500 tokens each = 7.5K total
    "conflicts": identified_conflicts,
    "metadata": source_provenance_map
}
```

---

## 2. Token Budgeting Strategies

### Allocating Context Window Space

Token budgeting means deliberately planning how much of the context window each component uses.

| Component | Budget Allocation | Rationale |
|-----------|------------------|-----------|
| System prompt | 5-10% | Agent instructions, role definition |
| Input context (from coordinator) | 40-50% | The actual data to process |
| Working space for reasoning | 30-40% | Room for Claude to think and generate |
| Output tokens | 10-15% | The structured response |

### Budget Calculation Example

For a 200K token context window model:

```
Total available: 200,000 tokens
System prompt:       10,000 (5%)
Agent instructions:   5,000 (2.5%)
Input data budget:   90,000 (45%)   ← This is what you can pass to the subagent
Reasoning space:     75,000 (37.5%)
Output budget:       20,000 (10%)
```

### Dynamic Token Budgeting

```python
def calculate_budget(model_context_limit, system_prompt_tokens, desired_output_tokens):
    """Calculate available budget for input data."""
    reserved_for_reasoning = model_context_limit * 0.35
    available_for_input = (
        model_context_limit 
        - system_prompt_tokens 
        - reserved_for_reasoning 
        - desired_output_tokens
    )
    return available_for_input

# Example: How many analyses can the synthesis subagent handle?
budget = calculate_budget(
    model_context_limit=200000,
    system_prompt_tokens=5000,
    desired_output_tokens=15000
)
# budget = 200000 - 5000 - 70000 - 15000 = 110000 tokens for input

# Each compressed analysis = ~500 tokens
max_analyses = budget // 500  # ~220 compressed analyses fit comfortably
```

### Exam Insight

Token budgeting is NOT about `max_tokens` (which controls output length). It's about managing how much INPUT context you send to each subagent call.

---

## 3. Context Compression Techniques

### When to Compress

| Trigger | Action | Example |
|---------|--------|---------|
| Between pipeline phases | Compress outputs before passing downstream | Analyses → compressed summaries before synthesis |
| Approaching token limits | Summarize older context | Drop low-relevance messages |
| Parallel fan-in | Compress before aggregation | 15 full analyses → 15 structured summaries |
| Before human review | Summarize for readability | Full context → executive summary for reviewer |

### Compression Strategies

#### Strategy 1: Structured Summarization

```python
# Convert detailed analysis to structured summary
def compress_analysis(full_analysis):
    return {
        "source": full_analysis["source_url"],
        "key_findings": full_analysis["key_findings"][:3],  # Top 3 only
        "confidence": full_analysis["confidence"],
        "relevance_to_query": full_analysis["relevance_score"],
        "one_sentence_summary": generate_summary(full_analysis)
    }
# Full analysis: ~3000 tokens → Compressed: ~500 tokens (83% reduction)
```

#### Strategy 2: Key-Fact Extraction

```python
# Extract only the facts, discard reasoning/evidence
def extract_key_facts(analysis):
    return {
        "facts": [
            {"claim": f["claim"], "confidence": f["confidence"]}
            for f in analysis["findings"]
            if f["confidence"] > 0.7  # Drop low-confidence findings
        ],
        "source": analysis["source_url"]
    }
```

#### Strategy 3: Tiered Compression

```python
# Different compression levels based on relevance
def tiered_compress(analyses, query_relevance_threshold=0.8):
    high_relevance = []
    low_relevance = []
    
    for a in analyses:
        if a["relevance_score"] >= query_relevance_threshold:
            high_relevance.append(compress_analysis(a))  # Light compression
        else:
            low_relevance.append(extract_key_facts(a))  # Heavy compression
    
    return {"detailed": high_relevance, "summary_only": low_relevance}
```

### Compression Quality Principles

1. **Preserve key findings** — never lose the primary conclusions
2. **Maintain provenance** — always keep source attribution
3. **Keep confidence scores** — downstream agents need to assess reliability
4. **Structure the output** — compressed data should be more structured, not less

---

## 4. Information Provenance Tracking

### What is Provenance?

Tracking which sources contributed to which conclusions throughout the pipeline.

### Why Provenance Matters (Exam Tested)

| Reason | Explanation |
|--------|-------------|
| **Auditability** | Can trace any claim in the report back to its source |
| **Debugging** | If the report contains an error, identify which subagent/source introduced it |
| **Trust** | Users can verify claims by checking original sources |
| **Conflict resolution** | When findings conflict, provenance reveals the sources behind each claim |

### Provenance Implementation

```python
# Every piece of data carries its provenance metadata
class ProvenanceRecord:
    source_url: str
    source_title: str
    extraction_timestamp: str
    subagent_that_extracted: str
    confidence: float
    transformation_history: list[str]  # ["extracted", "compressed", "synthesized"]

# Example: A finding in the final report
{
    "claim": "Climate adaptation costs will increase 40% by 2030",
    "provenance": {
        "source_url": "https://example.org/climate-report-2024",
        "extracted_by": "doc_analysis_subagent",
        "synthesized_by": "synthesis_subagent",
        "transformations": ["raw_extract", "compression", "synthesis"],
        "original_quote": "Our models project a 40% increase...",
        "confidence": 0.88
    }
}
```

### Provenance Through the Pipeline

```
Source Document → Doc Analysis (extracts, tags source)
     ↓
Coordinator (passes provenance metadata alongside findings)
     ↓
Synthesis (merges findings, maintains source attribution per claim)
     ↓
Report (inline citations, source list, confidence indicators)
```

---

## 5. Human Review Workflows

### Approval Gate Pattern

Insert human review between synthesis and report generation — the highest-value checkpoint.

```
Search → Analyze → Synthesize → [HUMAN REVIEW] → Report
                                       │
                                       ├── Approve → Continue to report
                                       ├── Reject → Re-synthesize with feedback
                                       └── Escalate → Full human takeover
```

### When to Trigger Human Review

| Trigger | Threshold | Rationale |
|---------|-----------|-----------|
| Low synthesis confidence | confidence < 0.6 | Uncertain conclusions need human judgment |
| Conflicting sources | conflict_count > 2 | Automated resolution risks bias |
| High-stakes topic | domain ∈ [medical, legal, financial] | Policy-sensitive decisions |
| Novel/unexpected findings | deviation_from_expected > 0.5 | May indicate errors or breakthroughs |
| Insufficient sources | quality_sources < 3 | Not enough evidence for confident synthesis |

### Implementation Pattern

```python
async def run_with_human_review(synthesis_result):
    # Check if human review is needed
    if needs_human_review(synthesis_result):
        # Prepare review package
        review_package = {
            "synthesis_summary": synthesis_result.summary,
            "confidence": synthesis_result.confidence,
            "conflicts": synthesis_result.conflicts,
            "sources_used": synthesis_result.provenance,
            "review_reason": determine_review_reason(synthesis_result)
        }
        
        # Submit for human review
        human_decision = await submit_for_review(review_package)
        
        if human_decision.action == "approve":
            return await trigger(report_agent, synthesis_result)
        elif human_decision.action == "reject":
            # Re-synthesize with human feedback
            return await pass_to(synthesis_agent, {
                "original": synthesis_result,
                "human_feedback": human_decision.feedback
            })
        elif human_decision.action == "escalate":
            return {"status": "escalated", "reason": human_decision.reason}
    else:
        # Auto-proceed to report generation
        return await trigger(report_agent, synthesis_result)
```

---

## 6. Handling Conflicting Findings

### Conflict Detection

When two document analyses contradict each other:

```python
def detect_conflicts(analyses):
    conflicts = []
    for i, a1 in enumerate(analyses):
        for a2 in analyses[i+1:]:
            for f1 in a1["findings"]:
                for f2 in a2["findings"]:
                    if contradicts(f1, f2):
                        conflicts.append({
                            "finding_1": f1,
                            "source_1": a1["source_url"],
                            "finding_2": f2,
                            "source_2": a2["source_url"],
                            "severity": assess_severity(f1, f2)
                        })
    return conflicts
```

### Conflict Resolution Strategies

| Strategy | When to Use | Who Decides |
|----------|-------------|-------------|
| Confidence-weighted | One source clearly more authoritative | Coordinator (automated) |
| Recency-weighted | Newer data supersedes older | Coordinator (automated) |
| Present both | Both sources equally valid | Synthesis subagent |
| Escalate to human | High-stakes or equal confidence | Human reviewer |

### Coordinator Decision Tree for Conflicts

```
Conflict detected
  ├── Confidence gap > 0.3 → Use higher-confidence finding
  ├── Same confidence, different dates → Use more recent source
  ├── Same confidence, same date → Present both in synthesis
  └── High-stakes domain → Escalate to human reviewer
```

---

## 7. Coordinator Decision Logic

### When to Stop Searching

```python
def should_stop_searching(state):
    return (
        state.high_quality_sources >= 5 or
        state.search_iterations >= 3 or
        state.no_new_relevant_results_for >= 2 or  # Diminishing returns
        state.total_sources >= 15  # Hard cap for context budget
    )
```

### When to Escalate

```python
def should_escalate(state):
    return (
        state.synthesis_confidence < 0.5 or
        state.unresolved_conflicts > 2 or
        state.subagent_failures >= 3 or
        state.domain in HIGH_STAKES_DOMAINS
    )
```

### When to Compress

```python
def should_compress(state):
    return (
        state.total_analysis_tokens > state.synthesis_budget * 0.8 or
        state.analyses_count > 10 or  # Many analyses = compress before synthesis
        state.approaching_phase_transition  # Between stages
    )
```

---

## 8. Anti-Patterns

| Anti-Pattern | Why It Fails | Correct Pattern |
|-------------|--------------|-----------------|
| Passing full context to every subagent | Exceeds token limits, degrades quality | Compress and scope context per subagent |
| No provenance tracking | Can't audit or debug the final report | Tag every finding with its source |
| Auto-resolving all conflicts | Introduces bias without human judgment | Escalate ambiguous conflicts |
| Compressing too aggressively | Loses critical information | Preserve key findings + confidence + source |
| Skipping human review for speed | High-stakes errors pass through | Insert review gates at critical checkpoints |
| max_tokens confusion | max_tokens controls OUTPUT, not input capacity | Manage INPUT context through compression |
| Synthesis from "general knowledge" | Hallucination — not grounded in sources | Synthesize ONLY from provided analyses |

---

## 9. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│     MULTI-AGENT RESEARCH: ADVANCED PATTERNS                  │
├─────────────────────────────────────────────────────────────┤
│ TOKEN BUDGETING:                                             │
│   System prompt: 5-10%                                       │
│   Input context: 40-50%                                      │
│   Reasoning space: 30-40%                                    │
│   Output: 10-15%                                             │
│                                                              │
│ COMPRESSION TRIGGERS:                                        │
│   • Between pipeline phases                                  │
│   • Approaching token limits (>80% budget)                   │
│   • Before passing to downstream subagents                   │
│   • Fan-in aggregation (many → one)                          │
│                                                              │
│ COMPRESSION PRESERVES:                                       │
│   ✓ Key findings  ✓ Confidence  ✓ Source attribution         │
│   ✗ Full reasoning  ✗ Low-relevance details                  │
│                                                              │
│ HUMAN REVIEW TRIGGERS:                                       │
│   • Confidence < 0.6                                         │
│   • Unresolved conflicts > 2                                 │
│   • High-stakes domain                                       │
│   • Novel/unexpected findings                                │
│                                                              │
│ PROVENANCE = source_url + extractor + transformations        │
│                                                              │
│ EXAM KEY: max_tokens ≠ context budget                        │
│   max_tokens = output length limit                           │
│   Context budget = managed via compression of INPUT          │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge

### Practice Question (From Exam SPEC)

> The Multi-Agent Research System's coordinator has collected analyses from 15 documents. The combined analysis text exceeds the synthesis subagent's context window. What is the most effective approach?
>
> **A)** Increase the model's max_tokens parameter to accommodate the larger input
>
> **B)** Have the coordinator compress each document analysis into a structured summary (key findings, confidence, source) before passing them to the synthesis subagent
>
> **C)** Split the analyses into two batches and run the synthesis subagent twice, then merge the results
>
> **D)** Drop the least relevant analyses until the remaining ones fit within the context window

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: B**

**Explanation:**
- **B is correct** — Context compression through structured summarization preserves the essential information while reducing token count. The structured format (key findings, confidence, source) maintains information provenance. This is the standard pattern for managing context between pipeline phases.
- **A is wrong** — `max_tokens` controls OUTPUT length, not input context capacity. This is a common exam distractor that tests whether you understand the difference.
- **C is wrong** — Splitting into batches may produce inconsistent syntheses (each batch lacks the full picture). The synthesis step needs visibility into ALL findings to identify themes and conflicts.
- **D is wrong** — Dropping analyses loses information without attempting compression. You should always try to preserve data through compression before discarding it.

**Key Concepts Tested:**
- Token budgeting (Domain 5)
- Context compression techniques (Domain 5)
- max_tokens misconception (Domain 4/5 crossover)
- Information provenance preservation (Domain 5)

</details>

### Bonus Practice Question

> The research system's synthesis subagent identifies that two highly-cited sources directly contradict each other on a key finding, both with confidence scores above 0.85. The research topic is pharmaceutical efficacy. What should the coordinator do?
>
> **A)** Use the more recent source and discard the older one
>
> **B)** Average the findings and present a middle-ground conclusion
>
> **C)** Escalate to human review with both sources, the conflict details, and provenance metadata
>
> **D)** Have the synthesis subagent resolve the conflict by weighing additional contextual factors

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: C**

**Explanation:**
- **C is correct** — This is a high-stakes domain (pharmaceutical) with a high-confidence conflict that cannot be automatically resolved without risking patient safety. Escalation with full provenance enables an expert human to make the judgment call.
- **A is wrong** — Recency doesn't guarantee correctness, especially in pharmaceutical research where older landmark studies may be more rigorous.
- **B is wrong** — Averaging contradictory findings is scientifically invalid. The truth isn't always in the middle.
- **D is wrong** — In high-stakes domains, automated conflict resolution introduces unacceptable risk. This requires human expert judgment.

**Key Concepts Tested:**
- Escalation triggers (Domain 1)
- High-stakes domain handling (Domain 5)
- Provenance for human review (Domain 5)
- Anti-pattern: automated resolution of ambiguous high-stakes conflicts

</details>

---

## Key Takeaways for Exam Day

1. **max_tokens ≠ context budget** — max_tokens limits output; context budget is managed via input compression
2. **Compress between phases** — always compress before passing data to the next pipeline stage
3. **Provenance travels with data** — every compressed summary retains source attribution
4. **Human review for high-stakes + low-confidence + conflicts** — don't auto-resolve what requires judgment
5. **Token budgeting is proactive** — plan how much space each component gets BEFORE running the pipeline
6. **Coordinator decides when to stop, compress, and escalate** — these are architectural decisions, not subagent decisions
