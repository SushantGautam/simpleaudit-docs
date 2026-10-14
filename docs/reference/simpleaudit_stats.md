## simpleaudit.stats

Pure statistics over audit verdicts.

### Functions

#### `wilson_interval(k: int, n: int, z: float = DEFAULT_Z) -> Tuple[float, float]`

Wilson score interval for a binomial proportion.

The Wilson interval is preferred over the naive Wald interval for small
`n` or proportions near 0 or 1, where the Wald interval is badly
calibrated. This is the standard form: a centre pulled toward 0.5 by the
`z^2/(2n)` term, with a half-width that shrinks as `n` grows.

**Args**
    k: number of successes (passing trials).
    n: total number of trials.
    z: critical value (default 1.96 ≈ 95%).

**Returns**
    `(lower, upper)` clamped to `[0, 1]`. When `n == 0` there is no
    evidence, so the full range `(0.0, 1.0)` is returned rather than a
    degenerate point.

#### `two_proportion_z(k1: int, n1: int, k2: int, n2: int) -> Optional[float]`

Pooled two-proportion z statistic for `p2 - p1`.

Tests whether the second proportion differs from the first, pooling the
two samples under the null hypothesis that they share one proportion. A
large magnitude (|z| >= 1.96 for 95%) indicates a real difference rather
than sampling noise.

**Args**
    k1, n1: successes and trials for the first (baseline) group.
    k2, n2: successes and trials for the second group.

**Returns**
    The z statistic, or `None` when it is undefined — either group has
    no trials, or the pooled standard error is zero (both groups agree
    perfectly, so there is no variance to test against).

### Constants

- `DEFAULT_Z`

