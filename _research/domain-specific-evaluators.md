# R6: Domain-Specific Evaluators

**Version**: 1.0
**Date**: 2026-03-02
**Source**: [otel-v3-research-directions.md](otel-v3-research-directions.md) R6, white paper Section 8.2
**Status**: Done — RAG and agent domain evaluators shipped; `DomainEvaluator` composite class, factory functions, 47 tests
**Verification Date**: 2026-03-02 — DeepEval, RAGAS, BFCL, Arize Phoenix, Langfuse, Datadog, Opik docs verified

---

## 1. Introduction

### 1.1 Domain Evaluation Landscape (2025-2026)

The LLM evaluation ecosystem has shifted from general-purpose metrics to domain-specialized evaluators across three axes:

1. **Agent/tool-use evaluation** — DeepEval v3.8.8 (Dec 2025) shipped six agent-specific metrics. RAGAS expanded to 30+ metrics covering RAG, agents, and general NLG. BFCL V4 (ICML 2025) provides holistic agentic evaluation.
2. **Domain benchmarks** — Code (SWE-bench Verified, BigCodeBench), medical (MedQA, MedHELM), legal (LegalBench, LegalBench-RAG) have matured but remain benchmark-centric without production evaluation pipelines.
3. **Evaluation composition** — Panel judges (SE-Jury, PoLL), meta-evaluation, and CI/CD pipeline integration have matured, with multi-judge ensembles approaching human-level correlation.

### 1.2 Scope

This document covers: DeepEval domain evaluators, code/medical/legal evaluation, RAG-specific metrics, agent evaluation, composition patterns, and platform support. It does not prescribe specific domain implementations.

---

## 2. DeepEval Domain Evaluators

### 2.1 Release Timeline (2025)

| Version | Date | Key Addition |
|---------|------|--------------|
| v3.1.9 | Jun 2025 | ArenaGEval pairwise metric |
| v3.2.6 | Jul 2025 | Multi-turn dataset support; conversational goldens |
| v3.5.9 | Sep 2025 | MCP metrics suite |
| v3.6.2-3.6.9 | Oct 2025 | Agent metrics: Goal Accuracy, Topic Adherence, Plan Adherence, Tool Use, Step Efficiency |
| v3.7.0-3.7.2 | Nov 2025 | ExactMatchMetric, PatternMatchMetric, Anthropic integration, CrewAI tracing |
| v3.8.8 | Dec 2025 | Six agentic metrics (see below) |

