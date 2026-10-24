# Dominated-column elimination

Dominated-column elimination fixes a variable at a bound when transferring its contribution to another variable preserves feasibility and does not increase the objective. It removes the fixed variable using the shared fixed-value substitution machinery.

Unlike parallel-column aggregation, this rule does not require proportional constraint columns. The permitted transfer depends on each constraint's direction. This implementation considers variables with no quadratic objective terms, including cross terms.

## Example

Consider

\\[
\min_{x,y} 2x+y,
\qquad x+y\ge5,
\qquad x,y\ge0.
\\]

Increasing `y` while decreasing `x` by the same amount leaves the constraint unchanged and reduces cost. Since `y` has no upper bound, this transfer can continue until `x = 0`. Presolve fixes `x` there, leaving the objective `y` and the row `y >= 5`.

For example, `x = 2`, `y = 3` has cost `7`; replacing it by `x = 0`, `y = 5` has cost `5` and the same constraint contribution.

If `y <= 3` is added, this argument no longer permits fixing `x = 0`: satisfying the row requires `x >= 2`. The receiving variable needs enough room to accept the transfer.

## Mathematical formulation

Let \\(x_j\\) be the proposed dominant variable and \\(x_k\\) the proposed dominated variable. Consider the transfer

\\[
x_j^{\prime}=x_j+t,
\qquad x_k^{\prime}=x_k-t,
\qquad t\ge0.
\\]

Other variables are unchanged. With no quadratic terms involving either variable, the objective change is

\\[
f(x^{\prime})-f(x)=t(c_j-c_k).
\\]

Thus \\(c_j\le c_k\\) ensures that the transfer does not increase the objective. Equal costs are allowed for a bound fixing; strict improvement is required for an improving recession certificate.

### Constraint directions

For a linear row \\(i\\), its expression changes by \\(t d_i\\), where

\\[
d_i=A_{ij}-A_{ik}.
\\]

A missing coefficient is treated as zero. For the transfer to preserve every feasible row value, the implementation requires:

| Row bounds | Required coefficient difference | Reason |
| --- | --- | --- |
| Finite lower and upper bounds | `d_i = 0` | Preserve the row expression exactly. |
| Upper bound only | `d_i <= 0` | Do not increase the row expression. |
| Lower bound only | `d_i >= 0` | Do not decrease the row expression. |
| Neither side finite | Any finite `d_i` | No row restriction to violate. |

The first case includes equalities and ranged rows. Non-linear row domains encountered in the pair's support reject the relation.

For example, transferring from `x` to `y` preserves `x + y >= 5` and increases the left-hand side of `x + 2y >= 6`, so both lower-bound constraints remain satisfied. The same transfer could violate `x + 2y <= 6` and would fail that row's check.

## Open bound sides and fixing targets

The transfer increases the dominant variable and decreases the dominated variable. Their opposite bound sides cannot be violated by this movement, but the dominant upper side and dominated lower side can stop it.

`open_sides` treats a side as open when it is infinite or a retained linear row already implies that variable bound using the other variables' current bounds. An implied side remains satisfied because the transfer preserves the rows and the other necessary variable bounds; the code saves its proving row for recovery.

After proving dominance, the implementation selects a target in this order:

1. If the dominated variable has a finite lower bound and the dominant upper side is open, fix the dominated variable at its lower bound.
2. Otherwise, if the dominant variable has a finite upper bound and the dominated lower side is open, fix the dominant variable at its upper bound.
3. If both directional endpoints are infinite and costs differ, return an improving recession certificate.

If neither fixing is justified, the pair is left unchanged. When both directional endpoints are infinite but costs are equal, the code also leaves the pair unchanged rather than claiming unbounded improvement.

Thus the rule does not always remove the variable called dominated: it can fix the dominant variable at its upper bound instead.

### Optimality and recession direction

