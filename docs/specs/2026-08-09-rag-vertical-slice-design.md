# RAG Diagnostic Vertical Slice — Design

Issue: #4 ("RAG diagnostic vertical slice"), child of epic #3 ("RootCause AI diagnostic engine").

## Goal

Given the seeded incident "our RAG assistant's answer quality dropped", run a
LangGraph-orchestrated investigation that discovers — from evidence, not a
hard-coded branch — that the embedding model changed without re-indexing the
vector store, and produce a verified, evidence-cited root cause plus a
remediation recommendation.

## Package layout

```
src/rootcause_ai/
  domain/
    models.py         # Incident, Hypothesis, Evidence, TraceEvent, RootCauseProposal,
                       # VerificationResult, Remediation — Pydantic BaseModel
  graph/
    state.py           # DiagnosticState (Pydantic BaseModel state schema)
    graph.py            # build_graph() -> CompiledStateGraph
    routing.py          # evidence_sufficient(), verification_passed() conditional-edge fns
    nodes/
      classify_incident.py
      select_skill.py
      generate_hypotheses.py
      select_next_investigation.py
      run_investigation.py
      evaluate_evidence.py
      propose_root_cause.py
      verify_root_cause.py
      generate_remediation.py
  agents/
    investigator.py     # build_investigator(skill, tools) -> decision-making chain
    verifier.py          # build_verifier() -> adversarial-check chain
  skills/
    troubleshoot_rag_quality/
      skill.py            # SKILL: Skill dataclass/const (loaded, not executed)
  tools/
    base.py               # shared Pydantic tool-arg/result base models
    rag_metrics.py         # get_rag_metrics
    embedding_config.py    # get_embedding_config
    vector_index.py        # get_vector_index_metadata
    deployments.py         # get_recent_deployments
    git_changes.py         # get_git_changes
    runbooks.py            # search_runbooks
  customers/
    acme_ai/
      seed_data/
        rag_metrics.json
        embedding_config.json
        vector_index.json
        deployments.json
        git_changes.json
        runbooks/*.md
  llm.py                  # get_chat_model(name: str = "default") -> ChatAnthropic
tests/
  domain/  graph/  tools/  skills/  agents/
```

`pyproject.toml`: rename `[tool.uv.build-backend] module-name` from
`example_app` to `rootcause_ai`; `uv add langgraph langchain langchain-anthropic
pydantic` (pydantic already present).

## Model configuration

`src/rootcause_ai/llm.py` reads `ROOTCAUSE_AI_MODEL` (default
`"claude-sonnet-5"`) via `python-dotenv` + `os.environ`, per this repo's
`load-dotenv-first` rule. `get_chat_model("verifier")` allows a per-role
override (`ROOTCAUSE_AI_VERIFIER_MODEL`) so the Verifier can be bumped to
`claude-opus-5` independently later without a code change — the seam exists
now, the override isn't exercised by default. LLM calls needing real API
access are marked `@pytest.mark.integration` and excluded from `make tests`.

## Domain models (`domain/models.py`)

```python
class IncidentClassification(BaseModel):
    category: Literal["rag_quality", "api_latency", "data_pipeline"]
    confidence: float
    rationale: str

class Hypothesis(BaseModel):
    id: str                      # "H1", "H2", ...
    statement: str
    status: Literal["untested", "supported", "rejected"] = "untested"
    confidence: float = 0.0

class Evidence(BaseModel):
    id: str                      # "E1", "E2", ...
    source_tool: str
    summary: str
    data: dict[str, Any]         # raw structured tool output
    supports: list[str]          # hypothesis ids
    contradicts: list[str]       # hypothesis ids

class ToolCall(BaseModel):
    tool_name: str
    args: dict[str, Any]
    target_hypothesis: str
    rationale: str

class TraceEvent(BaseModel):
    timestamp: datetime
    kind: Literal["classification", "skill_loaded", "hypotheses_generated",
                   "tool_called", "evidence_added", "verification", "root_cause",
                   "remediation"]
    message: str
    data: dict[str, Any] = {}

class RootCauseProposal(BaseModel):
    hypothesis_id: str
    statement: str
    confidence: float
    cited_evidence: list[str]    # evidence ids — every id must exist in the ledger

class VerificationResult(BaseModel):
    passed: bool
    checks: list[str]            # deterministic check descriptions, pass/fail noted inline
    missing_evidence: str | None
    verdict_rationale: str

class Remediation(BaseModel):
    action: str
    rationale: str
    high_impact: bool            # drives future HITL gating (Phase 3)
```

## Graph state (`graph/state.py`)

`DiagnosticState(BaseModel)` — LangGraph validates a Pydantic state schema on
invoke (confirmed against current LangGraph docs). List fields that
accumulate across loop iterations use the additive reducer:

```python
class DiagnosticState(BaseModel):
    incident: Incident
    classification: IncidentClassification | None = None
    skill_id: str | None = None
    hypotheses: list[Hypothesis] = []
    evidence_ledger: Annotated[list[Evidence], operator.add] = []
    tool_call_history: Annotated[list[ToolCall], operator.add] = []
    trace: Annotated[list[TraceEvent], operator.add] = []
    current_investigation_target: str | None = None
    iteration_count: int = 0
    max_iterations: int = 8
    root_cause_proposal: RootCauseProposal | None = None
    verification: VerificationResult | None = None
    remediation: Remediation | None = None
    status: Literal["investigating", "verifying", "confirmed", "inconclusive"] = "investigating"
```

`hypotheses` is NOT additive — nodes return the full updated list each time
(status/confidence mutate in place), since hypotheses are looked up and
edited, not appended.

