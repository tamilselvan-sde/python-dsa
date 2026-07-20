# Self-Reflection, Critique & Self-Correction

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>

## 1. What is it?
Self-reflection is an agent's ability to evaluate its own outputs, identify errors, and correct them autonomously. It mirrors human metacognition — thinking about one's own thinking. Techniques include self-critique (identifying flaws in own reasoning), reflection (re-analyzing after action), and self-correction (fixing errors without external feedback).

## 2. Why do we need it?
LLMs hallucinate, make reasoning errors, and produce low-quality outputs. In agentic systems, a wrong action can cascade into catastrophic failures. Self-reflection adds a quality gate: the agent double-checks its work, catches mistakes, and improves before the error propagates. It reduces the need for human review.

## 3. Real-world Example
**Meta's Cicero (Diplomacy AI)**: Before each move, Cicero generates K candidate actions, evaluates each by simulating opponent reactions, reflects on which is most likely to succeed, and self-corrects its strategy. This reflection loop was critical to achieving human-level performance in a game requiring negotiation.

## 4. Architecture Diagram (ASCII)
```
+------------------+
|   Agent Output   |
+------------------+
         |
         v
+-------------------------------+
|       REFLECTION ENGINE        |
|                                |
|  +----------+ +------------+  |
|  | Critic   | | Verifier   |  |
|  | (LLM)    | | (Rules)    |  |
|  +----------+ +------------+  |
|  +----------+ +------------+  |
|  | Scorer   | | Corrector  |  |
|  | (0-1)    | | (Generate  |  |
|  |          | |  Fix)      |  |
|  +----------+ +------------+  |
+-------------------------------+
         |
   +-----+------+
   |            |
[Pass]      [Fail]
   |            |
   v            v
[Output]   [Self-Correct]
                |
           [Re-generate]
```

## 5. Internal Working
The agent produces an output (action, response, plan). The reflection module evaluates it: (1) self-critique — the LLM is prompted to find flaws in its own reasoning, (2) rule-based verification — check constraints, schema, safety rules, (3) scoring — produce a quality score. If below threshold, the reflection module provides feedback to the agent, which regenerates. This repeats until quality threshold is met or max iterations reached.

## 6. Production Flow
```
1. Agent produces candidate output
2. Critic LLM analyzes output for errors, inconsistencies, omissions
3. Rule verifier checks format, schema, safety constraints
4. Scorer produces composite quality score
5. If score >= threshold: return output
6. If score < threshold: critic generates specific improvement instructions
7. Agent re-generates incorporating feedback
8. Repeat up to N iterations (typically 2-3)
9. If no improvement in last iteration: return best-so-far
```

## 7. HLD
```
+------------------+
|   Agent           |
+------------------+
         | output
         v
+---------------------------+
|    Reflection Service      |
|  +--------+ +----------+  |
|  | Critic | | Verifier |  |
|  | (LLM)  | | (Rules)  |  |
|  +--------+ +----------+  |
|  +--------+ +----------+  |
|  | Scorer | | Corrector|  |
|  +--------+ +----------+  |
+---------------------------+
         | feedback
         v
+------------------+
|   Agent (retry)  |
+------------------+
```