For an accepted fixing, every original feasible point can be transferred to a feasible point with that fixed value and no larger objective. Substituting the value therefore preserves the optimal value and permits recovery of an original optimum when one exists. It can discard other feasible points, so the reduction is not a one-to-one feasible-set correspondence.

When the dominant upper bound and dominated lower bound are infinite and \\(c_j<c_k\\), the direction is

\\[
d=e_j-e_k,\qquad c^Td=c_j-c_k<0.
\\]

The row-direction checks make this a feasible recession direction for the constraints, and neither variable has a bound blocking it. Since neither variable participates in the quadratic objective, \\(Pd=0\\). This establishes unbounded improvement from a feasible point; it does not prove that the rest of the problem is feasible.

## Implementation overview

The implementation is in `src/rules/dominated_columns.rs`:

- `dominates` compares linear costs and merges both sparse constraint columns to check every affected row, including entries present in only one column.
- `shift_target` checks whether the directional bound sides allow a fixing or recession direction.
- `dominated_pair` screens a pair, tests dominance in one orientation and, if that fails, the other, then performs the selected fixing or returns a certificate.

The pair screening uses a fingerprint of coefficients in rows with two finite sides. Such rows require exact coefficient equality, so different fingerprints can reject a pair cheaply. Matching fingerprints are not proof of dominance; the full row comparisons still follow.

The rule uses direct coefficient-difference and objective comparisons without a parallel-row tolerance. Nonfinite coefficient differences reject the relation. A successful fixing calls `Model::fix`, which checks the resulting row and objective transformations before applying the change. Failure to fix leaves the variable in place and does not prove infeasibility.

## Candidate search and work limits

The general shortest-row search is optional: `settings.dominated_columns.general_search` defaults to `false`. The scheduler runs it only when both this setting and `settings.rules.dominated_columns` are enabled. Otherwise, when the parallel-column scan runs with the dominated-column rule enabled, the default search uses its support groups.

The general `Model::dominated_columns` pass examines active variables without quadratic participation, skipping already fixed intervals. For each variable, it looks for partners in its shortest incident row, breaking shortest-row ties by row index. Rows longer than 256 entries are skipped. Column length and index ordering avoid mirror comparisons.

A second entry point, `dominated_support_groups`, reuses support-hash groups discovered by the parallel-column scan. It skips a group if any member has quadratic participation and compares each member with at most its next 32 group neighbors, checking active linear candidates before testing pairs.

Both searches share the same pair logic. Each pass has an estimated-work allowance from `settings.dominated_columns.work_limit`, defaulting to twice the current constraint nonzero count. Candidate visits and sparse comparison steps consume this allowance; the searches stop when it is exhausted. The rule is therefore a bounded search, not an exhaustive test of every variable pair.

The general pass returns the number of variables it fixes. The support-group pass returns its pair-comparison count.

## Implied bounds and numerical handling

Open-side proofs call `implying_row` with residual recomputation disabled. If a cancellation-sensitive cached activity cannot safely exclude the variable's own term, that proof is skipped rather than using a direct residual fallback. This can miss a valid reduction but avoids relying on that uncertain residual.

Per-pass scratch storage caches fingerprints, open-side results, and the rows proving implied bounds. Saved equation snapshots are shared for recovery. Only fixings change the model within these passes.

## Postsolve recovery

A successful elimination saves the ordinary `Record::Fixed`, restoring the chosen primal value and recovering its signed bound multiplier from stationarity during reverse recovery.

If the other variable's open side was finite and implied by a row, the rule additionally saves a `Record::ImpliedSide` after the fixing. Reverse recovery processes this record first, transferring any signed multiplier using that side to the proving row and adjusting the other bound multipliers. The earlier fixed-variable record then recovers the eliminated variable's multiplier.

This extra record accounts for the implied-side proof under the convention \\(Px+c=A^Ty+z\\). Infinite open sides need no such proof record. Recovery also maps a returned recession certificate through earlier reductions.
