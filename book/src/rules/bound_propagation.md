# Bound propagation

Bound propagation uses linear constraints and current variable bounds to derive tighter variable bounds. It can also detect infeasibility, remove redundant row sides, or expose a fixed variable for another reduction.

Unlike singleton-row elimination, this rule works on rows containing several variables. It does not change the objective.

## Example

Suppose a row is `x + y <= 10`, with `y >= 3` and `0 <= x <= 20`.

Even at its smallest allowed value, `y` contributes at least `3`, so `x` cannot exceed `7`. The updated bound is `0 <= x <= 7`.

Negative coefficients reverse the direction. For `-2x + y <= 10` and `y >= 4`, the row implies `-2x <= 6`, hence `x >= -3`.

The original row generally remains necessary: knowing `x <= 7` alone does not enforce `x + y <= 10` when `y` can exceed `3`.

## Mathematical formulation

Consider linear constraints and variable bounds

\\[
\ell\le Ax\le u,
\qquad L\le x\le U.
\\]

For each nonzero coefficient in row \\(i\\), define the smallest and largest possible contributions of its variable:

\\[
m_{ij}=\min(a_{ij}L_j,a_{ij}U_j),
\qquad M_{ij}=\max(a_{ij}L_j,a_{ij}U_j).
\\]

The row's minimum and maximum activities over the variable-bound box are

\\[
\alpha_i=\sum_j m_{ij},
\qquad \beta_i=\sum_j M_{ij}.
\\]

These are bounds on the row expression, not a guarantee that all values in the interval satisfy the other constraints. Sums here and below range over nonzero coefficients.

### Infeasibility and redundant sides

In exact arithmetic, \\(\alpha_i>u_i\\) or \\(\beta_i<\ell_i\\) proves infeasibility: even the most favorable variable values cannot satisfy that row side.

If \\(\alpha_i\ge\ell_i\\), the lower row side is already guaranteed by the variable bounds. If \\(\beta_i\le u_i\\), the upper row side is guaranteed. Either redundant side can be replaced by an infinite bound; if both sides are redundant, the row can be deleted.

### Isolate one variable

For a variable \\(x_k\\) with \\(a_{ik}\ne0\\), compute the activity of the remaining terms:

\\[
\alpha_{i,-k}=\sum_{j\ne k}m_{ij},
\qquad \beta_{i,-k}=\sum_{j\ne k}M_{ij}.
\\]

For \\(a_{ik}>0\\), the implied variable bounds are

\\[
\widehat L_k=\frac{\ell_i-\beta_{i,-k}}{a_{ik}},
\qquad
\widehat U_k=\frac{u_i-\alpha_{i,-k}}{a_{ik}}.
\\]

For \\(a_{ik}<0\\), division reverses the inequalities:

\\[
\widehat L_k=\frac{u_i-\alpha_{i,-k}}{a_{ik}},
\qquad
\widehat U_k=\frac{\ell_i-\beta_{i,-k}}{a_{ik}}.
\\]

Each implication requires a finite row side and a finite corresponding residual activity. An absent or unbounded side supplies no finite candidate through this formula.

Intersect each accepted candidate with the current variable interval:

\\[
L_k^{\prime}=\max(L_k,\widehat L_k),
\qquad U_k^{\prime}=\min(U_k,\widehat U_k).
\\]

A conflicting candidate can establish infeasibility. Equal finite updated bounds expose a variable for [fixed-variable elimination](./fixed_variables.md).

### Equivalence

In exact arithmetic, the derived bounds hold for every original feasible point, and removed row sides are already implied by retained variable bounds. Consequently, these updates preserve feasible variable vectors, objective values, and optimal solutions. Tightening a variable alone does not justify deleting its source row.

## Implementation overview

`Model::propagate_bounds` in `src/rules/bounds.rs` processes one round of rows from `changed_activities`. It skips non-linear row domains and rows with at most one coefficient; empty and singleton rows have separate rules.

For each row, the function obtains its activity range, checks for infeasibility, and removes redundant sides. It then visits the row's variables and calls `Activity::implied` in `src/model/activity.rs` to compute candidates from the other terms. The coefficient sign determines whether a candidate becomes a lower or upper variable bound.

`Model::implied_bound` checks each candidate and calls `tighten_bound` when appropriate. `Model::propagate_bounds` totals the successful updates and returns their count, which the scheduler uses when deciding whether further propagation rounds are useful.

Activity caches are updated as variable bounds change. Later variables in the same row can therefore benefit from earlier tightenings. A successful update also queues affected rows for reconsideration.

### Infinite contributions and cancellation

Activity storage tracks finite sums separately from counts of nonfinite contributions. This allows a finite residual to be recovered when the excluded variable supplied the only unbounded contribution.

The implementation first attempts to subtract the variable's contribution from the cached extreme. If cancellation makes that subtraction unreliable, it recomputes the residual directly from the other coefficients. If multiple unbounded contributions remain, no finite bound is derived from that extreme. A row-level shortcut skips candidate calculations when neither side can yield a finite residual.

## Numerical checks

Before returning a certificate for a row-activity contradiction, the function recomputes the activity directly and checks again. This avoids relying solely on a cached estimate affected by earlier arithmetic.

Contradictions use `Model::separated`, with the configured feasibility tolerance \\(\varepsilon\\):

\\[
L>U+\varepsilon\bigl(1+\max(|L|,|U|)\bigr).
\\]

A proposed bound that conflicts with its opposite bound beyond this tolerance returns an infeasibility certificate. A smaller conflict rejects the candidate instead of storing an inconsistent interval. Redundant-side removal uses direct containment comparisons without a tolerance.

Propagation calls `implied_bound` with `propagation = true`. It rejects nonfinite values and candidates whose absolute magnitude reaches `numerics.huge_bound`, then checks feasibility and whether the bound is tighter.

For an already finite bound, an update that does not exactly meet the opposite bound must also exceed a minimum improvement. The threshold is the larger of `minimum_gain_factor * numerics.feasibility` and `minimum_relative_gain * abs(old_bound)`, using the propagation settings. This avoids repeated insignificant changes. An update that fixes the variable bypasses this gain threshold but still undergoes the preceding checks.

## Scheduling and recovery

`Model::propagate_rounds` in `src/rules/mod.rs` runs an initial propagation round and permits additional rounds under `propagation.additional_rounds`, a shared work allowance, and a deadline. Between extra rounds it runs cleanup and enabled substitution rules, allowing tighter bounds to trigger structural reductions. Propagation therefore need not continue to a mathematical fixed point.

Each successful tightening saves a `Record::TightenedBound` with the old bound, affected variable side, and a snapshot of the row equation proving the update. Deleting a redundant row saves a `Record::DeletedRow`. Relaxing a redundant row side updates its bounds without adding a separate recovery record.

Bound propagation alone does not remove primal variables, so their values need no reconstruction. During reverse dual recovery, tightened-bound records can transfer signed bound multipliers to the saved source rows, adjusting the other bound multipliers to preserve stationarity. The saved equations remain available even if later reductions change or remove the source rows.