## 8. LLD
```python
from pydantic import BaseModel, Field
from enum import Enum
from typing import Any
import asyncio

class ReflectionResult(str, Enum):
    PASS = "pass"
    FAIL = "fail"
    NEEDS_IMPROVEMENT = "needs_improvement"

class Critique(BaseModel):
    issue: str
    severity: int  # 1-5, 5 being critical
    suggestion: str
    category: str  # "factual", "logic", "safety", "format", "completeness"

class ReflectionOutput(BaseModel):
    score: float  # 0.0 to 1.0
    result: ReflectionResult
    critiques: list[Critique] = Field(default_factory=list)
    summary: str = ""

class SelfReflectionEngine:
    def __init__(self, critic_llm, scorer_llm, max_iterations=3, pass_threshold=0.8):
        self.critic = critic_llm
        self.scorer = scorer_llm
        self.max_iterations = max_iterations
        self.pass_threshold = pass_threshold

    async def reflect(self, agent_output: str, context: dict) -> ReflectionOutput:
        critiques = await self._generate_critique(agent_output, context)
        score = await self._score_quality(agent_output, critiques, context)
        
        result = ReflectionResult.PASS if score >= self.pass_threshold else \
                 ReflectionResult.FAIL if score < 0.3 else \
                 ReflectionResult.NEEDS_IMPROVEMENT
        
        return ReflectionOutput(
            score=score,
            result=result,
            critiques=critiques,
            summary=await self._summarize_feedback(critiques)
        )

    async def self_correct(self, agent, input_data: dict, context: dict) -> tuple[str, list[ReflectionOutput]]:
        best_output = None
        best_score = 0.0
        reflections = []

        for i in range(self.max_iterations):
            if i == 0:
                output = await agent.generate(input_data)
            else:
                output = await agent.regenerate(input_data, reflections[-1])
            
            reflection = await self.reflect(output, context)
            reflections.append(reflection)
            
            if reflection.score > best_score:
                best_output = output
                best_score = reflection.score
            
            if reflection.result == ReflectionResult.PASS:
                return output, reflections
        
        return best_output, reflections

    async def _generate_critique(self, text: str, context: dict) -> list[Critique]:
        response = await self.critic.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": """You are a critical reviewer. Analyze the following agent output for:
                1. Factual errors or hallucinations
                2. Logical inconsistencies
                3. Missing information
                4. Safety concerns
                5. Format/schema violations
                
                Return specific, actionable critiques as JSON."""},
                {"role": "user", "content": text},
                {"role": "user", "content": f"Context: {json.dumps(context)}"}
            ],
            response_format={"type": "json_object"}
        )
        critiques_data = json.loads(response.choices[0].message.content)
        return [Critique(**c) for c in critiques_data.get("critiques", [])]

    async def _score_quality(self, text: str, critiques: list[Critique], context: dict) -> float:
        if not critiques:
            return 1.0
        
        max_severity = max(c.severity for c in critiques) if critiques else 0
        base_score = 1.0 - (len(critiques) * 0.1) - (max_severity * 0.15)
        
        response = await self.scorer.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": 
                f"Rate the quality of this output (0.0 to 1.0):\n{text}\n"
                f"Critiques found: {len(critiques)}\n"
                f"Max severity: {max_severity}\nReturn only a number."}]
        )
        llm_score = float(response.choices[0].message.content.strip())
        
        return max(0.0, min(1.0, (base_score + llm_score) / 2))


class RuleVerifier:
    def __init__(self):
        self.rules: list[tuple[str, callable]] = []

    def add_rule(self, name: str, check_fn: callable):
        self.rules.append((name, check_fn))

    async def verify(self, output: str) -> list[Critique]:
        critiques = []
        for name, fn in self.rules:
            try:
                if asyncio.iscoroutinefunction(fn):
                    passed = await fn(output)
                else:
                    passed = fn(output)
                if not passed:
                    critiques.append(Critique(
                        issue=f"Rule violation: {name}",
                        severity=4,
                        suggestion=f"Fix {name}",
                        category="format"
                    ))
            except Exception as e:
                critiques.append(Critique(
                    issue=f"Rule check error: {name}",
                    severity=3,
                    suggestion=str(e),
                    category="format"
                ))
        return critiques
```

