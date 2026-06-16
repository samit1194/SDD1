---
title: AI Code Review Assistant — MVP
status: final
created: 2026-06-17
updated: 2026-06-17
project: SDD1
author: Samit
audience: Backend Engineers (Internal)
scope: MVP — 2-hour delivery
---

# AI Code Review Assistant — MVP

## 1. Vision

Reduce time spent on manual code reviews by providing backend engineers with **instant, AI-powered analysis** of Git diffs covering security, quality, test coverage, and risk scoring. Engineers trigger reviews on-demand; the tool delivers structured JSON output they can read or integrate downstream.

**Success Metric**: Time saved per review vs. manual inspection.

---

## 2. User & Context

- **Users**: Backend engineers on the team
- **Trigger**: On-demand (command-line invocation)
- **Input**: Git diff + PR description (text or file)
- **Output**: JSON report + human-readable summary
- **Deployment**: Local system (Python script)
- **Environment**: Generic Linux/Windows/macOS with Python 3.9+

---

## 3. Core Capabilities (MVP)

### 3.1 Security Detection
Identify hardcoded credentials, injection vulnerabilities, and insecure patterns:
- Hardcoded API keys, passwords, tokens
- SQL and command injection risks
- Unvalidated user input handling
- Overly permissive access controls

**Output**: Issues with severity (critical/high/medium/low), line number, file, and remediation guidance.

### 3.2 Code Quality Analysis
Check against generic best practices for naming, structure, and readability:
- Inconsistent naming conventions (snake_case for Python, camelCase for JS)
- Functions exceeding 50 lines
- High cyclomatic complexity
- Missing or inadequate documentation
- Code duplication patterns

**Output**: Comments with category (style/logic/performance/readability), file, and line number.

### 3.3 Test Coverage Suggestions
Identify untested code paths and recommend missing test cases:
- New functions lacking unit tests
- Error handling changes without integration tests
- Database mutations without rollback verification
- Uncovered edge cases

**Output**: Suggestions with test type (unit/integration/e2e), description, and coverage area.

### 3.4 Risk Scoring
Aggregate all findings into a single **risk score (0–100)** with weighted components:
- Security issues: 40% (critical = 20pts, high = 10pts)
- Code quality issues: 30%
- Test coverage gaps: 20%
- Change volume: 10% (files and lines modified)

**Output**: Numeric score plus risk level (low/medium/high/critical).

---

## 4. Input & Output Specification

### 4.1 Input Format
**Invocation**:
```bash
python code_review.py --diff <path> --pr-description <text-or-path> [--output <path>]
```

**Parameters**:
- `--diff`: Path to `.patch` file or unified git diff output
- `--pr-description`: PR title and description (text string or file path)
- `--output`: Output JSON file path (default: `review_report.json`)
- [ASSUMPTION] No custom standards file in MVP; defaults to generic best practices

### 4.2 Output Format
**Primary Output**: `review_report.json`
```json
{
  "metadata": {
    "timestamp": "2026-06-17T10:30:00Z",
    "files_analyzed": 3,
    "lines_changed": 42
  },
  "security_issues": [
    {
      "severity": "critical",
      "issue": "Hardcoded API key detected",
      "line_number": 15,
      "file": "src/api.py",
      "recommendation": "Use environment variables or secrets manager"
    }
  ],
  "quality_comments": [
    {
      "line_number": 28,
      "file": "src/api.py",
      "category": "readability",
      "comment": "Function is 67 lines — consider breaking into smaller units"
    }
  ],
  "test_suggestions": [
    {
      "test_type": "unit",
      "description": "Test error handling for invalid API responses",
      "coverage_area": "api.py:handle_response()"
    }
  ],
  "risk_score": {
    "overall": 45,
    "level": "medium",
    "breakdown": {
      "security": 20,
      "quality": 15,
      "testing": 10,
      "volume": 0
    }
  },
  "summary": "1 critical security issue found. Risk level: MEDIUM. Recommend addressing credential exposure before merge."
}
```

**Secondary Output**: Human-readable text summary to stdout.

---

## 5. Technical Approach

### 5.1 Architecture
- **Orchestrator**: LangGraph StateGraph (manages workflow nodes)
- **LLM Provider**: OpenAI (ChatOpenAI, GPT-4o for analysis)
- **State Management**: Typed Pydantic models (CodeReviewState)
- **Execution**: Sequential + parallel analyzer nodes
- **Output Formatter**: Structured JSON (Pydantic serialization)

### 5.2 Analyzer Nodes
1. **Input Processor**: Parse diff, extract files/functions, normalize text
2. **Security Analyzer**: LLM-powered detection of vulnerabilities
3. **Quality Analyzer**: LLM-powered style/structure feedback
4. **Test Analyzer**: LLM-powered coverage gap identification
5. **Risk Scorer**: Aggregate findings into single score
6. **Aggregator**: Format and output JSON report

### 5.3 LLM Integration
- **Model**: GPT-4o (or GPT-4 for security-critical nodes)
- **API Key**: Via `OPENAI_API_KEY` environment variable
- **Prompts**: Structured templates with output schemas (Pydantic models)
- **Temperature**: 0.3 (deterministic output)

---

## 6. Scope & Constraints

### 6.1 MVP In Scope
✅ All 4 pillars (security, quality, tests, risk scoring)  
✅ JSON output + human-readable summary  
✅ Local Python CLI invocation  
✅ On-demand triggering  
✅ Generic best practices (no custom standards parsing)  

### 6.2 Out of Scope (Phase 2+)
❌ Web UI or GitHub App integration  
❌ Historical review database  
❌ Custom coding standards configuration  
❌ Automated GitHub comment posting  
❌ Large diff optimization (>10K lines)  
❌ Team settings and access control  

### 6.3 Core Assumptions
- [ASSUMPTION] OpenAI API key available in `OPENAI_API_KEY` environment variable
- [ASSUMPTION] Git diffs in unified format (standard `git diff` output)
- [ASSUMPTION] Generic best practices adequate for MVP (no org-specific rules)
- [ASSUMPTION] False positives acceptable; engineers manually filter
- [ASSUMPTION] No authentication required (internal tool, trusted users)
- [ASSUMPTION] Response time <30 seconds acceptable

---

## 7. Success Criteria

| Metric | Target | Validation |
|--------|--------|-----------|
| **Delivery** | 2 hours | Implementation clock |
| **Usability** | Single-command invocation | CLI test |
| **Feature Completeness** | All 4 pillars functional | Manual test on sample diff |
| **Accuracy** | Detects real security/quality issues | Compare vs manual review (5+ test PRs) |
| **Time Savings** | >50% faster than manual | Measure engineer review cycles |

---

## 8. Implementation Notes

- **LLM-Only Approach**: MVP uses OpenAI LLM analysis without RAG for faster delivery; standards embedded in prompts
- **Single Output File**: All results consolidated in `review_report.json` for simplicity
- **No Historical Storage**: Reviews are ephemeral; deferred to Phase 2 if needed
- **Manual Filtering**: Engineers review AI findings and filter; feedback improves prompts over time

---

## 9. Phase 2+ Roadmap

- **Integration**: GitHub PR comment posting, Slack/Teams notifications
- **Configuration**: Custom coding standards (TOML/YAML), team settings
- **Analytics**: Review history dashboard, trend analysis
- **Efficiency**: Batch processing, cost optimization, caching
- **Scale**: Role-based access, organization management
