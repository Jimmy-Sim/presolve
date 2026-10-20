# Empty-row elimination

An empty row is a linear constraint with no nonzero coefficients. Its left-hand side is always zero, so presolve either removes the row when zero satisfies its bounds or reports infeasibility when it does not.

This rule removes constraints, not variables. It does not change the objective or any variable bounds.

## Examples

The row `-2 <= 0 <= 7` is satisfied for every variable vector and can be removed.

The row `2 <= 0 <= 7` is impossible because its lower bound exceeds zero. Likewise, `-7 <= 0 <= -2` is impossible because its upper bound is below zero. Either row proves that the entire problem is infeasible.

For an empty equality, `0 = 0` is redundant, while `0 = 3` proves infeasibility in exact arithmetic.

## Mathematical formulation

Consider two-sided linear constraints

\\[
\ell \le Ax \le u.
\\]

If row \\(i\\) has no nonzero coefficients, then \\(A_{i:}=0\\) and its constraint reduces to

\\[
\ell_i \le 0 \le u_i.
\\]

In exact arithmetic, the row is redundant if \\(\ell_i\le0\\) and \\(u_i\ge0\\). Otherwise, no variable vector can satisfy it and the problem is infeasible. Infinite bounds representing an absent lower or upper restriction do not prevent removal when the other side is satisfied.

### Reduced problem and equivalence

Let \\(R\\) contain all row indices except \\(i\\). Removing a satisfied empty row leaves

\\[
\ell_R \le A_{R:}x \le u_R.
\\]

All variables, variable bounds, and objective data remain unchanged. In exact arithmetic, the original and reduced problems have the same feasible variable vectors and objective values, and therefore the same optimal solutions.

An empty row can be present initially or arise during another reduction. For example, fixing `x = 3` in `2x <= 6` leaves `0 <= 0`, which this rule can remove. Fixing `x = 3` in `2x >= 7` instead leaves `0 >= 1`, exposing infeasibility.

## Implementation overview

`Model::empty_rows` in `src/rules/rows.rs` drains the empty-row queue. For each candidate it checks that the row has a linear domain and that its sparse coefficient list is still empty. Other row domains and stale candidates that are no longer empty are skipped.

The rule checks both sides against zero:

```rust
if self.separated(b.lower, 0.0) {
    return Err(self.row_certificate(i, 1.0));
}
if self.separated(0.0, b.upper) {
    return Err(self.row_certificate(i, -1.0));
}
self.delete_row(i);
```

The lower-side check returns an infeasibility certificate with row multiplier `+1`; the upper-side check uses `-1`. Because the row has no coefficients, this row alone supplies the contradiction without requiring variable-bound multipliers at this stage of presolve. Recovery maps the certificate back through earlier reductions.

If neither check detects a violation beyond tolerance, `delete_row` retires the row and saves a `Record::DeletedRow` for postsolve.

`Queues::row_changed` in `src/model/queues.rs` queues rows whose coefficient count reaches zero. The cleanup loop in `src/rules/mod.rs` runs empty-row elimination when enabled and repeats while the model changes, allowing reductions such as fixed-variable elimination to expose new empty rows.

## Numerical tolerance

The implementation does not use an exact comparison to decide infeasibility. `Model::separated` in `src/rules/bounds.rs` tests

\\[
L > U + \varepsilon\bigl(1+\max(|L|,|U|)\bigr),
\\]

where \\(\varepsilon\\) is the configured feasibility tolerance. Empty-row elimination applies this test to \\((\ell_i,0)\\) and \\((0,u_i)\\).

A small violation within tolerance is therefore treated as satisfied and the row is removed. For example, `0 <= -7e-18` can be removed when the configured tolerance covers that residual. Retaining such roundoff can allow the downstream solver to magnify it into a misleading infeasibility certificate.

This tolerance-based behavior differs from exact feasible-set equivalence: a removed row can retain a tiny violation in the original problem. Recovery still exposes that original residual rather than making the original constraint exact.

## Postsolve recovery

No primal variable needs reconstruction for empty-row elimination alone. During dual recovery, `Record::DeletedRow` restores the removed row's multiplier as zero.

This is valid for a satisfied empty row because it contributes no variable coefficients to stationarity, and a zero multiplier satisfies complementarity. For a row removed within numerical tolerance, the recovered point can still have the small original primal residual described above.