## 9. Python Implementation
```python
from fastapi import FastAPI
from pydantic import BaseModel
import json

app = FastAPI()

class ReflexionAgent:
    """Implements the Reflexion pattern: actor + evaluator + self-reflection"""
    
    def __init__(self, actor_llm, evaluator_llm, max_trials=3):
        self.actor = actor_llm
        self.evaluator = evaluator_llm
        self.max_trials = max_trials
        self.memory = []

    async def solve(self, task: str) -> dict:
        trial_results = []
        
        for trial in range(self.max_trials):
            # Actor generates solution
            solution = await self._act(task, trial_results)
            
            # Evaluator provides feedback
            feedback = await self._evaluate(task, solution)
            
            trial_results.append({
                "trial": trial + 1,
                "solution": solution,
                "feedback": feedback
            })
            
            if feedback.get("is_correct", False):
                return {
                    "solution": solution,
                    "trials": trial_results,
                    "converged": True
                }
        
        # Return best effort
        best = max(trial_results, key=lambda t: t["feedback"].get("score", 0))
        return {
            "solution": best["solution"],
            "trials": trial_results,
            "converged": False
        }

    async def _act(self, task: str, history: list) -> str:
        msgs = [{"role": "system", "content": "Solve the task."}]
        msgs.append({"role": "user", "content": task})
        
        for h in history[-1:]:  # Only last trial for context
            msgs.append({"role": "assistant", "content": h["solution"]})
            msgs.append({
                "role": "user", 
                "content": f"Previous attempt feedback: {h['feedback']}\n\nImprove your solution."
            })
        
        resp = await self.actor.chat.completions.create(model="gpt-4o", messages=msgs)
        return resp.choices[0].message.content

    async def _evaluate(self, task: str, solution: str) -> dict:
        resp = await self.evaluator.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": """Evaluate the solution for:
                - Correctness (0-1)
                - Completeness (0-1)
                - Safety (0-1)
                Return JSON with fields: is_correct, score, issues, improvement_hint"""},
                {"role": "user", "content": f"Task: {task}\nSolution: {solution}"}
            ],
            response_format={"type": "json_object"}
        )
        return json.loads(resp.choices[0].message.content)


# --- Code-Specific Reflection ---

class CodeReflector:
    def __init__(self, llm, test_runner):
        self.llm = llm
        self.runner = test_runner

    async def reflect_on_code(self, code: str, tests: list) -> tuple[str, list[str]]:
        issues = []
        
        # 1. Syntax check
        try:
            compile(code, "<string>", "exec")
        except SyntaxError as e:
            return code, [f"Syntax error: {e}"]
        
        # 2. Self-critique
        critique = await self._critique_code(code)
        issues.extend(critique)
        
        # 3. Run tests if available
        if tests:
            test_results = await self.runner.run_tests(code, tests)
            failed = [t for t in test_results if not t["passed"]]
            issues.extend([f"Test failed: {t['name']}: {t['error']}" for t in failed])
        
        # 4. Self-correct
        if issues:
            code = await self._fix_code(code, issues)
        
        return code, issues

    async def _critique_code(self, code: str) -> list[str]:
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Review this code. Find bugs, edge cases, style issues, "
                f"and security problems. Return JSON array of strings:\n{code}"}],
            response_format={"type": "json_object"}
        )
        return json.loads(response.choices[0].message.content).get("issues", [])

    async def _fix_code(self, code: str, issues: list[str]) -> str:
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": 
                f"Fix these issues in the code:\nIssues: {issues}\n\nCode:\n{code}\n\n"
                f"Return only the fixed code, no explanation."}]
        )
        return response.choices[0].message.content
```

## 10. Folder Structure
```
reflection/
├── engine/
│   ├── self_reflection.py     # Core reflection engine
│   ├── critic.py              # LLM-based critic
│   ├── scorer.py              # Quality scoring
│   └── corrector.py           # Self-correction module
├── verifiers/
│   ├── rule_verifier.py       # Rule-based constraints
│   ├── schema_verifier.py     # JSON Schema validation
│   ├── safety_verifier.py     # Content safety checks
│   └── consistency_checker.py # Factual consistency
├── strategies/
│   ├── reflexion.py           # Reflexion pattern (Shinn et al.)
│   ├── self_consistency.py    # Majority voting
│   └── chain_of_verification.py
└── api/
    └── routes.py
```

## 11. Configuration
```yaml
reflection:
  enabled: true
  max_iterations: 3
  pass_threshold: 0.8
  
  critic:
    model: gpt-4o
    temperature: 0.3
    max_tokens: 1024
  
  scorer:
    model: gpt-4o-mini
    temperature: 0.0
  
  verifier:
    rules:
      - no_profanity
      - json_valid
      - no_code_injection
      - max_length: 4000
  
  correction:
    model: gpt-4o
    temperature: 0.5
    include_context: true
```

## 12. Flowchart
```
[Generate Output]
       |
  [Critic Evaluation]
       |
  [Score > threshold?] --Yes--> [Return Output]
       |
       No
       |
  [Critique < max_iter?] --No--> [Return Best]
       |
       Yes
       |
  [Generate Improvement Plan]
       |
  [Re-generate with Feedback]
       |
       +----------> [Loop]
```

## 13. Sequence Diagram
```
Agent         Critic_LLM      Scorer       Verifier      Memory
  |              |              |              |           |
  |--output----->|              |              |           |
  |              |--analyze---->|              |           |
  |              |<--critiques--|              |           |
  |              |              |              |           |
  |              |              |--score------>|           |
  |              |              |<--score------|           |
  |              |              |              |           |
  |              |--check------>|              |           |
  |              |<--violations-|              |           |
  |              |              |              |           |
  |<--reflection-|              |              |           |
  |              |              |              |           |
  |--store------>|              |              |           |
  |              |              |              |           |
  |--re-generate-|              |              |           |
  |  (if needed) |              |              |           |
```

