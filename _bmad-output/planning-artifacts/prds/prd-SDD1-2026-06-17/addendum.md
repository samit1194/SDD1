# Addendum — AI Code Review Assistant MVP

This document captures technical depth, design decisions, code patterns, and mechanisms that support the PRD but belong outside the main document.

---

## A. LangGraph Architecture & State Management

### A.1 CodeReviewState Pydantic Model

```python
from typing import Annotated, Any
from pydantic import BaseModel
from langgraph.graph.message import add_messages

class SecurityIssue(BaseModel):
    severity: str  # "critical", "high", "medium", "low"
    issue: str
    line_number: int
    file: str
    recommendation: str
    cwe_id: str | None = None

class ReviewComment(BaseModel):
    line_number: int
    file: str
    comment: str
    category: str  # "style", "logic", "performance", "readability"

class TestSuggestion(BaseModel):
    test_type: str  # "unit", "integration", "e2e"
    description: str
    coverage_area: str

class CodeReviewState(BaseModel):
    git_diff: str
    pr_description: str
    
    # Intermediate results
    review_comments: Annotated[list[ReviewComment], add_messages] = []
    security_issues: Annotated[list[SecurityIssue], add_messages] = []
    test_suggestions: Annotated[list[TestSuggestion], add_messages] = []
    quality_metrics: dict = {}
    
    # Final output
    risk_score: float = 0.0
    final_report: dict = {}
```

### A.2 Graph Node Implementations

#### Input Processor
```python
def process_inputs(state: CodeReviewState) -> CodeReviewState:
    """Normalize and parse inputs"""
    state.git_diff = state.git_diff.strip()
    state.pr_description = state.pr_description.strip()
    
    # Extract metrics
    import re
    state.quality_metrics = {
        "files_changed": len(set(re.findall(r'^\+\+\+ b/(\S+)', state.git_diff, re.M))),
        "lines_added": len(re.findall(r'^\+(?!\+\+)', state.git_diff, re.M)),
        "lines_removed": len(re.findall(r'^\-(?!\-\-)', state.git_diff, re.M)),
    }
    return state
```

#### Security Analyzer (LLM-powered)
```python
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import PydanticOutputParser

def security_analyzer(state: CodeReviewState, llm: ChatOpenAI) -> CodeReviewState:
    """Detect security vulnerabilities"""
    parser = PydanticOutputParser(pydantic_object=SecurityIssue)
    
    prompt = f"""Analyze this git diff for security vulnerabilities.
    
Git Diff:
{state.git_diff}

Return a JSON list of security issues found.
Format: {parser.get_format_instructions()}"""
    
    chain = llm | parser
    issues = chain.invoke({"input": prompt})
    state.security_issues.extend(issues if isinstance(issues, list) else [issues])
    return state
```

#### Code Quality Analyzer
```python
def code_quality_analyzer(state: CodeReviewState, llm: ChatOpenAI) -> CodeReviewState:
    """Analyze code style and best practices"""
    parser = PydanticOutputParser(pydantic_object=ReviewComment)
    
    prompt = f"""Review this code for quality issues against generic best practices:
- Naming conventions (snake_case for Python, camelCase for JS)
- Function length (warn if >50 lines)
- Code duplication
- Missing docstrings
- Complexity indicators

Code changes:
{state.git_diff}

Provide specific comments."""
    
    chain = llm | parser
    comments = chain.invoke({"input": prompt})
    state.review_comments.extend(comments if isinstance(comments, list) else [comments])
    return state
```

#### Test Coverage Analyzer
```python
def test_coverage_analyzer(state: CodeReviewState, llm: ChatOpenAI) -> CodeReviewState:
    """Suggest test cases needed"""
    parser = PydanticOutputParser(pydantic_object=TestSuggestion)
    
    prompt = f"""Based on these code changes, what tests are missing?

Code changes:
{state.git_diff}

PR Context:
{state.pr_description}

Suggest specific unit, integration, or e2e tests."""
    
    chain = llm | parser
    tests = chain.invoke({"input": prompt})
    state.test_suggestions.extend(tests if isinstance(tests, list) else [tests])
    return state
```

