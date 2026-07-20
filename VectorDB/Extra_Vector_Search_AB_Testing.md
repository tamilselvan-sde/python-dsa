# A/B Testing Framework for Vector Search

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>


```python
import random
from typing import Callable, Dict, List
from collections import defaultdict

class VectorSearchABTest:
    """A/B test framework for vector search strategies."""
    
    def __init__(self, 
                 control_search: Callable,
                 variant_search: Callable,
                 experiment_name: str,
                 traffic_percent: float = 0.05):
        
        self.control = control_search
        self.variant = variant_search
        self.name = experiment_name
        self.traffic_percent = traffic_percent
        self.results = defaultdict(list)
    
    def get_searcher(self, user_id: str) -> Callable:
        """Route user to control or variant."""
        if hash(f"{self.name}:{user_id}") % 100 < self.traffic_percent * 100:
            return "variant"
        return "control"
    
    def record(self, 
               user_id: str,
               arm: str,
               query: str,
               results: list,
               latency_ms: float,
               clicked: List[str] = None):
        
        self.results[arm].append({
            "user_id": user_id,
            "query": query,
            "latency_ms": latency_ms,
            "num_results": len(results),
            "clicked": clicked or [],
        })
    
    def analyze(self) -> Dict:
        control = self.results["control"]
        variant = self.results["variant"]
        
        if not control or not variant:
            return {"status": "insufficient_data"}
        
        from scipy import stats
        
        analysis = {
            "experiment": self.name,
            "samples": {
                "control": len(control),
                "variant": len(variant),
            },
            "metrics": {
                "latency": {
                    "control_mean": np.mean([
                        r["latency_ms"] for r in control
                    ]),
                    "variant_mean": np.mean([
                        r["latency_ms"] for r in variant
                    ]),
                    "p_value": stats.ttest_ind(
                        [r["latency_ms"] for r in control],
                        [r["latency_ms"] for r in variant]
                    ).pvalue,
                },
                "click_rate": {
                    "control": np.mean([
                        len(r["clicked"]) / max(r["num_results"], 1)
                        for r in control
                    ]),
                    "variant": np.mean([
                        len(r["clicked"]) / max(r["num_results"], 1)
                        for r in variant
                    ]),
                },
            },
            "recommendation": self._get_recommendation(analysis)
        }
        
        return analysis
    
    def _get_recommendation(self, analysis: Dict) -> str:
        from scipy import stats
        
        ctrl = analysis["metrics"]["latency"]["control_mean"]
        var = analysis["metrics"]["latency"]["variant_mean"]
        p = analysis["metrics"]["latency"]["p_value"]
        
        if p < 0.05 and var < ctrl * 0.9:
            return "ROLL_OUT: Variant significantly faster"
        elif p < 0.05 and var > ctrl * 1.1:
            return "ROLLBACK: Control significantly faster"
        elif p < 0.05:
            return "MONITOR: Statistically different, check other metrics"
        else:
            return "CONTINUE: No statistically significant difference"
```

---