## 14. Pros
- Catches errors before they reach users
- Reduces hallucination rates significantly (30-50%)
- No external data needed (self-contained)
- Improves iteratively without human feedback
- Works across all output types (code, text, plans)

## 15. Cons
- 2-3x more LLM calls per task (cost + latency)
- Can over-correct correct outputs (false positives)
- Critic LLM may miss subtle errors
- Reflection can converge to mediocre local optima
- Adds latency: each reflection iteration = 1-2 more LLM calls

## 16. Alternatives
- **Human-in-the-loop** — gold standard but slow
- **Constitutional AI** — fixed principles, no LLM critic
- **Ensemble methods** — multiple independent generations, pick best
- **External verification** — unit tests, formal verification
- **Re-ranking** — generate N candidates, score and pick best (no correction)

## 17. Performance Considerations
- Use cheaper model for critic (GPT-4o-mini) and expensive for actor (GPT-4o)
- Max 2-3 reflection iterations — diminishing returns after
- Cache reflection results for identical inputs
- Run reflection asynchronously in parallel wherever possible
- Skip reflection for trivial outputs (low-risk, high-confidence)

## 18. Scaling to Millions
```
Reflection Pipeline:
  - Dedicated critic pool (separate from actor pool)
  - Reflection jobs queued (Kafka) for background processing
  - Async reflection for non-blocking agent loop
  - Tiered reflection: full (expensive) for high-risk, quick (cheap) for low-risk

Caching:
  - Reflection results cache (identical input -> skip reflection)
  - Critic prompt cache (frequent patterns get pre-computed critiques)
  - Score cache (same output type, same score pattern)

Optimization:
  - Batch reflection: reflect on N outputs in one LLM call
  - Speculative reflection: predict likely critiques before generation
```

## 19. Failure Scenarios
| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Critic over-rejects | Good output discarded | Dual critic: LLM + rules, pass if either passes |
| Critic misses error | Bad output reaches user | Multiple verifier layers |
| Correction degrades | Worse output | Track score trajectory, reject if score drops |
| Reflection loops | Token waste | Hard limit of 3 iterations |
| Critic hallucinates | False positives | Verify critiques with rules before acting |

## 20. Security
- Reflection must not leak internal state in critique output
- Critic prompts could be injection targets — sanitize agent output before critique
- Reflection loop can be used for jailbreaking — monitor for adversarial inputs
- Rate limit reflection calls per session (prevent abuse of looping)

## 21. Monitoring
```prometheus
reflection_iterations_per_output
reflection_pass_ratio
reflection_improvement_delta (score after - score before)
reflection_false_positive_rate
reflection_latency_ms
reflection_correction_success_rate
reflection_critic_agreement_rate
reflection_avg_critiques_per_output
```

## 22. Interview Questions
**Q1**: How does self-reflection compare to supervised fine-tuning for quality? (Reflection is dynamic per-output, SFT is static; reflection catches novel errors, SFT prevents known ones)

**Q2**: How many reflection iterations is optimal? (2-3 empirically. First pass catches gross errors, second catches subtleties, third rarely adds value)

**Q3**: How do you prevent reflection from making correct outputs worse? (Score tracking: reject corrections that lower the score. Keep the best-so-far)

## 23. Cheat Sheet
```
Reflection = Generate -> Critique -> Score -> Correct -> Repeat
Critique types: factual, logical, safety, format, completeness
Score range: 0.0-1.0, threshold typically 0.7-0.8
Max iterations: 2-3 (diminishing returns after)
Best practice: cheap critic (4o-mini), expensive actor (4o)
Fallback: best-of-N generation (simpler, no reflection)
```

## 24. Common Mistakes
- Critic and actor use same model — same blind spots
- No score floor — correction can degrade quality
- Too many iterations — token costs explode, quality plateaus
- No caching — reflecting on identical outputs repeatedly
- Over-reflecting on simple outputs (just add latency, no value)
- Not tracking score trajectory — can oscillate between two bad solutions

## 25. Production Best Practices
1. Track quality scores over time — detect degradation trends
2. Use tiered reflection: brief for simple, comprehensive for complex
3. Always keep best-so-far output, not just latest
4. Log full reflection traces for post-hoc analysis
5. Separate critic and actor models (different biases = better coverage)
6. Combine LLM critic with rule-based verifier (best of both)
7. Set token budget per reflection cycle to bound costs
8. A/B test with/without reflection — quantify the quality improvement