#### Risk Scorer
```python
def risk_scorer(state: CodeReviewState) -> CodeReviewState:
    """Calculate overall risk score (0-100)"""
    risk_score = 0.0
    
    # Security weight: 40%
    critical_count = sum(1 for i in state.security_issues if i.severity == "critical")
    high_count = sum(1 for i in state.security_issues if i.severity == "high")
    risk_score += (critical_count * 20 + high_count * 10) * 0.4
    
    # Code quality weight: 30%
    quality_issues = len(state.review_comments)
    risk_score += min(quality_issues * 2, 30) * 0.3
    
    # Test coverage weight: 20%
    if state.quality_metrics.get("lines_added", 0) > 50:
        if len(state.test_suggestions) < 3:
            risk_score += 20 * 0.2
    
    # Changed files weight: 10%
    files_changed = state.quality_metrics.get("files_changed", 0)
    if files_changed > 5:
        risk_score += min(files_changed * 2, 10) * 0.1
    
    state.risk_score = min(risk_score, 100)
    return state
```

#### Aggregator & JSON Formatter
```python
from datetime import datetime
import json

def aggregator(state: CodeReviewState) -> CodeReviewState:
    """Format all findings into structured JSON report"""
    state.final_report = {
        "metadata": {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "files_analyzed": state.quality_metrics.get("files_changed", 0),
            "lines_changed": state.quality_metrics.get("lines_added", 0) + state.quality_metrics.get("lines_removed", 0),
        },
        "security_issues": [issue.dict() for issue in state.security_issues],
        "quality_comments": [comment.dict() for comment in state.review_comments],
        "test_suggestions": [test.dict() for test in state.test_suggestions],
        "risk_score": {
            "overall": round(state.risk_score, 1),
            "level": _risk_level(state.risk_score),
            "breakdown": {
                "security": round(sum(20 if i.severity == "critical" else 10 if i.severity == "high" else 5 if i.severity == "medium" else 2 for i in state.security_issues) * 0.4, 1),
                "quality": round(min(len(state.review_comments) * 2, 30) * 0.3, 1),
                "testing": round(20 * 0.2 if len(state.test_suggestions) < 3 else 0, 1),
                "volume": round(min(state.quality_metrics.get("files_changed", 0) * 2, 10) * 0.1, 1),
            }
        },
        "summary": f"{len(state.security_issues)} security issues found ({sum(1 for i in state.security_issues if i.severity in ['critical', 'high'])} high/critical). Risk level: {_risk_level(state.risk_score).upper()}."
    }
    return state

def _risk_level(score: float) -> str:
    if score >= 75: return "critical"
    if score >= 50: return "high"
    if score >= 25: return "medium"
    return "low"
```

### A.3 Graph Assembly & Execution

```python
from langgraph.graph import StateGraph
import os

def build_review_graph(llm: ChatOpenAI = None):
    """Build and compile the complete LangGraph workflow"""
    if llm is None:
        llm = ChatOpenAI(
            model="gpt-4o",
            temperature=0.3,
            max_tokens=2000,
            api_key=os.environ.get("OPENAI_API_KEY")
        )
    
    graph_builder = StateGraph(CodeReviewState)
    
    # Add nodes
    graph_builder.add_node("input_processor", process_inputs)
    graph_builder.add_node("security_analyzer", lambda state: security_analyzer(state, llm))
    graph_builder.add_node("quality_analyzer", lambda state: code_quality_analyzer(state, llm))
    graph_builder.add_node("test_analyzer", lambda state: test_coverage_analyzer(state, llm))
    graph_builder.add_node("risk_scorer", risk_scorer)
    graph_builder.add_node("aggregator", aggregator)
    
    # Define edges
    graph_builder.add_edge("__start__", "input_processor")
    
    # Parallel execution of analyzers
    graph_builder.add_edge("input_processor", "security_analyzer")
    graph_builder.add_edge("input_processor", "quality_analyzer")
    graph_builder.add_edge("input_processor", "test_analyzer")
    
    # Converge to risk scorer
    graph_builder.add_edge("security_analyzer", "risk_scorer")
    graph_builder.add_edge("quality_analyzer", "risk_scorer")
    graph_builder.add_edge("test_analyzer", "risk_scorer")
    
    graph_builder.add_edge("risk_scorer", "aggregator")
    graph_builder.add_edge("aggregator", "__end__")
    
    return graph_builder.compile()
```

---

## B. CLI Entry Point & Invocation

