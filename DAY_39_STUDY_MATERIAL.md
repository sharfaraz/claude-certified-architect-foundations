# Day 39: Scenario Deep-Dive — Multi-Agent Research System

---

| Field | Detail |
|-------|--------|
| **Domain** | Domain 1: Agentic Architecture & Orchestration (27%) + Domain 5: Context Management & Reliability (15%) |
| **Sprint** | Sprint 3 — Advanced Systems (Days 31–45) |
| **Academy Course** | Building with Claude API Pod 5 |
| **Estimated Study Time** | 2–3 hours |

---

## Table of Contents

1. [Research System Architecture Overview](#1-research-system-architecture-overview)
2. [The Four Subagents in Detail](#2-the-four-subagents-in-detail)
3. [Coordinator Role & Responsibilities](#3-coordinator-role--responsibilities)
4. [Orchestration Topology](#4-orchestration-topology)
5. [allowedTools Scoping per Subagent](#5-allowedtools-scoping-per-subagent)
6. [Communication Flow & Data Contracts](#6-communication-flow--data-contracts)
7. [Error Handling in the Research Pipeline](#7-error-handling-in-the-research-pipeline)
8. [Anti-Patterns](#8-anti-patterns)
9. [Quick Reference Card](#9-quick-reference-card)
10. [Scenario Challenge](#10-scenario-challenge)

---

## Recommended Video & Reading Resources

| Resource | Link |
|----------|------|
| Claude Agent SDK — Subagents | https://code.claude.com/docs/en/agent-sdk/subagents |
| Claude Agent SDK — Agent Loop | https://code.claude.com/docs/en/agent-sdk/agent-loop |
| Multi-Agent Orchestration Patterns | https://docs.anthropic.com/en/docs/agents |
| Claude API Messages Reference | https://docs.anthropic.com/en/api/messages |

---

## 1. Research System Architecture Overview

The Multi-Agent Research System is one of 6 exam scenarios (4 randomly selected per exam). It tests your ability to design coordinator-subagent architectures for knowledge-intensive tasks.

### System Purpose

A user submits a research query. The system autonomously:
1. Searches the web for relevant sources
2. Analyzes individual documents for key findings
3. Synthesizes findings into a coherent analysis
4. Generates a formatted report with citations

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        COORDINATOR                               │
│  (dispatch, route, aggregate, decide, escalate)                  │
└──────┬──────────┬──────────────┬──────────────┬─────────────────┘
       │          │              │              │
       ▼          ▼              ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐
│Web Search│ │Doc Analyze│ │Synthesis │ │Report Gen    │
│ Subagent │ │ Subagent  │ │ Subagent │ │ Subagent     │
└──────────┘ └──────────┘ └──────────┘ └──────────────┘
```

### Key Exam Insight

This scenario tests Domain 1 (27%) and Domain 5 (15%) simultaneously — you must demonstrate both architectural orchestration and context management strategies.

---

## 2. The Four Subagents in Detail

### 2.1 Web Search Subagent

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Find relevant sources for the research query |
| **Input** | Research query + search constraints (date range, domains) |
| **Output** | Structured list: URLs, titles, snippets, relevance scores |
| **allowedTools** | `web_search`, `url_fetch` only |
| **Max turns** | 5 (prevents infinite search expansion) |

```python
web_search_agent = Agent(
    name="web_search",
    instructions="Search for academic and authoritative sources. Return structured results with URLs, titles, and relevance scores.",
    allowedTools=["web_search", "url_fetch"],
    max_turns=5
)
```

### 2.2 Document Analysis Subagent

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Extract key findings from individual documents |
| **Input** | Document text + analysis focus (what to look for) |
| **Output** | Structured analysis: key findings, supporting quotes, confidence |
| **allowedTools** | `read_document`, `extract_text`, `summarize` |
| **Max turns** | 3 (per document; prevents over-processing) |

```python
doc_analysis_agent = Agent(
    name="document_analysis",
    instructions="Extract key findings, supporting evidence, and assess confidence. Flag incomplete analysis.",
    allowedTools=["read_document", "extract_text", "summarize"],
    max_turns=3
)
```

### 2.3 Synthesis Subagent

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Combine findings from multiple documents into coherent analysis |
| **Input** | Array of structured document analyses |
| **Output** | Unified analysis with themes, conflicts, and confidence levels |
| **allowedTools** | `compare_findings`, `identify_themes` |
| **Max turns** | 2 (synthesis is a focused operation) |

### 2.4 Report Generation Subagent

| Attribute | Detail |
|-----------|--------|
| **Purpose** | Format final report with proper citations |
| **Input** | Synthesized analysis + source metadata |
| **Output** | Formatted report (markdown/PDF) with inline citations |
| **allowedTools** | `format_report`, `generate_citations`, `export_document` |
| **Max turns** | 2 |

---

## 3. Coordinator Role & Responsibilities

The coordinator is the **brain** of the system. It does NOT perform research itself — it orchestrates.

### Five Coordinator Actions (Exam Key Terms)

| Action | Description | Example |
|--------|-------------|---------|
| **Dispatch** | Send initial query to web search subagent | "Search for: climate adaptation strategies 2023-2024" |
| **Route** | Direct documents to the analysis subagent | "Analyze document #3, focus on economic impacts" |
| **Pass** | Forward results from one subagent to another | Pass analyses → synthesis subagent |
| **Trigger** | Initiate the next pipeline stage | Trigger report generation after synthesis |
| **Escalate** | Hand off to human when issues arise | Conflicting findings with equal confidence |

### Coordinator Decision Logic

```python
coordinator = Agent(
    name="research_coordinator",
    instructions="""
    You coordinate a research pipeline:
    1. DISPATCH: Send query to web_search subagent
    2. ROUTE: Send each document to doc_analysis subagent
    3. ASSESS: Check if enough quality sources analyzed
       - If < 3 high-confidence analyses → dispatch more searches
       - If conflicting findings → flag for human review
    4. PASS: Forward analyses to synthesis subagent
    5. TRIGGER: Start report generation with synthesis output
    
    ESCALATE when:
    - All sources conflict with no clear resolution
    - Document analysis fails after 3 retries
    - Synthesis confidence < 0.5
    """,
    allowedTools=["dispatch_task", "route_task", "escalate_to_human"]
)
```

---

## 4. Orchestration Topology

### Sequential Pipeline with Parallel Fan-Out

This is the exam-tested topology for the research scenario:

```
                    ┌─── Doc Analysis (doc1) ───┐
                    │                            │
Search ──→ Results ─┼─── Doc Analysis (doc2) ───┼──→ Synthesis ──→ Report
                    │                            │
                    └─── Doc Analysis (doc3) ───┘
                    
         Sequential      Parallel Fan-Out        Sequential Pipeline
```

### Why This Topology?

| Phase | Pattern | Rationale |
|-------|---------|-----------|
| Search → Analysis | Sequential | Can't analyze docs until search finds them |
| Multiple Doc Analysis | Parallel fan-out | Documents are independent — analyze concurrently |
| Analysis → Synthesis | Fan-in (aggregation) | Must collect all analyses before synthesizing |
| Synthesis → Report | Sequential | Report depends on synthesis output |

### Implementation Pattern

```python
# Coordinator orchestration logic
async def run_research(query: str):
    # Phase 1: Sequential - Web Search
    search_results = await dispatch(web_search_agent, query)
    
    # Phase 2: Parallel Fan-Out - Document Analysis
    analysis_tasks = [
        route(doc_analysis_agent, doc) 
        for doc in search_results.documents
    ]
    analyses = await asyncio.gather(*analysis_tasks)
    
    # Phase 3: Sequential - Synthesis (with compression)
    compressed = compress_analyses(analyses)  # Token budgeting
    synthesis = await pass_to(synthesis_agent, compressed)
    
    # Phase 4: Sequential - Report Generation
    report = await trigger(report_agent, synthesis)
    
    return report
```

---

## 5. allowedTools Scoping per Subagent

### The Principle of Least Privilege

Each subagent gets ONLY the tools it needs. This is a critical exam concept.

| Subagent | Allowed Tools | Explicitly Denied |
|----------|--------------|-------------------|
| Web Search | `web_search`, `url_fetch` | `write_file`, `execute_code` |
| Doc Analysis | `read_document`, `extract_text`, `summarize` | `web_search`, `write_file` |
| Synthesis | `compare_findings`, `identify_themes` | `web_search`, `read_document` |
| Report Gen | `format_report`, `generate_citations`, `export_document` | All other tools |

### Why Scoping Matters

1. **Safety** — prevents a subagent from performing unintended actions
2. **Focus** — subagent can't get distracted by irrelevant capabilities
3. **Auditability** — clear boundaries for what each agent can/cannot do
4. **Exam testing** — expect questions asking which tools should be scoped to which subagent

---

## 6. Communication Flow & Data Contracts

### Structured Handoffs Between Subagents

Subagents don't communicate directly — all communication flows through the coordinator.

```
Web Search → Coordinator:
{
  "sources": [
    {"url": "...", "title": "...", "snippet": "...", "relevance": 0.92}
  ],
  "total_found": 15,
  "query_used": "climate adaptation economic impact"
}

Coordinator → Doc Analysis:
{
  "document_url": "...",
  "analysis_focus": "economic impacts of climate adaptation",
  "max_findings": 5
}

Doc Analysis → Coordinator:
{
  "key_findings": ["...", "..."],
  "confidence": 0.85,
  "supporting_quotes": [...],
  "source_url": "...",
  "completeness": "full"  // or "partial"
}
```

---

## 7. Error Handling in the Research Pipeline

### The Error-Context Feedback Pattern

When a subagent returns incomplete or failed results:

```python
# Anti-pattern: blind retry
result = await doc_analysis_agent.run(document)
if result.completeness == "partial":
    result = await doc_analysis_agent.run(document)  # Same input = same problem

# Correct: error-context feedback with circuit breaker
result = await doc_analysis_agent.run(document)
retries = 0
while result.completeness == "partial" and retries < 3:
    retries += 1
    result = await doc_analysis_agent.run(
        document,
        context={
            "previous_result": result,
            "error_feedback": "Previous analysis was incomplete. Missing: conclusions section.",
            "instruction": "Focus on the missing sections. Start from where you left off."
        }
    )
```

### Key Error Handling Rules

| Situation | Action | Rationale |
|-----------|--------|-----------|
| Subagent returns partial results | Error-context feedback + retry (max 3) | Gives subagent info to improve |
| Subagent fails completely | Escalate to coordinator for reassignment | May need different approach |
| Multiple subagents conflict | Flag for human review | Automated resolution risks bias |
| Token limit approached | Compress and checkpoint | Preserve progress, reduce input |

---

## 8. Anti-Patterns

| Anti-Pattern | Why It Fails | Correct Pattern |
|-------------|--------------|-----------------|
| Blind retry (same input, same context) | Produces same failure | Error-context feedback with details about what went wrong |
| No max_turns guard | Agent loops forever | Set max_turns per subagent (3-5 typical) |
| Coordinator performs research directly | Violates separation of concerns | Coordinator only orchestrates |
| Subagent has all tools | Violates least privilege, loses focus | Scope allowedTools narrowly |
| Synthesis fills gaps from "knowledge" | Introduces hallucination | Only synthesize from provided sources |
| Skipping failed documents silently | Data loss without recovery attempt | Retry with feedback, then log and document the gap |
| No circuit breaker on retries | Infinite loops, cost explosion | max_turns + escalation path |

---

## 9. Quick Reference Card

```
┌─────────────────────────────────────────────────────────┐
│            MULTI-AGENT RESEARCH SYSTEM                    │
├─────────────────────────────────────────────────────────┤
│ TOPOLOGY: Sequential pipeline + Parallel fan-out         │
│                                                          │
│ SUBAGENTS:                                              │
│   Web Search   → allowedTools: [web_search, url_fetch]  │
│   Doc Analysis → allowedTools: [read_doc, extract, sum] │
│   Synthesis    → allowedTools: [compare, themes]        │
│   Report Gen   → allowedTools: [format, cite, export]   │
│                                                          │
│ COORDINATOR ACTIONS:                                     │
│   dispatch / route / pass / trigger / escalate           │
│                                                          │
│ ERROR HANDLING:                                          │
│   Error-context feedback + max_turns=3 (NOT blind retry) │
│                                                          │
│ ESCALATION TRIGGERS:                                     │
│   • Conflicting findings with equal confidence           │
│   • Subagent fails after max retries                    │
│   • Synthesis confidence < threshold                     │
│                                                          │
│ KEY EXAM DISTINCTION:                                    │
│   Coordinator orchestrates, never performs research      │
│   Subagents execute within scoped boundaries             │
└─────────────────────────────────────────────────────────┘
```

---

## 10. Scenario Challenge

### Practice Question (From Exam SPEC)

> You are designing a Multi-Agent Research System where a coordinator delegates to four subagents: web search, document analysis, synthesis, and report generation. The document analysis subagent occasionally returns incomplete results when processing very long documents. What is the best architectural approach to handle this?
>
> **A)** Have the coordinator retry the document analysis subagent with the same input until it succeeds
>
> **B)** Have the coordinator pass the incomplete results along with the original document context back to the document analysis subagent with error-context feedback, and set a max_turns limit of 3 for retries
>
> **C)** Have the synthesis subagent handle incomplete results by filling in gaps from its own knowledge
>
> **D)** Skip documents that produce incomplete results and proceed with whatever analysis is available

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: B**

**Explanation:**
- **B is correct** — Error-context feedback gives the subagent the information it needs to improve on the next attempt (it knows what was incomplete and can focus there). The max_turns=3 limit acts as a circuit breaker to prevent infinite retries.
- **A is wrong** — Blind retry with the same input is a classic anti-pattern. If the document is too long, the same approach will produce the same incomplete result.
- **C is wrong** — Having the synthesis subagent fill gaps introduces hallucination risk. The synthesis subagent should only work with provided source material.
- **D is wrong** — Silently dropping data without attempting recovery means research gaps in the final report.

**Key Concepts Tested:**
- Error-context feedback vs. blind retry (Domain 1)
- Circuit breaker pattern / max_turns (Domain 1)
- Subagent boundary responsibilities (Domain 1)
- Information completeness and reliability (Domain 5)

</details>

### Bonus Practice Question

> In the research system, the coordinator dispatches a search query and receives 8 document URLs. It needs to analyze all 8 documents. Which orchestration pattern is most appropriate for the document analysis phase?
>
> **A)** Sequential — analyze doc 1, then doc 2, then doc 3... through doc 8
>
> **B)** Parallel fan-out — dispatch all 8 to the document analysis subagent concurrently
>
> **C)** Hierarchical — create two sub-coordinators, each handling 4 documents
>
> **D)** Single batch — send all 8 documents to one analysis call

<details>
<summary>Click to reveal answer</summary>

**Correct Answer: B**

**Explanation:**
- **B is correct** — Document analyses are independent of each other (no document depends on another's analysis). Parallel fan-out maximizes throughput while maintaining correctness.
- **A is wrong** — Sequential processing adds unnecessary latency when tasks are independent.
- **C is wrong** — Hierarchical delegation adds complexity without benefit when tasks are simple and independent.
- **D is wrong** — Sending all 8 documents in one call risks exceeding context limits and violates the single-responsibility principle of the subagent design.

</details>

---

## Key Takeaways for Exam Day

1. **Coordinator orchestrates, never performs** — it dispatches, routes, passes, triggers, escalates
2. **Error-context feedback > blind retry** — always pass failure context to the retried subagent
3. **max_turns is a circuit breaker** — prevents infinite loops, typically set to 3-5
4. **allowedTools enforces least privilege** — each subagent gets only what it needs
5. **Parallel fan-out for independent tasks** — document analysis is the canonical example
6. **Sequential for dependencies** — search before analyze, analyze before synthesize
