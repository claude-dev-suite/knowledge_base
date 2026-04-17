# Gradual Rollout -- Feature Flags, Multi-Armed Bandit, and Auto-Rollback

## TL;DR

After validating a RAG pipeline change via shadow mode, the next step is gradually rolling it out to real users. Gradual rollout uses feature flags to incrementally shift traffic from the old pipeline to the new one (1% -> 5% -> 25% -> 100%), monitoring quality metrics at each stage. Advanced approaches use multi-armed bandit algorithms that automatically adjust traffic percentages based on observed performance, and auto-rollback that instantly reverts to the old pipeline when metrics drop below thresholds. This article covers feature flag integration, percentage-based rollout with metric gates, Thompson Sampling for bandit-based allocation, and auto-rollback triggers.

---

## Feature Flag Integration

### Basic Feature Flag for RAG Pipelines

```python
import hashlib
import time
import logging
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)


@dataclass
class RolloutConfig:
    """Configuration for a gradual rollout."""

    feature_name: str
    rollout_percentage: float = 0.0  # 0-100
    enabled_user_ids: list[str] = field(default_factory=list)
    enabled_teams: list[str] = field(default_factory=list)
    start_time: float | None = None
    end_time: float | None = None


class FeatureFlagManager:
    """Manage feature flags for RAG pipeline rollout.

    Supports percentage-based rollout, user/team allowlists,
    and time-based activation.
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self.prefix = "ff:"

    def set_rollout(
        self, feature: str, percentage: float
    ) -> None:
        """Set rollout percentage for a feature."""
        self.redis.hset(
            f"{self.prefix}{feature}",
            "percentage",
            str(percentage),
        )

    def is_enabled(
        self,
        feature: str,
        user_id: str = "",
        team: str = "",
    ) -> bool:
        """Check if a feature is enabled for a given user."""
        config = self.redis.hgetall(f"{self.prefix}{feature}")
        if not config:
            return False

        # Check allowlist
        allowlist = config.get(b"allowlist", b"").decode()
        if user_id and user_id in allowlist:
            return True
        if team and team in allowlist:
            return True

        # Percentage-based rollout (deterministic per user)
        percentage = float(config.get(b"percentage", b"0"))
        if percentage >= 100:
            return True
        if percentage <= 0:
            return False

        # Use hash for deterministic assignment
        hash_input = f"{feature}:{user_id}".encode()
        hash_val = int(hashlib.md5(hash_input).hexdigest(), 16)
        bucket = hash_val % 100

        return bucket < percentage

    def add_to_allowlist(
        self, feature: str, identifier: str
    ) -> None:
        """Add a user or team to the allowlist."""
        key = f"{self.prefix}{feature}"
        current = self.redis.hget(key, "allowlist") or b""
        updated = current.decode() + f",{identifier}"
        self.redis.hset(key, "allowlist", updated)


class FeatureFlaggedRAG:
    """RAG pipeline that routes based on feature flags."""

    def __init__(
        self,
        stable_pipeline,
        new_pipeline,
        flag_manager: FeatureFlagManager,
        feature_name: str = "rag_v2",
    ):
        self.stable = stable_pipeline
        self.new = new_pipeline
        self.flags = flag_manager
        self.feature = feature_name

    def query(
        self, user_query: str, user_id: str = "", team: str = ""
    ) -> dict:
        """Route to stable or new pipeline based on feature flag."""
        use_new = self.flags.is_enabled(
            self.feature, user_id, team
        )

        if use_new:
            result = self.new.query(user_query)
            result["_pipeline"] = "new"
        else:
            result = self.stable.query(user_query)
            result["_pipeline"] = "stable"

        return result
```

---

## Staged Rollout with Metric Gates

### Rollout Stages

