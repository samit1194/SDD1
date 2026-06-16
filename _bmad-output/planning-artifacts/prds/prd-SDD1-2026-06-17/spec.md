# AI Code Review Assistant — Technical Specification

**Version**: 1.0  
**Status**: FINAL  
**Date**: 2026-06-17  
**Audience**: Backend Engineers, Architects  

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [API Specification](#3-api-specification)
4. [Data Models](#4-data-models)
5. [Node Specifications](#5-node-specifications)
6. [Error Handling](#6-error-handling)
7. [Performance & Constraints](#7-performance--constraints)
8. [Security Considerations](#8-security-considerations)
9. [Testing Strategy](#9-testing-strategy)

---

## 1. System Overview

### 1.1 Purpose
Provide automated code review analysis for Git diffs using LangGraph orchestration and OpenAI LLM, delivering structured findings across security, quality, test coverage, and risk scoring dimensions.

### 1.2 Core Principles
- **Single Responsibility**: Each analyzer node handles one dimension
- **Stateless Processing**: All state managed via immutable CodeReviewState
- **LLM-as-Service**: OpenAI as external dependency (not embedded logic)
- **Deterministic Output**: Same input → same structured output (via Pydantic)
- **Fast Feedback**: <30 seconds end-to-end for typical PRs

### 1.3 Execution Model
```
Input (diff + PR desc)
  ↓
Input Processor (normalize)
  ↓
Parallel: [Security, Quality, Test analyzers]
  ↓
Risk Scorer (aggregate)
  ↓
Aggregator (format JSON)
  ↓
Output (review_report.json)
```

---

## 2. Architecture

### 2.1 System Components

```
┌─────────────────────────────────────────────────────┐
│  CLI Entry Point (code_review.py)                   │
│  - Argument parsing                                 │
│  - File I/O (diff, PR description)                  │
│  - Result serialization                             │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────┐
│  LangGraph Graph (StateGraph)                       │
│  ┌─────────────────────────────────────────────┐   │
│  │ Nodes:                                      │   │
│  │  - input_processor                          │   │
│  │  - security_analyzer (LLM)                  │   │
│  │  - quality_analyzer (LLM)                   │   │
│  │  - test_analyzer (LLM)                      │   │
│  │  - risk_scorer (deterministic)              │   │
│  │  - aggregator (formatter)                   │   │
│  └─────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────┐   │
│  │ State: CodeReviewState (Pydantic model)     │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────┐
│  OpenAI API (ChatOpenAI)                            │
│  - Model: gpt-4o (default) | gpt-4                 │
│  - Temperature: 0.3                                 │
│  - max_tokens: 2000                                 │
└─────────────────────────────────────────────────────┘
```

### 2.2 Technology Stack

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Orchestration | LangGraph | ≥0.0.1 | Graph-based workflow |
| LLM Chain | LangChain | ≥0.1.0 | LLM abstraction |
| LLM Provider | OpenAI API | gpt-4o | Analysis engine |
| State Model | Pydantic | ≥2.0 | Type validation |
| Runtime | Python | ≥3.9 | Execution |
| CLI | argparse | stdlib | Argument parsing |
| JSON | json | stdlib | Output serialization |

### 2.3 Deployment Model

```
Developer Machine
  ├─ Python venv
  ├─ Installed dependencies (langchain, langgraph, openai, pydantic)
  ├─ OPENAI_API_KEY environment variable
  └─ code_review.py (executable script)
```

---

## 3. API Specification

### 3.1 CLI Interface

#### Invocation
```bash
python code_review.py --diff <path> --pr-description <text-or-path> [--output <path>] [--model <model>]
```

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `--diff` | str (file path) | Yes | — | Path to `.patch` file or unified diff |
| `--pr-description` | str (text or path) | Yes | — | PR title + body (inline or file) |
| `--output` | str (file path) | No | `review_report.json` | Output JSON file path |
| `--model` | str | No | `gpt-4o` | OpenAI model ID (gpt-4o, gpt-4, etc.) |

#### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success; report written to output file |
| `1` | File not found, invalid input, or processing error |
| `2` | OpenAI API error (rate limit, auth, quota) |

#### Example Invocations

```bash
# Minimal
python code_review.py --diff changes.patch --pr-description "Fix login bug"

# With custom output
python code_review.py --diff changes.patch --pr-description "Fix login bug" --output custom_report.json

# With custom model
python code_review.py --diff changes.patch --pr-description "Fix login bug" --model gpt-4

# With PR description from file
python code_review.py --diff changes.patch --pr-description pr_body.txt
```

### 3.2 Output Schema

#### File: `review_report.json`

```typescript
{
  metadata: {
    timestamp: string (ISO 8601)      // e.g. "2026-06-17T10:30:00Z"
    files_analyzed: number             // Count of changed files
    lines_changed: number              // Total lines added + removed
  }
  
  security_issues: SecurityIssue[]
  quality_comments: ReviewComment[]
  test_suggestions: TestSuggestion[]
  
  risk_score: {
    overall: number (0-100)           // Aggregate risk
    level: "low" | "medium" | "high" | "critical"
    breakdown: {
      security: number (0-100)        // Security component score
      quality: number (0-100)         // Quality component score
      testing: number (0-100)         // Test coverage component score
      volume: number (0-100)          // Change volume component score
    }
  }
  
  summary: string                      // Human-readable conclusion
}
```

#### SecurityIssue Schema

```typescript
{
  severity: "critical" | "high" | "medium" | "low"
  issue: string                       // Issue description
  line_number: number                 // First affected line
  file: string                        // File path
  recommendation: string              // Remediation guidance
  cwe_id: string | null               // CWE identifier if applicable
}
```

#### ReviewComment Schema

```typescript
{
  line_number: number
  file: string
  comment: string
  category: "style" | "logic" | "performance" | "readability"
}
```

#### TestSuggestion Schema

```typescript
{
  test_type: "unit" | "integration" | "e2e"
  description: string
  coverage_area: string               // e.g. "api.py:handle_response()"
}
```

### 3.3 Stdout Output

Human-readable summary printed to stderr:
```
🔍 Building review graph...
📊 Analyzing code...
✅ Review complete. Report saved to: review_report.json

============================================================
RISK SCORE: 45.2/100 (MEDIUM)
Security Issues: 1
Quality Comments: 3
Test Suggestions: 2
============================================================

1 critical security issue found. Risk level: MEDIUM. Recommend addressing credential exposure before merge.
```

---

## 4. Data Models

### 4.1 CodeReviewState

**Pydantic Model** — single source of truth for all workflow state.

```python
from pydantic import BaseModel
from typing import Annotated, Any

class CodeReviewState(BaseModel):
    # --- Inputs ---
    git_diff: str                       # Unified diff content
    pr_description: str                 # PR title + body
    
    # --- Intermediate Results ---
    review_comments: Annotated[
        list[ReviewComment], 
        add_messages
    ] = []
    security_issues: Annotated[
        list[SecurityIssue], 
        add_messages
    ] = []
    test_suggestions: Annotated[
        list[TestSuggestion], 
        add_messages
    ] = []
    quality_metrics: dict = {}          # {files_changed, lines_added, lines_removed}
    
    # --- Final Output ---
    risk_score: float = 0.0             # Aggregate 0-100
    final_report: dict = {}             # Complete JSON report
```

### 4.2 Sub-Models

**SecurityIssue**
```python
class SecurityIssue(BaseModel):
    severity: str                       # "critical", "high", "medium", "low"
    issue: str
    line_number: int
    file: str
    recommendation: str
    cwe_id: str | None = None
```

**ReviewComment**
```python
class ReviewComment(BaseModel):
    line_number: int
    file: str
    comment: str
    category: str                       # "style", "logic", "performance", "readability"
```

**TestSuggestion**
```python
class TestSuggestion(BaseModel):
    test_type: str                      # "unit", "integration", "e2e"
    description: str
    coverage_area: str
```

### 4.3 State Immutability

- All state mutations return a new `CodeReviewState`
- No in-place modifications; graph tracks state flow
- Pydantic validation runs on every state transition
- List fields use `add_messages` for proper aggregation in LangGraph

---

## 5. Node Specifications

### 5.1 Input Processor Node

**Purpose**: Normalize inputs, extract metadata, prepare state for analyzers.

**Input State**: `git_diff`, `pr_description`  
**Output State**: `quality_metrics`  
**Side Effects**: None (pure function)

**Algorithm**:
```
1. Strip whitespace from git_diff and pr_description
2. Parse diff for changed files:
   - Regex: `^\+\+\+ b/(\S+)` → file names
   - Count unique files
3. Count lines:
   - Lines added: `^\+(?!\+\+)`
   - Lines removed: `^\-(?!\-\-)`
4. Store in state.quality_metrics
```

**Error Handling**:
- If diff is empty: warn but continue (qualifies as "no changes")
- If PR description is empty: use placeholder text

---

### 5.2 Security Analyzer Node

**Purpose**: Detect hardcoded credentials, injections, insecure patterns.

**Input State**: `git_diff`, `pr_description`  
**Output State**: `security_issues` (appended)  
**LLM Call**: Yes (GPT-4o or GPT-4)

**Prompt Template**:
```
Analyze this git diff for security vulnerabilities.

LOOK FOR:
- Hardcoded API keys, passwords, tokens
- SQL injection risks (unescaped queries)
- Command injection risks (shell execution)
- Unvalidated user input handling
- Overly permissive access controls
- Use-after-free or memory safety issues
- Unencrypted sensitive data

Git Diff:
{git_diff}

PR Context:
{pr_description}

Return a JSON list of SecurityIssue objects.
Format: {parser.get_format_instructions()}

If no issues found, return empty list [].
```

**LLM Parameters**:
- Model: `gpt-4o` (default) or `gpt-4` for high-sensitivity reviews
- Temperature: 0.3 (deterministic)
- max_tokens: 2000

**Output Validation**:
- Pydantic parser ensures list[SecurityIssue]
- Fallback: If LLM returns non-JSON, parse error triggers node failure

**Severity Mapping**:
- Critical: Active exploitation (hardcoded creds, RCE, auth bypass)
- High: Likely exploitable (injection risks, weak crypto)
- Medium: Potential impact (missing validation, unsafe patterns)
- Low: Best-practice violations (logging, error handling)

---

### 5.3 Quality Analyzer Node

**Purpose**: Check code against generic best practices (style, structure, readability).

**Input State**: `git_diff`, `pr_description`  
**Output State**: `review_comments` (appended)  
**LLM Call**: Yes

**Prompt Template**:
```
Review this code for quality issues against generic best practices.

CHECK FOR:
- Naming conventions (snake_case for Python, camelCase for JS)
- Function length (warn if >50 lines)
- Code complexity (nested conditionals, cyclomatic complexity)
- Missing docstrings or inadequate documentation
- Code duplication (repeated logic)
- Unused variables or imports
- Type hints (if applicable)

Git Diff:
{git_diff}

Return a JSON list of ReviewComment objects.
Format: {parser.get_format_instructions()}

If no issues found, return empty list [].
```

**Categories**:
- `style`: Naming, formatting, conventions
- `logic`: Control flow, algorithm issues
- `performance`: Inefficient patterns, O(n²) loops
- `readability`: Clarity, verbosity, comments

**LLM Parameters**: Same as Security (temp=0.3, max_tokens=2000)

---

### 5.4 Test Analyzer Node

**Purpose**: Identify untested code paths and recommend test cases.

**Input State**: `git_diff`, `pr_description`  
**Output State**: `test_suggestions` (appended)  
**LLM Call**: Yes

**Prompt Template**:
```
Based on these code changes, what tests are missing?

ANALYZE FOR:
- New functions without unit tests
- Error handling changes (need exception tests)
- Database mutations (need rollback verification)
- Changed control flow (need path coverage)
- Edge cases (boundary conditions, null handling)

Git Diff:
{git_diff}

PR Context:
{pr_description}

Return a JSON list of TestSuggestion objects.
Format: {parser.get_format_instructions()}

If coverage appears adequate, return empty list [].
```

**Test Types**:
- `unit`: Single function or class
- `integration`: Component interaction, DB, external APIs
- `e2e`: Full workflow across system boundary

---

### 5.5 Risk Scorer Node

**Purpose**: Aggregate all findings into a single risk score (0-100).

**Input State**: `security_issues`, `review_comments`, `test_suggestions`, `quality_metrics`  
**Output State**: `risk_score`  
**LLM Call**: No (deterministic calculation)

**Algorithm**:
```
risk_score = 0.0

# Security component (40% weight)
critical_count = count(security_issues where severity == "critical")
high_count = count(security_issues where severity == "high")
security_component = (critical_count * 20 + high_count * 10) * 0.4
risk_score += security_component

# Quality component (30% weight)
quality_component = min(len(review_comments) * 2, 30) * 0.3
risk_score += quality_component

# Test coverage component (20% weight)
if lines_added > 50 and len(test_suggestions) < 3:
    test_component = 20 * 0.2
    risk_score += test_component

# Change volume component (10% weight)
if files_changed > 5:
    volume_component = min(files_changed * 2, 10) * 0.1
    risk_score += volume_component

risk_score = min(risk_score, 100)  # Cap at 100

# Risk level mapping
if risk_score >= 75: level = "critical"
elif risk_score >= 50: level = "high"
elif risk_score >= 25: level = "medium"
else: level = "low"
```

**Constraints**:
- Score always in [0, 100]
- Proportional weighting ensures no single dimension dominates
- Conservative defaults (e.g., missing tests only count if >50 lines added)

---

### 5.6 Aggregator Node

**Purpose**: Format all findings into structured JSON report.

**Input State**: All previous outputs  
**Output State**: `final_report`  
**LLM Call**: No

**Output Format**: See § 3.2 (API Specification)

---

## 6. Error Handling

### 6.1 Input Validation

| Condition | Behavior |
|-----------|----------|
| Diff file not found | Exit code 1, clear error message |
| PR description file not found | Exit code 1 |
| Diff is empty | Continue with warning; no files analyzed |
| Diff format invalid | Attempt parse; may miss some changes |
| PR description is empty | Use placeholder; proceed |

### 6.2 LLM Failures

| Error | Behavior |
|-------|----------|
| OpenAI auth failure (bad API key) | Exit code 2, print API error |
| Rate limit (429) | Exit code 2, suggest retry |
| Token limit exceeded | Exit code 1, analyze smaller diff |
| Timeout (>30s) | Exit code 2, suggestion to retry |
| Non-JSON response | Log error, return empty list for that analyzer |

### 6.3 Pydantic Validation Failures

If LLM returns malformed JSON:
```python
try:
    issues = parser.invoke(...)  # Pydantic parser
except ValidationError as e:
    logger.error(f"Validation failed: {e}")
    state.security_issues = []  # Empty result for this node
    # Continue (fail-safe)
```

### 6.4 Graph Execution Failures

If a node raises unhandled exception:
- LangGraph halts graph execution
- Exception propagates to CLI
- Exit code 1, error message printed

---

## 7. Performance & Constraints

### 7.1 Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| End-to-end time | <30 seconds | Measured to JSON output |
| Diff parsing | <1s | Regex operations |
| LLM latency | <25s | Network + inference |
| Output serialization | <1s | JSON encoding |

### 7.2 Scalability Limits

| Input | Limit | Behavior |
|-------|-------|----------|
| Diff size | <50KB | Larger diffs slow; >100KB may hit token limits |
| Files changed | <20 | Performance degrades with more files |
| Lines changed | <1000 | LLM prompt becomes very long |
| PR description | <5000 chars | Reasonable context window |

### 7.3 Token Usage Estimation

**Per PR review** (typical):
- Input tokens: ~1500 (diff + context + prompt template)
- Output tokens: ~500 (findings)
- **Total**: ~6000 tokens (3 analyzers × 2000 tokens)
- **Cost**: ~$0.02 at GPT-4o pricing (Jan 2025)

---

## 8. Security Considerations

### 8.1 API Key Management

- **Requirement**: `OPENAI_API_KEY` environment variable
- **Never**: Hardcode keys, log keys, commit to repo
- **Validation**: Check key existence before graph execution

```python
if not os.environ.get("OPENAI_API_KEY"):
    print("Error: OPENAI_API_KEY not set", file=sys.stderr)
    sys.exit(1)
```

### 8.2 Input Sanitization

- Diffs are read as-is (no filtering)
- PR descriptions are read as-is
- **Risk**: LLM may see secrets in PR description (acceptable; LLM doesn't store)
- **Mitigation**: User should not include secrets in PR descriptions

### 8.3 Output Security

- JSON report written to disk
- No encryption (local file system assumed trusted)
- No sensitive data in final report (except what was in diff/PR)

### 8.4 LLM Prompt Injection

- User input (diff, PR description) embedded directly in prompts
- **Risk**: Malicious diff could trick LLM into ignoring instructions
- **Mitigation**: Structured output parsing (Pydantic) validates results; malicious results ignored

---

## 9. Testing Strategy

### 9.1 Unit Tests

**Test Input Processor**:
```python
def test_input_processor_counts_files():
    state = CodeReviewState(git_diff="...", pr_description="...")
    result = process_inputs(state)
    assert result.quality_metrics["files_changed"] == 2
```

**Test Risk Scorer**:
```python
def test_risk_scorer_critical():
    state = CodeReviewState(security_issues=[...])
    result = risk_scorer(state)
    assert result.risk_score >= 75
    assert result.risk_score <= 100
```

### 9.2 Integration Tests (LLM)

**Test Security Analyzer**:
```python
def test_security_analyzer_detects_hardcoded_cred():
    state = CodeReviewState(git_diff="+password = 'admin123'", ...)
    result = security_analyzer(state, llm)
    assert len(result.security_issues) >= 1
    assert any("hardcoded" in issue.issue.lower() for issue in result.security_issues)
```

**Test End-to-End**:
```python
def test_full_graph():
    graph = build_review_graph()
    result = graph.invoke({
        "git_diff": sample_diff,
        "pr_description": "Fix login",
    })
    assert result.final_report["risk_score"]["overall"] >= 0
    assert result.final_report["risk_score"]["overall"] <= 100
```

### 9.3 Test Cases (Manual)

| Test Case | Input | Expected Behavior |
|-----------|-------|-------------------|
| **Hardcoded credential** | `password = "admin123"` | Critical security issue |
| **SQL injection** | `query = f"SELECT * FROM users WHERE id={id}"` | High security issue |
| **Long function** | 100-line function | Code quality comment |
| **No tests** | New 100-line function | Test suggestion |
| **Multiple issues** | Complex diff | Multiple findings across dimensions |
| **Empty diff** | `""` | Runs; no issues found |

### 9.4 Evaluation Metrics

| Metric | Method | Target |
|--------|--------|--------|
| Accuracy | Compare vs manual review (5+ PRs) | >80% match |
| False Positives | Count non-issues reported | <20% |
| False Negatives | Count missed issues | <10% |
| Speed | Clock execution time | <30 seconds |
| Reliability | Run on diverse diffs | 0 crashes on valid input |

---

## 10. Future Extensions (Phase 2+)

### 10.1 Planned Features

- **RAG Integration**: Vector store of company standards
- **GitHub Integration**: Post comments directly to PR
- **Batch Processing**: Review multiple PRs concurrently
- **Custom Standards**: Load org-specific rules from TOML/YAML
- **Review History**: Database of past reviews for analytics
- **Slack Notifications**: Alert on critical issues

### 10.2 API Stability

- **Breaking Changes**: Avoid changes to CLI interface and JSON schema
- **Backwards Compatibility**: Support multiple model versions
- **Deprecation**: 2-release notice before removing features

---

## Appendix A: Glossary

| Term | Definition |
|------|-----------|
| **Unified Diff** | Standard format for `git diff` output (RFC 3881) |
| **LLM** | Large Language Model (OpenAI GPT-4o) |
| **Pydantic** | Python data validation library |
| **LangGraph** | Graph-based workflow orchestration framework |
| **State** | Immutable data structure tracking analysis progress |
| **Node** | Single transformation or analysis step in the graph |
| **Edge** | Control flow connection between nodes |

---

## Appendix B: References

- [OpenAI API Documentation](https://platform.openai.com/docs)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [Pydantic Documentation](https://docs.pydantic.dev/latest/)
- [Unified Diff Format (RFC 3881)](https://tools.ietf.org/html/rfc3881)

---

**End of Specification**

---