```python
import argparse
import json
import sys
from pathlib import Path

def main():
    parser = argparse.ArgumentParser(
        description="AI Code Review Assistant — Analyze Git diffs for security, quality, tests, and risk."
    )
    parser.add_argument("--diff", required=True, help="Path to .patch file or git diff")
    parser.add_argument("--pr-description", required=True, help="PR title + body (text or file path)")
    parser.add_argument("--output", default="review_report.json", help="Output JSON file (default: review_report.json)")
    parser.add_argument("--model", default="gpt-4o", help="OpenAI model (default: gpt-4o)")
    
    args = parser.parse_args()
    
    # Load inputs
    diff_path = Path(args.diff)
    if not diff_path.exists():
        print(f"Error: Diff file not found: {args.diff}", file=sys.stderr)
        sys.exit(1)
    
    git_diff = diff_path.read_text()
    
    # Handle PR description (text or file)
    pr_desc_path = Path(args.pr_description)
    if pr_desc_path.exists():
        pr_description = pr_desc_path.read_text()
    else:
        pr_description = args.pr_description
    
    # Build and run graph
    print("🔍 Building review graph...", file=sys.stderr)
    graph = build_review_graph()
    
    print("📊 Analyzing code...", file=sys.stderr)
    result = graph.invoke({
        "git_diff": git_diff,
        "pr_description": pr_description,
    })
    
    # Write JSON output
    output_path = Path(args.output)
    output_path.write_text(json.dumps(result.final_report, indent=2))
    print(f"✅ Review complete. Report saved to: {output_path}", file=sys.stderr)
    
    # Print human-readable summary
    report = result.final_report
    print("\n" + "="*60, file=sys.stderr)
    print(f"RISK SCORE: {report['risk_score']['overall']}/100 ({report['risk_score']['level'].upper()})", file=sys.stderr)
    print(f"Security Issues: {len(report['security_issues'])}", file=sys.stderr)
    print(f"Quality Comments: {len(report['quality_comments'])}", file=sys.stderr)
    print(f"Test Suggestions: {len(report['test_suggestions'])}", file=sys.stderr)
    print("="*60, file=sys.stderr)
    print(f"\n{report['summary']}", file=sys.stderr)

if __name__ == "__main__":
    main()
```

**Invocation Example**:
```bash
export OPENAI_API_KEY="sk-..."
python code_review.py --diff changes.patch --pr-description "Fix auth bug in login" --output review.json
```

---

## C. Dependencies & Setup

### C.1 Requirements
```txt
langchain>=0.1.0
langgraph>=0.0.1
langchain-openai>=0.0.1
pydantic>=2.0
python-dotenv>=1.0
```

### C.2 Quick Setup (2 hours)
1. Clone repo / create venv
2. `pip install -r requirements.txt`
3. `export OPENAI_API_KEY="sk-..."`
4. `python code_review.py --diff <path> --pr-description <text>`

---

## D. Evaluation & Metrics

### D.1 Evaluation Dimensions

| Dimension | Method | Success Criteria |
|-----------|--------|------------------|
| **Accuracy** | Manual comparison (5-10 test PRs) | ≥80% match vs manual review |
| **Security Detection** | Test with known vulns (hardcoded creds, injection) | Catches all critical issues |
| **Time Saved** | Before/after measurement | >50% faster than manual |
| **False Positives** | Engineer feedback | <20% false positives acceptable for MVP |
| **Reliability** | Run on diverse diffs | No crashes on valid input |

### D.2 Test Cases
1. **PR with hardcoded credential** → Detects critical security issue
2. **Large function (>100 lines)** → Flags readability issue
3. **Code with no tests** → Suggests unit tests
4. **Multiple file changes** → Scores aggregated risk
5. **Generic Python code** → Works without custom standards

---

## E. Post-MVP Roadmap (Phase 2+)

### Phase 2: Integration & UX
- GitHub PR comment integration
- Slack/Teams notifications
- Web dashboard for review history
- Custom coding standards (TOML/YAML config)

### Phase 3: Advanced Analysis
- RAG-based context retrieval (company-specific best practices)
- Cost optimization (batch processing, model selection)
- Performance optimization (streaming output, caching)

### Phase 4: Team Features
- Organization settings & role-based access
- Review history & analytics
- Integration with CI/CD pipelines
- Webhooks for automation

---

## F. Assumptions Validation Checklist

- [ ] OpenAI API key accessible in environment
- [ ] LangGraph + LangChain dependencies installable
- [ ] Git diff format is unified diff (standard `git diff` output)
- [ ] Generic best practices sufficient for MVP (no org-specific standards parsing)
- [ ] Response time <30 seconds acceptable
- [ ] 2-hour delivery achievable with provided code patterns
- [ ] Backend engineers can invoke from CLI without UI