```python
from enum import Enum


class RolloutStage(Enum):
    SHADOW = "shadow"         # 0% traffic, comparison only
    CANARY = "canary"         # 1-5% traffic
    EARLY_ADOPTER = "early"   # 10-25% traffic
    GENERAL = "general"       # 50% traffic
    FULL = "full"             # 100% traffic


@dataclass
class StageConfig:
    stage: RolloutStage
    percentage: float
    min_duration_hours: int
    promotion_criteria: dict


ROLLOUT_STAGES = [
    StageConfig(
        stage=RolloutStage.CANARY,
        percentage=5.0,
        min_duration_hours=4,
        promotion_criteria={
            "min_faithfulness": 0.75,
            "max_error_rate": 0.02,
            "max_latency_p99_ms": 8000,
            "min_queries": 100,
        },
    ),
    StageConfig(
        stage=RolloutStage.EARLY_ADOPTER,
        percentage=25.0,
        min_duration_hours=12,
        promotion_criteria={
            "min_faithfulness": 0.75,
            "max_error_rate": 0.02,
            "max_latency_p99_ms": 7000,
            "min_queries": 500,
        },
    ),
    StageConfig(
        stage=RolloutStage.GENERAL,
        percentage=50.0,
        min_duration_hours=24,
        promotion_criteria={
            "min_faithfulness": 0.75,
            "max_error_rate": 0.01,
            "max_latency_p99_ms": 6000,
            "min_queries": 2000,
        },
    ),
    StageConfig(
        stage=RolloutStage.FULL,
        percentage=100.0,
        min_duration_hours=0,
        promotion_criteria={},
    ),
]


class StagedRolloutManager:
    """Manage staged rollout with metric gates between stages."""

    def __init__(
        self,
        flag_manager: FeatureFlagManager,
        metrics_collector,
        feature_name: str = "rag_v2",
    ):
        self.flags = flag_manager
        self.metrics = metrics_collector
        self.feature = feature_name
        self.current_stage_idx = 0
        self.stage_start_time = time.time()

    @property
    def current_stage(self) -> StageConfig:
        return ROLLOUT_STAGES[self.current_stage_idx]

    def check_and_advance(self) -> dict:
        """Check if current stage criteria are met and advance."""
        stage = self.current_stage

        # Check minimum duration
        hours_elapsed = (
            time.time() - self.stage_start_time
        ) / 3600
        if hours_elapsed < stage.min_duration_hours:
            return {
                "action": "wait",
                "reason": (
                    f"Stage {stage.stage.value}: "
                    f"{hours_elapsed:.1f}/{stage.min_duration_hours}h elapsed"
                ),
            }

        # Check metric criteria
        current_metrics = self.metrics.get_current(
            pipeline="new"
        )
        criteria = stage.promotion_criteria

        checks = {}
        if "min_faithfulness" in criteria:
            checks["faithfulness"] = (
                current_metrics.get("faithfulness", 0)
                >= criteria["min_faithfulness"]
            )
        if "max_error_rate" in criteria:
            checks["error_rate"] = (
                current_metrics.get("error_rate", 1)
                <= criteria["max_error_rate"]
            )
        if "max_latency_p99_ms" in criteria:
            checks["latency"] = (
                current_metrics.get("latency_p99_ms", 99999)
                <= criteria["max_latency_p99_ms"]
            )
        if "min_queries" in criteria:
            checks["sample_size"] = (
                current_metrics.get("query_count", 0)
                >= criteria["min_queries"]
            )

        all_passed = all(checks.values())

        if all_passed:
            return self._advance_stage()
        else:
            failed = [k for k, v in checks.items() if not v]
            return {
                "action": "hold",
                "reason": f"Criteria not met: {failed}",
                "checks": checks,
            }

    def _advance_stage(self) -> dict:
        """Advance to the next rollout stage."""
        if self.current_stage_idx >= len(ROLLOUT_STAGES) - 1:
            return {
                "action": "complete",
                "reason": "Rollout complete (100%)",
            }

        self.current_stage_idx += 1
        new_stage = self.current_stage
        self.stage_start_time = time.time()

        # Update feature flag percentage
        self.flags.set_rollout(
            self.feature, new_stage.percentage
        )

        logger.info(
            "Advanced to stage %s (%.0f%%)",
            new_stage.stage.value,
            new_stage.percentage,
        )

        return {
            "action": "advanced",
            "new_stage": new_stage.stage.value,
            "new_percentage": new_stage.percentage,
        }

    def rollback(self) -> dict:
        """Rollback to 0% (disable new pipeline entirely)."""
        self.flags.set_rollout(self.feature, 0.0)
        self.current_stage_idx = 0
        self.stage_start_time = time.time()

        return {
            "action": "rolled_back",
            "percentage": 0.0,
        }
```

---

## Multi-Armed Bandit for Traffic Allocation

### Thompson Sampling

```python
import numpy as np
from dataclasses import dataclass


@dataclass
class BanditArm:
    """One arm of the bandit (one pipeline version)."""

    name: str
    successes: int = 1  # Prior: 1 success (Beta prior)
    failures: int = 1   # Prior: 1 failure (Beta prior)

    def sample(self) -> float:
        """Sample from Beta distribution."""
        return np.random.beta(self.successes, self.failures)

    def update(self, reward: bool) -> None:
        if reward:
            self.successes += 1
        else:
            self.failures += 1

    @property
    def mean_reward(self) -> float:
        return self.successes / (self.successes + self.failures)


class BanditRAGRouter:
    """Route RAG queries using Thompson Sampling.

    Each pipeline version is a "bandit arm". The algorithm
    automatically allocates more traffic to the better-performing
    pipeline based on observed quality metrics.

    Advantages over fixed percentage rollout:
    - Automatically converges on the better pipeline
    - Minimizes regret (bad answers served during evaluation)
    - No manual stage advancement needed
    """

    def __init__(
        self,
        pipelines: dict[str, object],
        quality_fn=None,
    ):
        self.pipelines = pipelines
        self.arms = {
            name: BanditArm(name=name)
            for name in pipelines
        }
        self.quality_fn = quality_fn or self._default_quality

    def query(self, user_query: str) -> dict:
        """Route query to a pipeline selected by Thompson Sampling."""
        # Sample from each arm's distribution
        samples = {
            name: arm.sample()
            for name, arm in self.arms.items()
        }

        # Select the arm with highest sample
        selected = max(samples, key=samples.get)
        pipeline = self.pipelines[selected]

        # Execute query
        result = pipeline.query(user_query)
        result["_pipeline"] = selected

        # Compute quality reward (async in production)
        reward = self.quality_fn(result)
        self.arms[selected].update(reward)

        return result

    def _default_quality(self, result: dict) -> bool:
        """Default quality function: true if answer is non-empty
        and was generated without error."""
        return bool(result.get("answer", "").strip())

    def get_allocation(self) -> dict:
        """Get current traffic allocation probabilities."""
        # Simulate many samples to estimate allocation
        n_samples = 10000
        counts = {name: 0 for name in self.arms}

        for _ in range(n_samples):
            samples = {
                name: arm.sample()
                for name, arm in self.arms.items()
            }
            winner = max(samples, key=samples.get)
            counts[winner] += 1

        return {
            name: {
                "allocation_pct": count / n_samples * 100,
                "mean_reward": self.arms[name].mean_reward,
                "successes": self.arms[name].successes,
                "failures": self.arms[name].failures,
            }
            for name, count in counts.items()
        }
```