## Graph (`graph/graph.py`)

```
START -> classify_incident -> select_skill -> generate_hypotheses
      -> select_next_investigation
      -> run_investigation -> evaluate_evidence
      -> [conditional: evidence_sufficient?]
           NO  -> select_next_investigation (loop)
           YES -> propose_root_cause -> verify_root_cause
      -> [conditional: verification_passed?]
           NO, gap + iterations < max -> select_next_investigation (loop)
           NO, iterations >= max      -> END (status=inconclusive)
           YES -> generate_remediation -> END (status=confirmed)
```

`max_iterations` (default 8) is the loop-safety cap enforced in
`evidence_sufficient` / `verification_passed`. Checkpointer:
`langgraph.checkpoint.memory.InMemorySaver` for this phase (swapped for a
durable backend in Phase 6).

## Node classifications (recap from architecture review)

| Node | Kind |
|---|---|
| `classify_incident` | LangChain agent (single structured-output call) |
| `select_skill` | deterministic Python |
| `generate_hypotheses` | skill-assisted agent |
| `select_next_investigation` | skill-assisted agent (tool-selection reasoning) |
| `run_investigation` | tool execution (deterministic dispatch) |
| `evaluate_evidence` | skill-assisted agent |
| `evidence_sufficient` (edge) | routing/control logic |
| `propose_root_cause` | deterministic Python (argmax confidence over `supported` hypotheses; no new LLM claims) |
| `verify_root_cause` | verifier (deterministic checks + independent LLM judgment) |
| `verification_passed` (edge) | routing/control logic |
| `generate_remediation` | skill-assisted agent |

## Tools (`tools/*.py`)

Each tool is a `@tool`-decorated function (LangChain) returning a Pydantic
model (not a string) serialized via `.model_dump()`, reading from the
matching JSON fixture under `customers/acme_ai/seed_data/`. Read-only, no
side effects, no network calls.

```python
class RagMetrics(BaseModel):
    recall_at_5_before: float
    recall_at_5_after: float
    measured_before: datetime
    measured_after: datetime

@tool
def get_rag_metrics() -> dict:
    """Return retrieval quality metrics (Recall@5) before/after the incident window."""
    ...
```

Seed data for `acme_ai` (the fictional customer) encodes the scenario:
`embedding_config.json` shows `text-embedding-v2 -> text-embedding-v3` at
deployment `dep-2026-08-07`; `vector_index.json` shows `last_rebuilt` predates
that deployment; `rag_metrics.json` shows Recall@5 0.84 -> 0.49 across the same
window; `deployments.json` lists the embedding-model-change deployment;
`git_changes.json` shows no prompt/system-prompt diff in that window (so H2
prompt-regression is falsified); `runbooks/` has a
`reindex-after-embedding-change.md` the remediation step can cite.

## Skill (`skills/troubleshoot_rag_quality/skill.py`)

Not a Claude Code skill — a plain Python module holding the domain-knowledge
text injected into `generate_hypotheses`, `select_next_investigation`,
`evaluate_evidence`, and `generate_remediation` prompts. Content: what RAG
failure modes to consider (retrieval Recall@K, precision, embedding model
changes, index freshness/version mismatch, chunking changes, reranking
changes, prompt/system-prompt changes, LLM/model changes, source-document
availability, ingestion failures) and which tool answers which question.

## Agents (`agents/*.py`)

- **Investigator** (`select_next_investigation`): `ChatAnthropic(...).bind_tools(tools)`
  invoked with state (hypotheses + evidence-so-far + skill text) and asked to
  emit exactly one tool call plus a `target_hypothesis` + `rationale` — the
  LangGraph node then executes that single tool call itself (deterministic
  dispatch in `run_investigation`), keeping the loop visible at the graph
  level rather than hidden inside a multi-step LangChain agent executor.
- **Verifier** (`verify_root_cause`): a separate `ChatAnthropic` instance with
  its own adversarial system prompt ("does the evidence really support this,
  is anything missing, is another hypothesis still plausible, is this
  correlation not causation") — invoked fresh, without the Investigator's
  running context, for genuine independence. Runs after deterministic checks
  (every cited evidence id exists in the ledger; no `rejected` hypothesis is
  cited; no other hypothesis is also `supported`) — the LLM judgment
  supplements, never replaces, those checks.

## Test plan

- `domain/`: model validation (e.g. `RootCauseProposal` citing a nonexistent
  evidence id is a Pydantic validation error where feasible, or a deterministic
  check failure in the verifier otherwise).
- `tools/`: each tool against the `acme_ai` fixtures — pure, deterministic, no
  network.
- `graph/routing.py`: `evidence_sufficient` / `verification_passed` unit
  tests including the `max_iterations` cutoff path.
- `graph/graph.py`: one end-to-end test running the compiled graph against
  the seeded incident with LLM calls mocked (deterministic stand-ins for the
  agent nodes) to keep `make tests` offline; a separate
  `@pytest.mark.integration` test runs the same scenario against the real
  model and asserts the discovered root cause matches
  "embedding model changed without re-indexing" and cites the expected
  evidence ids — this is the one test that proves no hard-coded branch.

## Open items for TDD execution (not blocking the spec)

- Exact `@tool` return-type handling (dict vs Pydantic passthrough) will be
  confirmed against the installed `langchain` version's docs at
  implementation time per this repo's `fetch-library-docs-first` rule.
- Whether `generate_hypotheses` needs `with_structured_output` vs raw
  `bind_tools` for its typed list output will be decided the same way.
