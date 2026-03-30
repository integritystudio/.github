# Agent-as-Judge: Architecture and Design

**Version**: 3.0.0
**Status**: Production
**Last Updated**: 2026-03-26

---

## Abstract

Agent-as-Judge is an evaluation framework for agentic AI systems. It addresses a fundamental gap in standard LLM evaluation: single-pass scoring cannot assess multi-step reasoning, tool use correctness, or cross-agent collaboration. This document describes the current production architecture, the design decisions behind it, and how the system relates to the complementary DAG-based evaluation framework.

---

## 1. Motivation

Standard LLM evaluation — scoring a model's output against a reference or rubric in a single forward pass — works well for bounded tasks. It fails for agentic systems because:

- **Trajectory matters**: an agent may produce a correct final answer via an inefficient or incorrect sequence of steps. Scoring only the output misses the quality of reasoning.
- **Tool use is verifiable**: whether the agent called the right tool with the right arguments is a factual question, not a fluency judgment.
- **Multi-agent interactions compound errors**: incorrect handoffs and delegation failures are invisible to output-only evaluation.
- **Conversation sessions span many turns**: single-turn metrics do not accumulate across context windows.

Agent-as-Judge addresses these by running an autonomous judge agent that has access to the full execution trace: actions, tool calls, intermediate results, and final output.

---

## 2. Architecture

The system is organized into two complementary frameworks:

```
┌─────────────────────────────────────────────────────────────────┐
│                   Quality Evaluation System                      │
├───────────────────────────┬─────────────────────────────────────┤
│   Agent-as-Judge          │   DAG Evaluation                    │
│   src/lib/agent-judge/    │   src/lib/judge/llm-judge-dag.ts    │
│                           │                                     │
│   Evaluates agentic        │   Structured multi-hop evaluation   │
│   trajectories, tool use,  │   via directed acyclic graph.       │
│   and multi-agent flows.   │   Deterministic traversal paths     │
│   Class-based patterns.    │   with LLM nodes at branch points.  │
└───────────────────────────┴─────────────────────────────────────┘
```

Both frameworks produce `EvalResult` / `EvaluationResult` objects compatible with OTel span attributes and the MCP `obs_query_evaluations` tool.

### 2.1 Agent-as-Judge Module (`src/lib/agent-judge/`)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent-as-Judge Components                     │
├─────────────────────────────────────────────────────────────────┤
│  Judge Classes                  Tool Verification                │
│  ├── AgentJudge (abstract)     ├── verifyToolCall()             │
│  ├── ProceduralJudge           ├── verifyToolCalls()            │
│  └── ReactiveJudge             └── Weighted scoring             │
├─────────────────────────────────────────────────────────────────┤
│  Step Scoring                   Trajectory Analysis              │
│  ├── scoreStep()               ├── analyzeTrajectory()          │
│  ├── aggregateStepScores()     ├── Redundancy detection         │
│  └── Weighted aggregation      └── Loopiness metrics            │
├─────────────────────────────────────────────────────────────────┤
│  Multi-Agent Collaboration      Production Utilities             │
│  ├── collectiveConsensus()     ├── AgentEvalTimeoutError        │
│  ├── Convergence detection     ├── withAgentTimeout()           │
│  └── Variance tracking         └── LRU memory (bounded 1000)   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 DAG Evaluation Framework (`src/lib/judge/llm-judge-dag.ts`)

Structured evaluation via a `Map<string, DAGNode>` graph. The evaluator traverses from a root node, calling the LLM only at judgement and task nodes, until it reaches a `VerdictNode`. This separates evaluation logic (the graph structure) from execution (LLM calls), making evaluations composable, testable, and auditable.

---

## 3. Core Types

### 3.1 Agent Evaluation Types

```typescript
// The subject being evaluated
interface Evaluand {
  input: string;              // Original user task
  output: string;             // Final agent output
  actions?: AgentAction[];    // Execution trace
  context?: Record<string, unknown>;
  expectedOutput?: string;
  agentId?: string;
  agentName?: string;
}

// A single action in the agent's trace
interface AgentAction {
  type: string;               // 'tool_call' | 'reasoning' | 'response'
  tool?: string;
  toolCallId?: string;
  arguments?: Record<string, unknown>;
  result?: unknown;
  reasoning?: string;
  timestamp?: string;
}

// Evaluation output
interface AgentEvalResult {
  overallScore: number;              // Normalized 0–1
  stepScores: StepScore[];           // Per-step breakdown
  toolVerifications: ToolVerification[];
  trajectoryLength: number;
  explanation: string;
  actionableFeedback: string[];
  rawResponse?: string;
}
```