---

## Auto-Rollback

### Automated Rollback on Metric Degradation

```python
class AutoRollbackMonitor:
    """Monitor RAG metrics and automatically rollback on degradation.

    Runs as a continuous background process that checks metrics
    at regular intervals. Triggers rollback when metrics breach
    thresholds for a sustained period.
    """

    def __init__(
        self,
        rollout_manager: StagedRolloutManager,
        metrics_collector,
        check_interval_seconds: int = 60,
    ):
        self.rollout = rollout_manager
        self.metrics = metrics_collector
        self.interval = check_interval_seconds
        self.breach_count = 0
        self.max_breaches = 3  # Rollback after 3 consecutive breaches

        self.thresholds = {
            "max_error_rate": 0.05,
            "min_faithfulness": 0.60,
            "max_latency_p99_ms": 10000,
        }

    def check(self) -> dict:
        """Run a single metric check."""
        current = self.metrics.get_current(pipeline="new")

        breaches = []
        if current.get("error_rate", 0) > self.thresholds["max_error_rate"]:
            breaches.append(
                f"error_rate: {current['error_rate']:.3f} "
                f"> {self.thresholds['max_error_rate']}"
            )

        if current.get("faithfulness", 1) < self.thresholds["min_faithfulness"]:
            breaches.append(
                f"faithfulness: {current['faithfulness']:.3f} "
                f"< {self.thresholds['min_faithfulness']}"
            )

        if current.get("latency_p99_ms", 0) > self.thresholds["max_latency_p99_ms"]:
            breaches.append(
                f"latency_p99: {current['latency_p99_ms']:.0f}ms "
                f"> {self.thresholds['max_latency_p99_ms']}ms"
            )

        if breaches:
            self.breach_count += 1
            logger.warning(
                "Metric breach %d/%d: %s",
                self.breach_count,
                self.max_breaches,
                breaches,
            )

            if self.breach_count >= self.max_breaches:
                logger.error(
                    "Auto-rollback triggered after %d breaches",
                    self.breach_count,
                )
                rollback_result = self.rollout.rollback()
                self.breach_count = 0
                return {
                    "action": "rollback",
                    "breaches": breaches,
                    **rollback_result,
                }

            return {"action": "warning", "breaches": breaches}
        else:
            self.breach_count = 0
            return {"action": "ok"}
```

---

## Common Pitfalls

1. **Not using deterministic user assignment.** Random traffic splitting means the same user may flip between pipelines across requests, getting inconsistent experiences. Use hash-based assignment so each user consistently sees one pipeline.
2. **Advancing rollout stages based on time alone.** "It has been 24 hours, let us go to 50%" ignores metric quality. Always gate stage advancement on both time AND metric criteria.
3. **Thompson Sampling with binary reward only.** Using only "answer exists / does not exist" as reward is too coarse. Use graded rewards based on faithfulness scores or user feedback for faster convergence.
4. **Auto-rollback on transient spikes.** A single metric spike should not trigger rollback. Require consecutive breaches (3+) to filter out noise and prevent unnecessary rollbacks.
5. **Not notifying the team on rollback.** Auto-rollback prevents user impact, but the team must investigate why it happened. Send alerts on every rollback with the triggering metrics.
6. **Keeping the old pipeline running indefinitely "just in case."** After the new pipeline has been at 100% for a week with good metrics, decommission the old pipeline. Running both wastes resources.

---

## References

- LaunchDarkly feature flags: https://docs.launchdarkly.com/
- Thompson Sampling: Chapelle & Li "An Empirical Evaluation of Thompson Sampling" (NeurIPS 2011)
- Netflix Kayenta automated canary analysis: https://netflixtechblog.com/automated-canary-analysis-at-netflix-with-kayenta-3260bc7acc69
- Google progressive rollouts: https://cloud.google.com/deploy/docs/deployment-strategies/canary
