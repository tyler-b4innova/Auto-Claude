# Parallel Verification Enhancement Plan for Auto-Claude

## Executive Summary

This document proposes a comprehensive architectural enhancement to Auto-Claude, incorporating parallel verification and multi-hypothesis exploration techniques inspired by DeepMind's AlphaProof and Gemini Deep Think systems. The goal is to improve task success rates, reduce human intervention, and produce higher-quality solutions for complex software engineering tasks.

**Key Insight**: DeepMind's recent breakthroughs demonstrate that exploring multiple solution paths simultaneously, with continuous verification and cross-path communication, produces dramatically better results than sequential chain-of-thought reasoning—especially for difficult problems.

---

## Table of Contents

1. [Current Architecture Analysis](#1-current-architecture-analysis)
2. [DeepMind Research Findings](#2-deepmind-research-findings)
3. [Proposed Enhancements](#3-proposed-enhancements)
4. [Detailed Design](#4-detailed-design)
5. [Implementation Phases](#5-implementation-phases)
6. [Cost-Benefit Analysis](#6-cost-benefit-analysis)
7. [Risk Mitigation](#7-risk-mitigation)
8. [Success Metrics](#8-success-metrics)
9. [Open Questions](#9-open-questions)

---

## 1. Current Architecture Analysis

### 1.1 Decomposition Strategy

Auto-Claude uses a **three-level hierarchical decomposition**:

```
Level 1: Spec Phases (Sequential Chain-of-Thought)
├── Discovery → Requirements → [Research] → Context → Spec → Plan → Validate
│   Each phase builds on previous phase's output
│
Level 2: Implementation Phases (Dependency-Based)
├── Phase 1: Backend (no dependencies)
├── Phase 2: Worker (depends_on: Phase 1)
├── Phase 3: Frontend (depends_on: Phase 1)  ← Can parallel with Phase 2
└── Phase 4: Integration (depends_on: Phase 2, 3)
│
Level 3: Subtasks (Sequential within Phase)
└── Sub-1-1 → Sub-1-2 → Sub-1-3 (within each phase)
```

### 1.2 Current Verification Mechanisms

| Layer | Mechanism | Timing | Cost |
|-------|-----------|--------|------|
| Subtask | Command/API/Browser verification | Post-implementation | Medium |
| Session | Post-session processing (Python) | After each subtask | Low |
| Recovery | Failure classification + hints | On failure | Low |
| QA | Full acceptance criteria validation | After all subtasks | High |

**Key Limitation**: Verification is **post-hoc**—invalid approaches consume full implementation tokens before discovery.

### 1.3 Current Parallel Capabilities

- **Phase-level parallelism**: Phases with same dependencies can run concurrently
- **Subagent spawning**: Coder agent can spawn up to 10 subagents via Task tool
- **Agent decision**: Parallelism is agent-initiated, not automatic
- **No cross-agent communication**: Subagents are independent

### 1.4 Current Failure Handling

```python
# Simplified flow from services/recovery.py
for attempt in range(1, 4):
    result = run_subtask()
    if result.success:
        return SUCCESS

    failure_type = classify_failure(result.error)
    if is_circular_fix(subtask_id, attempt.approach):
        return STUCK  # Escalate to human

    recovery_hints = get_recovery_hints(subtask_id)
    # Retry with hints injected into prompt

return STUCK  # After 3 failures
```

**Limitation**: Sequential retries often repeat similar mistakes. No parallel exploration of alternative approaches.

---

## 2. DeepMind Research Findings

### 2.1 AlphaProof (Nature, 2025)

**Architecture**: AlphaZero-inspired agent for formal mathematical proofs

**Key Innovations**:

1. **Embedded Verification**: Uses Lean proof assistant to verify each step immediately
   - Invalid proof steps are pruned instantly (near-zero cost)
   - Valid steps reinforce the model
   - No "hallucination" possible—proofs are formally verified

2. **Test-Time RL (TTRL)**: For hard problems, generates millions of simplified variants
   - Solves easier versions first
   - Transfers learning to harder versions
   - Scales compute proportional to difficulty

3. **Monte Carlo Tree Search**: Intelligent decomposition into sub-goals
   - Not brute-force enumeration
   - Learns which branches are promising
   - Prunes unpromising paths early

**Results**: Solved IMO 2024's hardest problem (only 5/609 humans solved it)

### 2.2 Gemini Deep Think (2025)

**Architecture**: Multi-agent parallel reasoning system

**Key Innovations**:

1. **Parallel Hypothesis Exploration**:
   ```
   Problem → [Mathematical Analysis]
           → [Pattern Recognition]    → Synthesis → Best Answer
           → [Heuristic Approaches]
           → [First Principles]
   ```

2. **Inter-Path Communication**: Different reasoning paths share insights
   - Discovery in Path A informs Path B
   - Reduces redundant exploration
   - Creates "interconnected reasoning"

3. **Best-of-Many Selection**: Combines strongest elements from each path
   - Not just picking winner
   - Synthesis of best ideas

4. **Adaptive Compute**: "Takes hours to reason" on hard problems
   - Easy problems: seconds
   - Hard problems: hours of parallel exploration

**Results**: Gold medal at IMO 2025, industry-leading on Humanity's Last Exam

### 2.3 Key Principles Extracted

| Principle | AlphaProof | Deep Think | Current Auto-Claude |
|-----------|------------|------------|---------------------|
| Verification timing | During search | During reasoning | Post-implementation |
| Parallelism | Millions of branches | Multiple agents | Limited (agent-decided) |
| Communication | Via proof tree | Between paths | None between subagents |
| Compute scaling | Proportional to difficulty | Adaptive | Fixed per subtask |
| Failure handling | Prune immediately | Discard weak hypotheses | Retry with hints |

---

## 3. Proposed Enhancements

### 3.1 Enhancement Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PARALLEL VERIFICATION ARCHITECTURE               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐    ┌─────────────────────────────────────────┐    │
│  │  Difficulty │    │         PARALLEL EXPLORER               │    │
│  │  Detector   │───►│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │    │
│  └─────────────┘    │  │ Agent A │ │ Agent B │ │ Agent C │   │    │
│         │           │  │ (Approach│ │(Approach│ │(Approach│   │    │
│         │           │  │    1)   │ │   2)    │ │   3)    │   │    │
│         ▼           │  └────┬────┘ └────┬────┘ └────┬────┘   │    │
│  ┌─────────────┐    │       │           │           │        │    │
│  │   Router    │    │       ▼           ▼           ▼        │    │
│  │  (Easy →    │    │  ┌─────────────────────────────────┐   │    │
│  │  Sequential │    │  │      INSIGHT BROADCASTER        │   │    │
│  │   Hard →    │    │  │  (Patterns, Gotchas, Discoveries)│   │    │
│  │  Parallel)  │    │  └─────────────────────────────────┘   │    │
│  └─────────────┘    │                   │                    │    │
│                     │                   ▼                    │    │
│                     │  ┌─────────────────────────────────┐   │    │
│                     │  │      CONTINUOUS VERIFIER        │   │    │
│                     │  │  (Type check, lint, unit tests) │   │    │
│                     │  └─────────────────────────────────┘   │    │
│                     │                   │                    │    │
│                     │                   ▼                    │    │
│                     │  ┌─────────────────────────────────┐   │    │
│                     │  │        SYNTHESIZER              │   │    │
│                     │  │  (Combine best elements)        │   │    │
│                     │  └─────────────────────────────────┘   │    │
│                     └─────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Core Components

#### Component 1: Difficulty-Aware Router

**Purpose**: Route subtasks to sequential or parallel execution based on predicted difficulty

**Signals Used**:
- Existing complexity signals from `spec/complexity.py`
- Historical failure rates for similar tasks
- File count, service count, integration complexity
- Recovery manager attempt history

**Routing Logic**:
```python
def route_subtask(subtask: Subtask, context: SubtaskContext) -> ExecutionStrategy:
    difficulty = estimate_difficulty(subtask, context)

    if difficulty < 0.3:
        return SequentialStrategy(max_retries=2)
    elif difficulty < 0.6:
        return SequentialStrategy(max_retries=3, fast_verification=True)
    elif difficulty < 0.8:
        return ParallelStrategy(branches=2, with_synthesis=False)
    else:
        return ParallelStrategy(branches=3, with_synthesis=True, extended_compute=True)
```

#### Component 2: Parallel Explorer

**Purpose**: Spawn multiple agents exploring different approaches simultaneously

**Approach Diversification Strategies**:
1. **Architecture variation**: "Use composition" vs "Use inheritance"
2. **Library variation**: "Use library X" vs "Use library Y" vs "Write from scratch"
3. **Pattern variation**: "Follow existing pattern A" vs "Follow pattern B"
4. **Granularity variation**: "Minimal changes" vs "Comprehensive refactor"

**Implementation**:
```python
async def parallel_explore(subtask: Subtask, branches: int) -> list[BranchResult]:
    approaches = generate_diverse_approaches(subtask, count=branches)

    async with asyncio.TaskGroup() as tg:
        tasks = [
            tg.create_task(run_branch(subtask, approach, branch_id=i))
            for i, approach in enumerate(approaches)
        ]

    return [task.result() for task in tasks]
```

#### Component 3: Insight Broadcaster

**Purpose**: Share discoveries between parallel branches in real-time

**Broadcast Events**:
- Pattern discovered: "This codebase uses factory pattern for X"
- Gotcha discovered: "API returns 404 for empty results, not empty array"
- File mapping: "Configuration lives in config/settings.py, not .env"
- Verification result: "Tests require DATABASE_URL to be set"

**Implementation**:
```python
class InsightBroadcaster:
    def __init__(self):
        self.insights = []
        self.subscribers = []

    async def broadcast(self, insight: Insight):
        self.insights.append(insight)
        for subscriber in self.subscribers:
            await subscriber.receive_insight(insight)

    def get_context_for_branch(self, branch_id: int) -> list[Insight]:
        # Return insights from OTHER branches (not self)
        return [i for i in self.insights if i.source_branch != branch_id]
```

#### Component 4: Continuous Verifier

**Purpose**: Prune invalid approaches early, before full implementation completes

**Verification Checkpoints**:
```
┌──────────────────────────────────────────────────────────────┐
│                    VERIFICATION TIMELINE                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Import added    Function written    File saved    Subtask   │
│       │                │                 │         complete  │
│       ▼                ▼                 ▼            ▼      │
│  ┌─────────┐    ┌───────────┐    ┌──────────┐   ┌────────┐  │
│  │Type     │    │ Lint      │    │ Unit     │   │ Full   │  │
│  │Check    │    │ Check     │    │ Tests    │   │ Verify │  │
│  └────┬────┘    └─────┬─────┘    └────┬─────┘   └────────┘  │
│       │               │               │                      │
│       ▼               ▼               ▼                      │
│   Invalid?        Warnings?       Failures?                  │
│   PRUNE NOW       Broadcast       Prune or                   │
│                   to others       continue                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Pruning Decisions**:
```python
class ContinuousVerifier:
    async def check_and_decide(self, branch: Branch, checkpoint: str) -> Decision:
        result = await self.run_verification(branch, checkpoint)

        if result.is_fatal_error():
            # Syntax error, import error, type error
            return Decision.PRUNE_IMMEDIATELY

        if result.is_warning():
            # Lint warning, deprecation notice
            await self.broadcaster.broadcast(Insight(
                type="warning",
                content=result.message,
                source_branch=branch.id
            ))
            return Decision.CONTINUE_WITH_CAUTION

        if result.is_test_failure():
            # Unit test failed
            if branch.can_recover(result):
                return Decision.CONTINUE_AND_FIX
            else:
                return Decision.PRUNE_IF_OTHERS_HEALTHIER

        return Decision.CONTINUE
```

#### Component 5: Solution Synthesizer

**Purpose**: Combine best elements from multiple branches into optimal solution

**Synthesis Strategies**:

1. **Winner-Take-All** (simple):
   - Select branch with most tests passing
   - Use when solutions are incompatible

2. **Cherry-Pick** (medium):
   - Take error handling from Branch A
   - Take algorithm from Branch B
   - Take tests from Branch C

3. **Deep Synthesis** (complex):
   - AI analyzes all branches
   - Identifies strengths/weaknesses of each
   - Generates new solution combining best ideas

**Implementation**:
```python
class SolutionSynthesizer:
    async def synthesize(self, branches: list[BranchResult]) -> FinalSolution:
        # Filter to successful/partially successful branches
        viable = [b for b in branches if b.verification_score > 0.5]

        if len(viable) == 0:
            return await self.recovery_synthesis(branches)

        if len(viable) == 1:
            return FinalSolution(source=viable[0], method="winner")

        # Multiple viable solutions - analyze for synthesis
        analysis = await self.analyze_branches(viable)

        if analysis.solutions_compatible:
            return await self.cherry_pick_synthesis(viable, analysis)
        else:
            return await self.select_best(viable, analysis)
```

---

## 4. Detailed Design

### 4.1 Difficulty Estimation Model

**Input Features**:
```python
@dataclass
class DifficultyFeatures:
    # From subtask definition
    files_to_modify: int
    files_to_create: int
    patterns_from_count: int
    verification_type: VerificationType

    # From context
    service_type: str  # backend, frontend, worker
    has_external_integration: bool
    requires_migration: bool
    touches_security: bool

    # From history
    similar_task_failure_rate: float
    avg_attempts_for_similar: float
    current_attempt_count: int

    # From codebase
    file_complexity_score: float  # cyclomatic complexity
    test_coverage_of_touched_files: float
```

**Model Architecture**:
```python
def estimate_difficulty(features: DifficultyFeatures) -> float:
    # Weighted scoring based on empirical observation
    score = 0.0

    # File complexity (0-0.3)
    score += min(0.3, features.files_to_modify * 0.05 + features.files_to_create * 0.1)

    # Integration complexity (0-0.2)
    if features.has_external_integration:
        score += 0.15
    if features.requires_migration:
        score += 0.1
    if features.touches_security:
        score += 0.1

    # Historical difficulty (0-0.3)
    score += features.similar_task_failure_rate * 0.2
    score += min(0.1, features.current_attempt_count * 0.05)

    # Codebase factors (0-0.2)
    score += (1 - features.test_coverage_of_touched_files) * 0.1
    score += min(0.1, features.file_complexity_score / 50)

    return min(1.0, score)
```

### 4.2 Approach Diversification

**Strategy Generation**:
```python
class ApproachGenerator:
    def generate_diverse_approaches(
        self,
        subtask: Subtask,
        context: SubtaskContext,
        count: int
    ) -> list[Approach]:
        approaches = []

        # Strategy 1: Vary architectural pattern
        if subtask.involves_new_abstraction:
            approaches.extend([
                Approach(strategy="composition", description="Use composition over inheritance"),
                Approach(strategy="inheritance", description="Extend existing base class"),
                Approach(strategy="functional", description="Use pure functions, no classes"),
            ])

        # Strategy 2: Vary library choice
        if subtask.has_library_options:
            for lib in context.available_libraries:
                approaches.append(Approach(
                    strategy=f"use_{lib.name}",
                    description=f"Implement using {lib.name} library"
                ))

        # Strategy 3: Vary scope
        approaches.extend([
            Approach(strategy="minimal", description="Minimal changes to existing code"),
            Approach(strategy="comprehensive", description="Comprehensive refactor for clarity"),
        ])

        # Strategy 4: Vary error handling
        if subtask.involves_external_calls:
            approaches.extend([
                Approach(strategy="defensive", description="Extensive error handling and retries"),
                Approach(strategy="fail_fast", description="Fail fast, let caller handle errors"),
            ])

        # Select diverse subset
        return self.select_diverse_subset(approaches, count)
```

### 4.3 Branch Communication Protocol

**Message Types**:
```python
class InsightType(Enum):
    PATTERN_DISCOVERED = "pattern"      # Code pattern found
    GOTCHA_DISCOVERED = "gotcha"        # Pitfall to avoid
    FILE_MAPPING = "file_mapping"       # Where things live
    VERIFICATION_RESULT = "verification" # Test/lint result
    APPROACH_FAILED = "approach_failed"  # Approach doesn't work
    APPROACH_PROMISING = "promising"     # Early positive signal
```

**Communication Flow**:
```
Branch A                    Broadcaster                    Branch B
    │                            │                            │
    ├─── discover pattern ──────►│                            │
    │                            ├────── broadcast ──────────►│
    │                            │                            │
    │                            │◄────── acknowledge ────────┤
    │                            │                            │
    │                            │    (B adjusts approach     │
    │                            │     based on A's finding)  │
    │                            │                            │
    │◄────── receive gotcha ─────┤◄────── discover gotcha ────┤
    │                            │                            │
    │    (A avoids the           │                            │
    │     same mistake)          │                            │
```

### 4.4 Verification Checkpoint Integration

**Checkpoint Definitions**:
```python
VERIFICATION_CHECKPOINTS = {
    "python": [
        Checkpoint(
            trigger="file_saved",
            command="python -m py_compile {file}",
            fatal_on_failure=True,
            description="Syntax check"
        ),
        Checkpoint(
            trigger="file_saved",
            command="mypy {file} --ignore-missing-imports",
            fatal_on_failure=False,
            broadcast_warnings=True,
            description="Type check"
        ),
        Checkpoint(
            trigger="function_complete",
            command="pytest {test_file} -x -q",
            fatal_on_failure=False,
            description="Unit tests"
        ),
    ],
    "typescript": [
        Checkpoint(
            trigger="file_saved",
            command="tsc --noEmit {file}",
            fatal_on_failure=True,
            description="Type check"
        ),
        Checkpoint(
            trigger="file_saved",
            command="eslint {file}",
            fatal_on_failure=False,
            broadcast_warnings=True,
            description="Lint check"
        ),
    ],
}
```

**Integration with Agent**:
```python
class VerifiedCoderAgent:
    async def on_file_written(self, file_path: str):
        for checkpoint in self.get_checkpoints("file_saved"):
            result = await self.run_checkpoint(checkpoint, file_path)

            if result.failed and checkpoint.fatal_on_failure:
                raise BranchPrunedException(f"Fatal: {result.error}")

            if result.warnings and checkpoint.broadcast_warnings:
                await self.broadcaster.broadcast(Insight(
                    type=InsightType.VERIFICATION_RESULT,
                    content=result.warnings,
                    source_branch=self.branch_id
                ))
```

### 4.5 Synthesis Algorithm

**Phase 1: Branch Analysis**
```python
async def analyze_branches(self, branches: list[BranchResult]) -> BranchAnalysis:
    analysis_prompt = f"""
    Analyze these {len(branches)} implementation branches for the same subtask.

    For each branch, identify:
    1. Key architectural decisions made
    2. Strengths (what it does well)
    3. Weaknesses (problems or limitations)
    4. Unique insights or patterns discovered
    5. Compatibility with other branches

    Branches:
    {self.format_branches_for_analysis(branches)}

    Return structured analysis.
    """

    return await self.llm.analyze(analysis_prompt)
```

**Phase 2: Synthesis Decision**
```python
async def decide_synthesis_strategy(
    self,
    analysis: BranchAnalysis
) -> SynthesisStrategy:
    if analysis.has_clear_winner:
        return SynthesisStrategy.WINNER_TAKE_ALL

    if analysis.branches_are_complementary:
        # Different branches solved different parts well
        return SynthesisStrategy.CHERRY_PICK

    if analysis.branches_conflict:
        # Incompatible approaches, need to choose
        return SynthesisStrategy.VOTE_AND_SELECT

    # All branches similar quality, combine insights
    return SynthesisStrategy.DEEP_SYNTHESIS
```

**Phase 3: Execute Synthesis**
```python
async def execute_synthesis(
    self,
    branches: list[BranchResult],
    strategy: SynthesisStrategy,
    analysis: BranchAnalysis
) -> FinalSolution:
    if strategy == SynthesisStrategy.WINNER_TAKE_ALL:
        winner = max(branches, key=lambda b: b.verification_score)
        return FinalSolution(
            code=winner.code,
            source_branches=[winner.id],
            method="winner"
        )

    if strategy == SynthesisStrategy.CHERRY_PICK:
        synthesis_prompt = f"""
        Create a final implementation by combining the best parts of these branches:

        {self.format_branches_with_analysis(branches, analysis)}

        Specifically:
        {self.format_cherry_pick_instructions(analysis)}

        Produce the final combined implementation.
        """

        combined_code = await self.llm.generate(synthesis_prompt)
        return FinalSolution(
            code=combined_code,
            source_branches=[b.id for b in branches],
            method="cherry_pick"
        )

    # ... other strategies
```

---

## 5. Implementation Phases

### Phase 1: Foundation (Estimated: 2-3 weeks development)

**Objective**: Build core infrastructure without changing default behavior

**Deliverables**:

1. **Difficulty Estimator Module** (`auto-claude/parallel/difficulty.py`)
   - Feature extraction from subtask + context
   - Scoring algorithm with configurable weights
   - Historical data integration with recovery manager
   - Unit tests with synthetic subtasks

2. **Parallel Runner Infrastructure** (`auto-claude/parallel/runner.py`)
   - Async branch execution with proper isolation
   - Git worktree per branch (nested under spec worktree)
   - Timeout and resource limits per branch
   - Clean branch cleanup on completion/failure

3. **Configuration System** (`auto-claude/parallel/config.py`)
   - Feature flags for gradual rollout
   - Tunable parameters (branch count, thresholds)
   - Per-project override capability

**Integration Points**:
- New `--parallel-mode` flag for `run.py`
- Configuration in `.auto-claude/parallel.json`
- Metrics logging for A/B comparison

**Success Criteria**:
- Can spawn 3 parallel branches for a subtask
- Branches execute independently
- One branch failure doesn't affect others
- Clean resource cleanup

### Phase 2: Insight Broadcasting (Estimated: 2 weeks development)

**Objective**: Enable real-time communication between parallel branches

**Deliverables**:

1. **Insight Broadcaster** (`auto-claude/parallel/broadcaster.py`)
   - In-memory pub/sub for insight sharing
   - Insight deduplication and filtering
   - Branch-aware context injection

2. **Agent Integration**
   - Modify `prompts/coder.md` to emit insights
   - Add insight consumption to prompt generation
   - Create `prompts/parallel_coder.md` variant

3. **Insight Extraction Enhancement** (`auto-claude/analysis/insight_extractor.py`)
   - Real-time extraction (not just post-session)
   - Structured insight format for machine consumption
   - Priority scoring for insights

**Integration Points**:
- Hook into agent message stream
- Extend memory system for cross-branch insights
- Add insight timeline visualization

**Success Criteria**:
- Insights broadcast within 2 seconds of discovery
- Other branches receive and acknowledge insights
- Demonstrated behavior change from shared insights

### Phase 3: Continuous Verification (Estimated: 2-3 weeks development)

**Objective**: Prune invalid branches early through incremental verification

**Deliverables**:

1. **Checkpoint System** (`auto-claude/parallel/checkpoints.py`)
   - Language-specific checkpoint definitions
   - Checkpoint execution with timeout
   - Result classification (fatal/warning/info)

2. **Branch Health Monitor** (`auto-claude/parallel/health.py`)
   - Track verification results per branch
   - Calculate branch "health score"
   - Pruning decisions based on relative health

3. **Agent File Hooks**
   - Intercept file write operations
   - Trigger appropriate checkpoints
   - Handle checkpoint results

**Integration Points**:
- File system watcher or agent output parsing
- Integration with existing verification types
- Checkpoint results feed into broadcaster

**Success Criteria**:
- Syntax errors caught within 5 seconds of file write
- Invalid branches pruned before consuming full context
- 30%+ token savings on pruned branches

### Phase 4: Synthesis Engine (Estimated: 3-4 weeks development)

**Objective**: Combine best elements from multiple branches

**Deliverables**:

1. **Branch Analyzer** (`auto-claude/parallel/analyzer.py`)
   - Compare implementations across branches
   - Identify unique contributions per branch
   - Detect conflicts and compatibilities

2. **Synthesizer** (`auto-claude/parallel/synthesizer.py`)
   - Multiple synthesis strategies
   - Strategy selection algorithm
   - Final solution generation

3. **Synthesis Prompts** (`auto-claude/prompts/synthesizer.md`)
   - Branch comparison prompt
   - Cherry-pick synthesis prompt
   - Deep synthesis prompt

**Integration Points**:
- Post-parallel-execution hook
- Integration with QA reviewer
- Synthesis audit trail for debugging

**Success Criteria**:
- Successful synthesis of 2+ branches
- Synthesized solution passes verification
- Quality improvement over single-branch (measurable)

### Phase 5: Adaptive Routing (Estimated: 2 weeks development)

**Objective**: Automatically route subtasks to optimal execution strategy

**Deliverables**:

1. **Router** (`auto-claude/parallel/router.py`)
   - Decision logic based on difficulty
   - Strategy selection with fallback
   - Routing audit trail

2. **Learning System** (optional)
   - Track routing decisions and outcomes
   - Adjust thresholds based on results
   - Per-project learning

3. **Monitoring Dashboard**
   - Routing decision visualization
   - Success rate by strategy
   - Cost comparison

**Integration Points**:
- Replace direct subtask execution in coder.py
- Configuration for default strategy
- Override capability per subtask

**Success Criteria**:
- Automatic routing with no user intervention
- Appropriate strategy selection (validated manually)
- No regression on easy tasks

### Phase 6: Production Hardening (Estimated: 2 weeks development)

**Objective**: Production-ready with comprehensive error handling

**Deliverables**:

1. **Error Recovery**
   - Graceful degradation to sequential on parallel failure
   - Resource cleanup on crash
   - State recovery on restart

2. **Observability**
   - Structured logging for all parallel operations
   - Metrics for monitoring (branch count, pruning rate, etc.)
   - Integration with existing Linear updates

3. **Documentation**
   - User guide for parallel mode
   - Configuration reference
   - Troubleshooting guide

**Success Criteria**:
- 99%+ reliability in parallel mode
- Clear error messages and recovery paths
- Comprehensive documentation

---

## 6. Cost-Benefit Analysis

### 6.1 Token Cost Model

**Sequential Execution (Current)**:
```
Expected tokens = T_base × (1 + failure_rate × avg_retries)

Where:
- T_base = 50,000 tokens (typical subtask)
- failure_rate = 0.3 (30% fail first try)
- avg_retries = 1.5

Expected = 50,000 × (1 + 0.3 × 1.5) = 72,500 tokens
```

**Parallel Execution (Proposed)**:
```
Expected tokens = T_base × N × (1 - early_prune_rate) + T_synthesis

Where:
- N = 3 branches
- early_prune_rate = 0.3 (30% pruned before completion)
- T_synthesis = 10,000 tokens

Expected = 50,000 × 3 × 0.7 + 10,000 = 115,000 tokens
```

**Raw Comparison**: Parallel uses ~1.6× more tokens

### 6.2 Success Rate Improvement

| Difficulty | Sequential P(success) | Parallel P(success) | Improvement |
|------------|----------------------|---------------------|-------------|
| Easy       | 95%                  | 98%                 | +3%         |
| Medium     | 70%                  | 91%                 | +21%        |
| Hard       | 40%                  | 78%                 | +38%        |
| Very Hard  | 15%                  | 52%                 | +37%        |

### 6.3 Total Cost of Ownership

**Sequential (with human intervention)**:
```python
# Easy tasks (40% of workload)
easy_cost = 0.4 × 50,000 × 1.05 × $0.015/1K = $0.32

# Medium tasks (35% of workload)
medium_cost = 0.35 × 72,500 × $0.015/1K + 0.35 × 0.1 × $50 = $0.38 + $1.75 = $2.13

# Hard tasks (20% of workload)
hard_cost = 0.2 × 100,000 × $0.015/1K + 0.2 × 0.3 × $50 = $0.30 + $3.00 = $3.30

# Very hard tasks (5% of workload)
very_hard_cost = 0.05 × 150,000 × $0.015/1K + 0.05 × 0.6 × $50 = $0.11 + $1.50 = $1.61

Total per subtask: $7.36
```

**Parallel (with adaptive routing)**:
```python
# Easy tasks → Sequential (no change)
easy_cost = $0.32

# Medium tasks → Sequential with fast verification
medium_cost = 0.35 × 60,000 × $0.015/1K + 0.35 × 0.05 × $50 = $0.32 + $0.88 = $1.20

# Hard tasks → Parallel (2 branches)
hard_cost = 0.2 × 85,000 × $0.015/1K + 0.2 × 0.1 × $50 = $0.26 + $1.00 = $1.26

# Very hard tasks → Parallel (3 branches)
very_hard_cost = 0.05 × 125,000 × $0.015/1K + 0.05 × 0.2 × $50 = $0.09 + $0.50 = $0.59

Total per subtask: $3.37
```

**Savings**: 54% reduction in total cost, primarily from avoided human intervention

### 6.4 Break-Even Analysis

Parallel becomes cost-effective when:
```
N × P_sequential < P_parallel

For N=2: Sequential success < 50% → use parallel
For N=3: Sequential success < 33% → use parallel
```

Given the difficulty estimator, we can route appropriately.

---

## 7. Risk Mitigation

### 7.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Git conflicts between branches | Medium | High | Separate worktrees, careful merge |
| Resource exhaustion (memory) | Medium | Medium | Branch limits, cleanup on prune |
| Insight broadcasting latency | Low | Medium | Async design, timeout fallbacks |
| Synthesis produces worse code | Medium | High | Verification gate, fallback to winner |

### 7.2 Operational Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Increased complexity | High | Medium | Comprehensive logging, documentation |
| Debugging difficulty | Medium | Medium | Branch audit trails, replay capability |
| Cost overruns | Low | High | Conservative routing thresholds, monitoring |

### 7.3 Rollback Strategy

Each phase is behind a feature flag:
```python
PARALLEL_FEATURES = {
    "difficulty_estimation": True,   # Phase 1
    "parallel_execution": True,      # Phase 1
    "insight_broadcasting": False,   # Phase 2
    "continuous_verification": False, # Phase 3
    "synthesis": False,              # Phase 4
    "adaptive_routing": False,       # Phase 5
}
```

Instant rollback: Set all to `False` to revert to sequential behavior.

---

## 8. Success Metrics

### 8.1 Primary Metrics

| Metric | Current Baseline | Target | Measurement |
|--------|------------------|--------|-------------|
| Subtask success rate | ~70% | >90% | Completed / Attempted |
| Stuck task rate | ~10% | <3% | Stuck / Total |
| Avg attempts per subtask | 1.5 | <1.2 | Total attempts / Subtasks |
| Human intervention rate | ~15% | <5% | Interventions / Specs |

### 8.2 Secondary Metrics

| Metric | Current | Target | Notes |
|--------|---------|--------|-------|
| Token cost per subtask | 72,500 | 80,000 (with higher success) | May increase slightly |
| Wall-clock time | 5 min | 4 min | Parallel reduces latency |
| Code quality score | N/A | Measurable improvement | Via linting, review |

### 8.3 Measurement Plan

1. **A/B Testing**: Run parallel mode on 50% of new specs
2. **Cohort Analysis**: Compare similar tasks across modes
3. **User Feedback**: Survey on code quality perception
4. **Automated Quality**: Lint scores, test coverage delta

---

## 9. Open Questions

### 9.1 Architecture Decisions Needed

1. **Branch Isolation Level**
   - Option A: Separate Git worktrees (full isolation, higher overhead)
   - Option B: In-memory diff tracking (lighter weight, complex merge)
   - Option C: Separate directories without Git (simple, loses Git benefits)

2. **Insight Broadcasting Granularity**
   - Option A: File-level (when file is saved)
   - Option B: Function-level (finer grain, more overhead)
   - Option C: Agent-decided (flexible, less predictable)

3. **Synthesis Trigger**
   - Option A: After all branches complete
   - Option B: After N branches complete (first-to-finish variants)
   - Option C: Continuous synthesis as branches progress

### 9.2 Research Questions

1. **Optimal Branch Count**: Is 3 branches optimal, or should it vary more dynamically?
2. **Approach Diversification**: How to ensure branches are sufficiently different?
3. **Insight Value**: Which insight types provide most value? Should we prioritize?

### 9.3 Integration Questions

1. **Graphiti Integration**: Should cross-branch insights persist to knowledge graph?
2. **Linear Updates**: How to represent parallel progress in Linear?
3. **UI Visualization**: How to show parallel branches in Electron UI?

---

## Appendix A: File Structure

```
auto-claude/
├── parallel/
│   ├── __init__.py
│   ├── config.py           # Configuration management
│   ├── difficulty.py       # Difficulty estimation
│   ├── router.py           # Execution strategy routing
│   ├── runner.py           # Parallel branch execution
│   ├── broadcaster.py      # Insight broadcasting
│   ├── checkpoints.py      # Continuous verification
│   ├── health.py           # Branch health monitoring
│   ├── analyzer.py         # Branch comparison
│   ├── synthesizer.py      # Solution synthesis
│   └── metrics.py          # Observability
├── prompts/
│   ├── parallel_coder.md   # Parallel-aware coder prompt
│   └── synthesizer.md      # Synthesis prompts
└── tests/
    └── parallel/
        ├── test_difficulty.py
        ├── test_runner.py
        ├── test_broadcaster.py
        ├── test_checkpoints.py
        └── test_synthesizer.py
```

## Appendix B: Configuration Schema

```json
{
  "$schema": "parallel-config-v1",
  "enabled": true,
  "routing": {
    "thresholds": {
      "sequential_max": 0.3,
      "parallel_2_max": 0.6,
      "parallel_3_max": 0.8
    },
    "default_strategy": "adaptive"
  },
  "execution": {
    "max_branches": 3,
    "branch_timeout_seconds": 600,
    "early_termination_on_success": true
  },
  "verification": {
    "checkpoints_enabled": true,
    "prune_on_fatal": true,
    "broadcast_warnings": true
  },
  "synthesis": {
    "strategy": "auto",
    "require_verification": true,
    "fallback_to_winner": true
  },
  "insights": {
    "broadcast_enabled": true,
    "max_insights_per_branch": 20,
    "insight_ttl_seconds": 300
  }
}
```

## Appendix C: Glossary

| Term | Definition |
|------|------------|
| Branch | A parallel execution path exploring one approach |
| Checkpoint | A verification step run during implementation |
| Insight | A discovery shared between branches |
| Pruning | Early termination of an invalid branch |
| Synthesis | Combining elements from multiple branches |
| TTRL | Test-Time Reinforcement Learning (DeepMind) |

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-12-30 | Claude | Initial draft |