Source: [deepeval.com/changelog/changelog-2025](https://deepeval.com/changelog/changelog-2025)

### 2.2 DAGMetric (Deep Acyclic Graph Metric)

The most versatile custom metric in DeepEval: deterministic control over LLM-as-Judge via decision trees.

- Traverses a custom acyclic decision tree in topological order; each node uses an evaluation model to determine the path
- **Node types**: `TaskNode` (data transformation), `BinaryJudgementNode` (true/false), `NonBinaryJudgementNode` (multi-class), `VerdictNode` (leaf score, 0-10)
- Reduces hallucination in evaluation by decomposing criteria into atomic units
- Extended to `ConversationalDAGMetric` (Sep 2025) for multi-turn workflows with cycle detection
- Docs: [deepeval.com/docs/metrics-dag](https://deepeval.com/docs/metrics-dag)

### 2.3 Agent-Specific Evaluators (v3.8.8)

All six are LLM-as-Judge, configurable threshold, strict mode, async mode:

| Metric | Scope | Measures |
|--------|-------|----------|
| Task Completion | End-to-end | Whether the agent accomplished the intended task |
| Tool Correctness | Component | Whether the right tools were selected |
| Argument Correctness | Component | Whether tool parameters are valid |
| Step Efficiency | End-to-end | No unnecessary or redundant steps |
| Plan Adherence | End-to-end | Whether the agent followed its own plan |
| Plan Quality | End-to-end | Whether the plan was logical, complete, efficient |

Source: [deepeval.com/guides/guides-ai-agent-evaluation](https://deepeval.com/guides/guides-ai-agent-evaluation)

### 2.4 MCP Evaluation (Sep 2025)

| Metric | Description |
|--------|-------------|
| `MCPUseMetric` | Evaluates which MCP primitives were called and argument correctness |
| `MultiTurnMCPUseMetric` | Conversational variant for multi-turn MCP sessions |
| `MCPTaskCompletionMetric` | Whether the MCP agent accomplished the end task |

Separate: [MCPEval](https://arxiv.org/abs/2507.12806) (arXiv) proposes automatic MCP-based deep evaluation independent of DeepEval.

Source: [deepeval.com/docs/metrics-mcp-use](https://deepeval.com/docs/metrics-mcp-use)

### 2.5 ArenaGEval (v3.1.9)

Pairwise comparative evaluation — first DeepEval metric to compare two test cases and select the better output. Uses blinded position swapping to reduce position bias. Suitable for A/B model comparison and prompt iteration ranking.

---

## 3. Code Generation Evaluators

### 3.1 Benchmark Current State

| Benchmark | Description | Best Score | Status |
|-----------|-------------|------------|--------|
| **HumanEval** | 164 function-level tasks | >90% Pass@1 (frontier) | Saturated |
| **MBPP** | 374 entry-level Python tasks | Near-solved | Retired as differentiator |
| **EvalPlus** | HumanEval x80 tests, MBPP x35 tests | 79.9% (MBPP+) | Active; rigorous augmentation |
| **BigCodeBench** | 1,140 tasks, 139 libraries | ~61% (GPT-4o) | Active, ICLR 2025 |
| **SWE-bench Verified** | 500 validated GitHub issues | 76.1% (Verdent, Feb 2026) | Active; v2.0.2 Feb 2026 |
| **HumanEval Pro** | Self-invoking compositional code | 76.2% (o1-mini) | New, ACL 2025 |
| **LiveCodeBench** | Live competitive programming | Ongoing | Anti-contamination design |
| **CodeJudgeBench** | 5,352 LLM-as-Judge code eval samples | Judges, not models | New, ASE 2025 |

Sources: [epoch.ai/benchmarks/swe-bench-verified](https://epoch.ai/benchmarks/swe-bench-verified), [bigcode-bench.github.io](https://bigcode-bench.github.io/)

### 3.2 Code-Specific Metrics

| Metric | Approach | Limitation |
|--------|----------|------------|
| **CodeBLEU** | BLEU + AST similarity + data-flow graph | Quality inversely correlates with code length |
| **CodeBERTScore** | Cosine similarity via CodeBERT encoder | Better semantic capture than CodeBLEU |
| **Pass@k** | Execution-based test pass rate | Gold standard; requires test harness |
| **LLM-as-Judge** | CodeJudge, SE-Jury | Approaching human correlation |

**SE-Jury** (ASE 2025): Five distinct judge strategies with dynamic team selection. Narrows gap with human evaluation for software engineering tasks.

**OTel gap**: No published standard for emitting code eval metrics as OTel spans or semantic conventions for code-specific evaluation.

---

## 4. Medical/Clinical Evaluators

### 4.1 Benchmarks

| Benchmark | Description | Best Score |
|-----------|-------------|------------|
| **MedQA** (USMLE) | USMLE-style multiple-choice | ~91% (MedGemma) |
| **MedMCQA** | 194K Indian medical licensing | Active |
| **PubMedQA** | Biomedical research QA | Active |
| **MedHELM** | 120+ scenarios, 22 health categories | Stanford CRFM |

Models failing <60% on MedQA are considered unqualified for clinical assessment. ACL 2025 paper ("Questioning Our Questions") found benchmarks fail to capture patient communication, longitudinal care, and clinical information extraction.

### 4.2 HIPAA-Compliant Evaluation

Production approaches (2025-2026):
- **Vendor BAAs**: OpenAI, Anthropic, AWS Bedrock, Google Vertex
- **PHI sanitization**: Hybrid regex + BERT-based NER before data enters evaluation systems ([arXiv:2504.17669](https://arxiv.org/abs/2504.17669))
- **De-identification**: MIMIC-style tooling before JSONL/trace storage
- No purpose-built HIPAA-compliant evaluation platform exists

---

## 5. Legal Domain Evaluators

### 5.1 LegalBench

- 162 tasks, 40 contributors (Stanford/Hazy Research)
- Covers legal reasoning, rule-recall, issue-spotting, interpretation, rhetoric
- Dataset: [huggingface.co/datasets/nguha/legalbench](https://huggingface.co/datasets/nguha/legalbench)
- Extension: **LegalBench-RAG** for retrieval quality — Precision@k, Recall@k metrics

### 5.2 Legal Hallucination Data

Fabricated case citations are the primary legal hallucination failure mode:

| Study | Finding |
|-------|---------|
| Stanford study | LLMs hallucinated in 75%+ of cases about court rulings; 120+ fabricated cases |
| GPT-4/GPT-3.5/PaLM 2/Llama 2 study | 58-88% baseline hallucination on US federal case law |
| LexisNexis/Thomson Reuters (preregistered) | 17-33% hallucination in production |

Detection: Span-level verification (REFIND, SemEval 2025), LLM-as-Judge with citation verification rubric.

---

## 6. RAG-Specific Evaluators

### 6.1 RAGAS Framework (30+ Metrics)

RAGAS expanded from 4 original metrics to a comprehensive suite:

**Retrieval metrics**: `ContextPrecision`, `ContextRecall`, `ContextEntitiesRecall`, `NoiseSensitivity`

**Response metrics**: `Faithfulness` (0-1), `ResponseRelevancy`, `ResponseGroundedness`, `AnswerAccuracy` (requires reference)

**Agent metrics** (added 2024-2025): `ToolCallAccuracy` (exact order+content), `ToolCallF1` (unordered soft), `AgentGoalAccuracy` (binary end-state), `TopicAdherence`

**Multimodal** (2025): `MultimodalFaithfulness`, `MultimodalRelevance`

**Custom**: `AspectCritic`, `SimpleCriteriaScoring`, `RubricsBasedScoring`

Source: [docs.ragas.io/en/stable/concepts/metrics/available_metrics](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/)

### 6.2 Arize Phoenix

- Built-in evaluators: hallucination (Q&A and RAG), summarization, toxicity, agent tool selection
- Custom LLM judge: rubric-based, any provider
- RAG metrics: Hit@k, Precision@k, nDCG, `RelevanceEvaluator`
- Source: [arize.com/docs/phoenix](https://arize.com/docs/phoenix/cookbook/evaluation/evaluate-rag)

### 6.3 LlamaIndex Evaluation

Core evaluators: `FaithfulnessEvaluator`, `RelevancyEvaluator`, `CorrectnessEvaluator`, `RetrieverEvaluator`. All use a "gold" LLM (default GPT-4) as judge. RAGAS metrics can layer on top.

Source: [developers.llamaindex.ai/python/framework/module_guides/evaluating](https://developers.llamaindex.ai/python/framework/module_guides/evaluating/)

---

## 7. Agent/Tool-Use Evaluators

### 7.1 Berkeley Function-Calling Leaderboard (BFCL)

- **Version**: BFCL V4 (agentic evaluation), ICML 2025
- **Method**: AST evaluation for scalable function call correctness; stateful multi-turn agentic evaluation
- **Coverage**: Serial calls, parallel calls, multi-turn memory, abstention, dynamic decision-making

Top scores (2025-2026):

| Model | Score |
|-------|-------|
| Claude Opus 4.1 | 70.36% |
| Claude Sonnet 4 | 70.29% |
| GPT-5 | 59.22% |

**Finding**: Frontier models ace single-turn calls but still fail on long-horizon reasoning, memory, and abstention.

Source: [gorilla.cs.berkeley.edu/leaderboard.html](https://gorilla.cs.berkeley.edu/leaderboard.html)

### 7.2 Trajectory Evaluation

- BFCL V3/V4 multi-step evaluation covers correct sequence of tool calls
- RAGAS `ToolCallAccuracy` requires exact order+content; `ToolCallF1` is order-agnostic
- DeepEval's Plan Adherence metric evaluates trajectory against plan
- No single standard benchmark for multi-step reasoning evaluation

---

## 8. Evaluation Composition Patterns

### 8.1 Panel / Ensemble Evaluation

| System | Approach | Result |
|--------|----------|--------|
| **PoLL** | 3 smaller models (command-r, gpt-3.5-turbo, haiku); max voting or average | Reduces cost vs. single GPT-4 judge |
| **SE-Jury** (ASE 2025) | 5 distinct strategies; dynamic team selection | Near-human correlation for SE tasks |
| **RADAR** | Role-specialized panel (Security Auditor, Vulnerability Detector, Critic, Arbiter) | Safety evaluation focus |
| **Agent-as-a-Judge** ([arXiv:2508.02994](https://arxiv.org/html/2508.02994v1)) | LLM agents with distinct personas in structured debate | More robust for complex multi-step tasks |

### 8.2 Meta-Evaluation

**MT-Bench / Chatbot Arena** (Zheng et al., NeurIPS 2023): GPT-4 as judge achieves >80% agreement with humans. Identified biases: position, verbosity, self-enhancement. Mitigations: positional randomization, few-shot prompting.

**Multi-Agent Meta-Judge** ([arXiv:2504.17087](https://arxiv.org/html/2504.17087v1)): N LLMs evaluate judgments of other LLMs; aggregated scores approach ground truth.

### 8.3 G-Eval Composition

G-Eval supports injectable `evaluation_template` for domain customization. Stacking pattern:
1. G-Eval for general quality
2. Domain-specific DAGMetric for structured criteria
3. ArenaGEval for pairwise comparison
4. Results normalized to 0-1, aggregated by weighted average or minimum-pass threshold

### 8.4 CI/CD Pipeline Integration

Standard pattern across Langfuse, DeepEval, Braintrust:
1. Golden dataset stored in evaluation platform
2. Code change triggers automated evaluation run
3. LLM-as-Judge + deterministic checks score outputs
4. Pass/fail thresholds gate deployment; regression alerts on score drops
5. Results pushed to observability platform for trend tracking

Source: [braintrust.dev/articles/best-ai-evals-tools-cicd-2025](https://www.braintrust.dev/articles/best-ai-evals-tools-cicd-2025)

---

## 9. Observability Platform Support

### 9.1 Platform Comparison

| Platform | LLM-as-Judge | Custom Evaluators | Span-Level Metrics | OTel Native |
|----------|-------------|-------------------|-------------------|-------------|
| **Langfuse** | OpenAI, Anthropic, Azure, any OpenAI-compatible | Python/TS SDK | Scores on traces, observations, sessions | Yes |
| **Arize Phoenix** | Built-in templates + custom rubric | Any provider | Per-span evaluator results | Yes (OpenInference) |
| **Datadog** | Built-in (hallucination, failed response, PII) | Custom KPIs | Latency, tokens, errors per step | Yes (v1.37+) |
| **Opik** (Comet) | Built-in (Hallucination, Moderation, AnswerRelevance) | Subclass `BaseMetric` | `task_span` parameter for span-level | Yes |

### 9.2 Opik Span-Level Metrics

Opik's distinguishing feature is `task_span` metrics — span-level analysis by subclassing `BaseMetric` with a `task_span` parameter. System auto-enables span collection. Python code metrics (2025) evaluate production traces in real-time.

Source: [comet.com/docs/opik/evaluation/metrics/task_span_metrics](https://www.comet.com/docs/opik/evaluation/metrics/task_span_metrics)

---

## 10. Toolkit Relevance

| Toolkit Component | Relationship |
|-------------------|-------------|
| `GEvalConfig` at `src/lib/llm-as-judge.ts:327` | Accepts custom criteria and evaluation steps — domain evaluators implemented as specialized configs |
| `TestCase` at `src/lib/llm-as-judge.ts:308` | Extensible with domain-specific fields |
| `panelEvaluation()` in `src/lib/llm-as-judge.ts` | Multi-evaluator pattern — domain evaluators compose via panels |
| `buildEvalPrompt()` at `src/lib/llm-as-judge.ts:803` | G-Eval prompt construction with injectable templates |
| `EvaluationResult.explanation` | Maps to Phoenix `provide_explanation` and domain-specific reasoning |

Domain evaluators would be implemented as specialized `GEvalConfig` instances with domain criteria, not new code paths. The DAGMetric pattern from DeepEval could inform a future `DAGEvalConfig` for the toolkit.

---

## 11. Open Questions

- Whether DeepEval's OTLP export for evaluation runs uses standard OTel semconv or custom attributes
- MedHELM adoption in production medical AI evaluation pipelines
- LegalBench-RAG adoption by LexisNexis or Thomson Reuters for internal evaluation
- CodeJudgeBench calibration against human expert code reviewers — not yet widely replicated
- No platform ships formalized OTel semconv for code eval metrics (syntax correctness, test pass rate)