### 3.2 DAG Node Types

```typescript
type VerdictNode = {
  type: 'verdict';
  score: number;       // 0–10 (normalized to 0–1 on output)
  label: string;
};

type TaskNode = {
  type: 'task';
  instruction: string; // Prompt to transform current input
  next: string;        // Next node ID
};

type BinaryJudgementNode = {
  type: 'binary_judgement';
  criteria: string;
  trueChild: string;
  falseChild: string;
};

type NonBinaryJudgementNode = {
  type: 'non_binary_judgement';
  criteria: string;
  verdicts: Map<string, string>; // label → next node ID
};

// Routes by numeric score from LLM (0–10).
// Branches to aboveChild if score >= scoreThreshold.
type ConditionalJudgementNode = {
  type: 'conditional_judgement';
  criteria: string;
  scoreThreshold: number;   // [0, 10]
  aboveChild: string;
  belowChild: string;
};

type DAGNode =
  | VerdictNode
  | TaskNode
  | BinaryJudgementNode
  | NonBinaryJudgementNode
  | ConditionalJudgementNode;
```

### 3.3 DAG Configuration

```typescript
interface DAGEvalConfig {
  name: string;
  nodes: Map<string, DAGNode>;
  rootId: string;
  temperature?: number;
  maxDepth?: number;        // Guard against runaway traversal
  edgeWeights?: Map<string, number>; // "fromId:toId" → multiplier ∈ (0, 1]
}
```

`validateDAGConfig` runs at evaluation start: it verifies the root exists, all child references resolve, `scoreThreshold` is in range, and no cycles are present (DFS).

---

## 4. Evaluation Patterns

### 4.1 Procedural Judge

Fixed pipeline — ordered stages run in sequence, each receiving the evaluand and an accumulated context map. Useful for domain-specific evaluations with known, ordered criteria.

```
Evaluand → Stage 1 → Stage 2 → ... → Stage N → aggregateStepScores()
               ↓
          [early termination if critical stage score < 0.3]
```

- Stages share context via a `Record<string, unknown>` passed forward.
- `earlyTerminationOn` configures which stage triggers early exit on failure.
- Returns `overallScore = 0` on early termination.

### 4.2 Reactive Judge

Adaptive routing — a `router` function selects relevant specialist evaluators based on the evaluand content. Optional `deepDiveSpecialists` are invoked when a specialist flags `needsDeepDive: true`.

```
Evaluand → router() → [Specialist A, Specialist C] → aggregateStepScores()
                                ↓
                     [deepDive if needsDeepDive]
```

- Intermediate specialist results stored in bounded LRU memory.
- Max 10 concurrent evaluators (`MAX_CONCURRENT_EVALUATORS`).

### 4.3 Consensus Judge

Multi-agent debate via `collectiveConsensus()`. Multiple judge agents score independently; results are aggregated per round until variance falls below a threshold or `MAX_CONSENSUS_ROUNDS` is reached.

```
Judges → Round 1 → (variance > threshold?) → Round 2 → ... → median()
```

- `DEFAULT_CONVERGENCE_THRESHOLD = 0.1`
- `MAX_CONSENSUS_ROUNDS = 5`
- `calculateVariance()` and `calculateMedian()` are exported for custom pipelines.

### 4.4 DAG Evaluation

Structured traversal replaces the ad-hoc judge pattern for evaluations where the decision tree can be specified declaratively. The LLM is called only at judgement/task nodes; the rest is deterministic graph traversal.

**Design advantages:**
- Evaluation logic is a pure data structure — serializable, diffable, and version-controlled.
- LLM calls are isolated to well-defined prompt/response boundaries.
- `edgeWeights` allow path-specific score penalties without changing node verdicts.
- `maxDepth` prevents unbounded traversal in malformed or adversarial inputs.

**Score computation:**
```
finalScore = (verdictScore / 10) × ∏(edgeWeights along traversed path)
```

Edge weights default to 1.0; values must be in `(0, 1]`. A traversal through a penalized path (e.g., a fallback branch) produces a lower final score even if the terminal verdict is high.

---

## 5. Metric Categories

### 5.1 Single-Turn

| Metric | Description | Source |
|--------|-------------|--------|
| Task completion | End-to-end goal achievement | `agent-eval-metrics.ts` |
| Tool correctness | Correct tool selected | `verifyToolCall()` |
| Argument correctness | Valid arguments passed | `verifyToolCalls()` |

### 5.2 Multi-Turn

| Metric | Description |
|--------|-------------|
| Conversation completeness | User satisfaction across session |
| Turn relevancy | On-topic responses per turn |
| Context retention | Correct use of prior turns |

### 5.3 Multi-Agent

| Metric | Description |
|--------|-------------|
| Handoff correctness | Appropriate delegation decisions |
| Collaboration efficiency | Steps vs optimal delegation path |
| Role adherence | Agent stayed within assigned scope |

---

## 6. Safety and Resource Bounds

| Guard | Mechanism | Constant |
|-------|-----------|----------|
| Evaluation timeout | `withAgentTimeout()` wraps all judge `evaluate()` calls | `DEFAULT_AGENT_EVAL_TIMEOUT_MS = 60_000` |
| Judge memory | LRU eviction in `AgentJudge.storeInMemory()` | `MAX_JUDGE_MEMORY_SIZE = 1000` |
| DAG depth | `maxDepth` check per traversal step | Configurable per `DAGEvalConfig` |
| Trajectory length | `analyzeTrajectory()` caps input | `MAX_TRAJECTORY_LENGTH = 1000` |
| Consensus rounds | Loop exit after N rounds | `MAX_CONSENSUS_ROUNDS = 5` |
| Concurrent evaluators | Specialist concurrency cap | `MAX_CONCURRENT_EVALUATORS = 10` |

---

## 7. OTel and MCP Integration

All judge outputs are compatible with the OTel semantic conventions for `gen_ai.evaluation.*` spans.

### MCP Query Parameters (`obs_query_evaluations`)

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | string | Subject agent ID (max 128 chars) |
| `agentName` | string | Subject agent name (max 256 chars) |
| `evaluatorType` | `llm \| human \| rule \| classifier` | Filter by evaluator kind |
| `evaluationName` | string | Metric name filter |
| `aggregation` | `avg \| p50 \| p95 \| p99` | Aggregation function |
| `groupBy` | string[] | Grouping dimensions |

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `stepScores` | `StepScore[]` | Per-step evaluation scores |
| `toolVerifications` | `ToolVerification[]` | Tool call correctness results |
| `trajectoryLength` | number | Total steps in agent trace |

### Export Platforms

| Platform | Agent-Specific Feature |
|----------|----------------------|
| Langfuse | Step scores as span annotations |
| Confident AI | Agent trajectory in test case metadata |
| Arize Phoenix | Tool verifications as evaluation feedback |
| Datadog | Agent metrics with ML app segmentation |

---

## 8. Design Decisions

**Why class-based judge patterns?**
`ProceduralJudge` and `ReactiveJudge` require persistent state across a multi-step evaluation (accumulated context, specialist results). A class with bounded LRU memory provides a natural container. The `AgentJudge` abstract base enforces the interface while keeping memory management consistent.

**Why separate DAG evaluation from agent-judge patterns?**
The agent-judge patterns (`ProceduralJudge`, `ReactiveJudge`) are imperative: the evaluation logic lives in functions you write and pass in. DAG evaluation is declarative: you describe the evaluation as a graph, and `dagEval` executes it. The two are complementary. DAG is suited for evaluations that can be fully specified in advance; agent-judge patterns are suited for evaluations where the judge needs to reason adaptively.

**Why `ConditionalJudgementNode` in addition to `BinaryJudgementNode`?**
Binary judgement forces the LLM into a true/false answer, which is unreliable for graded criteria. `ConditionalJudgementNode` asks for a numeric score (0–10) and routes based on a configurable threshold. This allows partial-credit semantics in a deterministic branching structure.

**Why edge weights instead of node-level score adjustments?**
Edge weights encode path quality independently of verdict quality. A correct but circuitous path (multiple fallback branches) should score lower than a direct path, even if both reach the same terminal verdict. Encoding this at the edge level keeps node verdicts semantically clean.

---

## Related Documentation

- [LLM-as-Judge](llm-as-judge.md) — single-pass G-Eval and QAG evaluation baseline
- [Quality Evaluation](quality-evaluation.md) — evaluation storage, export, and dashboard integration
- [Interface Research Index](../interface/README.md) — explainability roadmap
